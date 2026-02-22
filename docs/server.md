# Server

## Overview

A single Python asyncio process (`server.py`) handles all backend logic: WebRTC signaling, Gemini batch analysis, vitals relay, TURN credential generation, and static file serving for the dispatcher dashboard. No separate Node.js process, no IPC.

Uses `aiohttp` for WebSocket serving, `python-dotenv` for environment config. One asyncio `Task` per active call manages periodic Gemini analysis using the batch `generateContent` API (not the Live API).

---

## Dependencies

```
aiohttp>=3.9.0
google-genai>=0.4.0
python-dotenv>=1.2.1
pytest>=8.0.0
pytest-aiohttp>=1.0.0
pytest-asyncio>=0.23.0
```

Python 3.11+ required. Install via `pip install -r requirements.txt`.

---

## File Structure

```
server/
├── server.py           # Everything — single file for hackathon simplicity
├── requirements.txt    # Pinned dependencies
├── pyproject.toml      # Project metadata
├── static/
│   └── index.html      # Dispatcher dashboard (served at /)
└── tests/
    ├── conftest.py
    └── test_server.py  # Pytest suite for WebSocket handlers
```

---

## server.py

### Imports and Setup

```python
import asyncio, base64, io, json, logging, os, struct, time, uuid, wave
from dotenv import load_dotenv
load_dotenv()

import aiohttp
from aiohttp import web
from google import genai
from google.genai import types
```

### State

```python
active_calls: dict[str, dict] = {}
# Structure per call_id:
# {
#   "caller_ws": WebSocketResponse,
#   "dispatcher_ws": Optional[WebSocketResponse],
#   "dashboard_ws": Optional[WebSocketResponse],
#   "audio_queue": asyncio.Queue,
#   "gemini_task": Optional[asyncio.Task],
#   "location": {"lat": float, "lng": float},
#   "started_at": float,
#   "last_vitals": Optional[dict],
# }

dispatcher_connections: set[web.WebSocketResponse] = set()

# Buffer vitals that arrive before call_initiated registers the call
pending_vitals: dict[str, dict] = {}
```

### Gemini Configuration

```python
GEMINI_MODEL = "gemini-2.5-flash"
ANALYSIS_INTERVAL = 10  # seconds between Gemini calls
AUDIO_SAMPLE_RATE = 16000
AUDIO_BYTES_PER_SAMPLE = 2  # 16-bit PCM

SYSTEM_PROMPT = """You are an AI emergency triage assistant analyzing a 911 call audio clip.
Output ONLY a JSON object — no preamble, no markdown, no extra text:
{
  "situation_summary": "one sentence description",
  "detected_keywords": ["list", "of", "keywords"],
  "caller_emotional_state": "calm|distressed|panicked|unresponsive",
  "recommended_response_type": "medical|police|fire|unknown",
  "severity": 1,
  "can_speak": true
}
Severity: 1=minor, 2=moderate, 3=urgent, 4=serious, 5=life-threatening.
Keep output under 100 tokens. Speed over verbosity."""
```

**Key change from original design:** Uses batch `generateContent` instead of the Gemini Live API. No `flag_critical` function tool — severity is communicated via the JSON output directly.

### pcm_to_wav Helper

Raw PCM bytes must be wrapped in a WAV container because `generateContent` does not accept `audio/pcm`:

```python
def pcm_to_wav(pcm_bytes: bytes, sample_rate=16000, channels=1, sample_width=2) -> bytes:
    buf = io.BytesIO()
    with wave.open(buf, "wb") as wf:
        wf.setnchannels(channels)
        wf.setsampwidth(sample_width)
        wf.setframerate(sample_rate)
        wf.writeframes(pcm_bytes)
    return buf.getvalue()
```

### Gemini Analysis Task

```python
async def gemini_session_task(call_id, audio_queue, dashboard_ws_getter):
    """
    Periodically drains the audio queue, sends buffered PCM (as WAV) +
    latest video frame to Gemini generateContent, and forwards triage JSON
    to the dashboard.
    """
    client = genai.Client(api_key=os.environ["GEMINI_API_KEY"])
    audio_buffer = bytearray()
    latest_frame: str | None = None  # base64 JPEG
    previous_summary: str = ""  # cumulative context across rounds
    analysis_count = 0

    while True:
        # Collect audio for ANALYSIS_INTERVAL seconds
        # ... drain queue, accumulate audio_buffer, keep latest_frame ...

        # Convert raw PCM to WAV
        wav_bytes = pcm_to_wav(chunk)

        # Build content parts
        parts = [
            types.Part(inline_data=types.Blob(data=wav_bytes, mime_type="audio/wav")),
        ]
        if latest_frame:
            parts.append(types.Part(inline_data=types.Blob(
                data=base64.b64decode(latest_frame), mime_type="image/jpeg")))

        # Prompt with cumulative context
        prompt = "Analyze this 911 call audio clip and output triage JSON."
        if previous_summary:
            prompt = f"Previous analysis: {previous_summary}\n\nNew audio segment (update #{analysis_count}). Update your triage. Output triage JSON."

        parts.append(types.Part(text=prompt))

        response = await client.aio.models.generate_content(
            model=GEMINI_MODEL, contents=parts,
            config=types.GenerateContentConfig(
                system_instruction=SYSTEM_PROMPT, temperature=0.1))

        # Parse JSON from response, forward to dashboard
        # Save situation_summary for next round's context
```

Error handling: on analysis failure, sends a triage_update with error info to the dashboard so it doesn't stay on "Waiting for analysis..." forever.

### WebSocket Handlers

#### `/ws/signal` — Signaling

```python
async def handle_signal(request):
    # Query params: call_id, role (caller|dispatcher)
```

Handles call lifecycle and WebRTC SDP/ICE forwarding:
- `call_initiated`: Creates the call entry in `active_calls` with audio_queue, notifies all connected dispatchers via `incoming_call`. Checks `pending_vitals` buffer and replays any early vitals.
- `call_ended`: Triggers `cleanup_call()`.
- Other messages (SDP offers/answers, ICE candidates): forwarded to the opposing party (caller↔dispatcher).
- On WebSocket close: if caller disconnects, triggers cleanup.

Guard against duplicate `call_initiated` for the same `call_id`.

#### `/ws/audio` — Audio + Frame Ingestion

```python
async def handle_audio(request):
    # Query params: call_id
```

- **Binary messages**: raw PCM chunks → `audio_queue.put({"type": "audio", "data": msg.data})`
- **Text messages**: JSON `{ "type": "frame", "data": "<base64 JPEG>" }` → `audio_queue.put({"type": "frame", "data": ...})`

If queue is full (maxsize=500), drops silently. Audio and frames are consumed by the Gemini analysis task.

#### `/ws/vitals` — Vitals Relay

```python
async def handle_vitals(request):
    # Query params: call_id
```

Receives vitals JSON from iOS. Two behaviors:
1. **Call exists**: stores as `call["last_vitals"]`, forwards to dashboard WebSocket.
2. **Call not yet registered** (vitals arrive before `call_initiated`): stores in `pending_vitals[call_id]` buffer. These are replayed when `call_initiated` fires.

When dispatcher joins (`dispatcher_joined` message on dashboard WS), `last_vitals` is replayed so the dashboard immediately shows the most recent reading.

#### `/ws/dashboard` — Dispatcher Dashboard

```python
async def handle_dashboard(request):
```

Bidirectional WebSocket for the dispatcher browser:
- `dispatcher_joined`: Registers the dashboard WS on the call, starts the Gemini analysis task, replays `last_vitals`, sends `dispatcher_ready` to caller.
- `call_ended`: Triggers `cleanup_call()`.
- Other messages (SDP/ICE): forwarded to caller.
- On disconnect: removes from `dispatcher_connections` set.

### REST Endpoints

#### `GET /` — Dashboard Page

```python
async def handle_index(request):
    return web.FileResponse("./static/index.html")
```

Serves the dashboard HTML directly at the root path.

#### `GET /api/turn-credentials` — Dynamic TURN Credentials

```python
async def handle_turn_credentials(request):
```

Generates time-limited TURN credentials using HMAC-SHA1:
- Username = `{expiry_timestamp}:visual911` (24-hour TTL)
- Password = base64-encoded HMAC-SHA1 of the username with `TURN_SECRET`
- Returns JSON with `username`, `credential`, `ttl`, and `uris` array

Requires `TURN_SECRET` environment variable matching the coturn `static-auth-secret`.

### Cleanup

```python
async def cleanup_call(call_id: str, reason: str = "ended"):
```

1. Pops call from `active_calls`
2. Cancels Gemini task (sends sentinel to queue, then `task.cancel()`)
3. Notifies dashboard with `call_ended`
4. Notifies caller with `call_ended`
5. Cleans up `pending_vitals` for the call_id

### App Setup

```python
def create_app() -> web.Application:
    app = web.Application()
    app.router.add_get("/", handle_index)
    app.router.add_get("/api/turn-credentials", handle_turn_credentials)
    app.router.add_get("/ws/signal", handle_signal)
    app.router.add_get("/ws/audio", handle_audio)
    app.router.add_get("/ws/vitals", handle_vitals)
    app.router.add_get("/ws/dashboard", handle_dashboard)
    app.router.add_static("/static", path="./static", name="static")
    return app
```

Routes summary:
| Route | Type | Purpose |
|---|---|---|
| `/` | GET | Dashboard HTML |
| `/api/turn-credentials` | GET | Dynamic TURN creds |
| `/static/` | Static | CSS, JS, assets |
| `/ws/signal` | WS | Signaling + call lifecycle |
| `/ws/audio` | WS | Audio PCM + JPEG frames |
| `/ws/vitals` | WS | Vitals relay |
| `/ws/dashboard` | WS | Dispatcher communication |

```python
if __name__ == "__main__":
    import ssl as _ssl
    port = int(os.environ.get("PORT", 8080))
    ssl_cert = os.environ.get("SSL_CERT")
    ssl_key = os.environ.get("SSL_KEY")

    ssl_context = None
    if ssl_cert and ssl_key:
        ssl_context = _ssl.create_default_context(_ssl.Purpose.CLIENT_AUTH)
        ssl_context.load_cert_chain(ssl_cert, ssl_key)

    web.run_app(create_app(), port=port, ssl_context=ssl_context)
```

---

## Environment Variables

```bash
GEMINI_API_KEY="your-gemini-api-key"   # Required
TURN_SECRET="shared-secret-with-coturn" # Required for /api/turn-credentials
PORT=8080                               # Optional, defaults to 8080
SSL_CERT="/path/to/fullchain.pem"       # Optional, for TLS termination
SSL_KEY="/path/to/privkey.pem"          # Optional, for TLS termination
```

Uses `python-dotenv` — place a `.env` file in the server directory for local development. In production on Vultr, SSL termination is handled by the server process itself. See `deployment.md` for the full SSL + coturn setup.

---

## Gemini Analysis Lifecycle

Each call gets one Gemini analysis loop (not a persistent session). The task starts when the **dispatcher joins** (not at `call_initiated`) to save API quota.

```
iOS audio tap → /ws/audio handler → audio_queue.put()
                                           ↓
                                  gemini_session_task (every 10s)
                                  drains queue → PCM buffer
                                  wraps in WAV via pcm_to_wav()
                                  + latest JPEG frame
                                  + previous_summary context
                                           ↓
                                  client.aio.models.generate_content()
                                  model=gemini-2.5-flash, temp=0.1
                                           ↓
                                  parses JSON from response
                                  stores situation_summary for next round
                                           ↓
                                  /ws/dashboard → triage_update
```

### Cumulative Context

Each analysis round receives the `previous_summary` from the last successful analysis. This gives Gemini continuity without maintaining a Live API session:

```
Round 1: "Analyze this 911 call audio clip. Output triage JSON."
Round 2: "Previous analysis: {summary}. New audio segment (update #2). Update your triage."
Round 3: "Previous analysis: {summary}. New audio segment (update #3). Update your triage."
```

### API Limits (Free Tier)

- 10 requests per minute
- 250,000 tokens per minute
- 250 requests per day

With 10-second intervals, one call uses ~6 RPM. Two simultaneous calls would approach the 10 RPM limit. For the hackathon demo, one call at a time is safe.

### Error Handling

On Gemini API failure, the server sends an error triage_update to the dashboard:
```json
{
  "type": "triage_update",
  "call_id": "...",
  "report": {
    "situation_summary": "Gemini analysis error: ...",
    "severity": 0
  }
}
```

This prevents the dashboard from showing "Waiting for analysis..." indefinitely.

---

## Running Locally

```bash
cd server/
cp .env.example .env  # Add your GEMINI_API_KEY and TURN_SECRET
python3 server.py
```

The server listens on port 8080 by default. For local testing, iOS must connect via your Mac's IP address. For testing with ngrok, set the server URL in the iOS `Config.swift` accordingly.

---

## Testing

```bash
cd server/
pip install -r requirements.txt  # Includes pytest, pytest-aiohttp, pytest-asyncio
pytest tests/
```

Tests in `tests/test_server.py` verify WebSocket handler behavior using `aiohttp.test_utils`. See `tests/conftest.py` for shared fixtures.

---

## Message Schema Reference

See `CLAUDE.md` for the full message schema. Quick reference:

**iOS → Server (via /ws/signal):** `call_initiated`, `call_ended`, WebRTC SDP/ICE

**iOS → Server (via /ws/audio):** binary PCM, `{ type: "frame", data: "<base64 JPEG>" }`

**iOS → Server (via /ws/vitals):** `{ type: "vitals", call_id, hr, hrConfidence, breathing, breathingConfidence, timestamp }`

**Server → iOS:** `dispatcher_ready`, `call_ended`, WebRTC SDP/ICE

**Server → Dashboard (/ws/dashboard):** `incoming_call`, `triage_update`, `vitals`, `call_ended`

**Dashboard → Server (/ws/dashboard):** `dispatcher_joined`, `call_ended`, WebRTC SDP/ICE
