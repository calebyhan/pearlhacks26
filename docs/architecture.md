# Architecture

## System Overview

Visual911 has three main components: an iOS caller app, a Python backend on Vultr, and a browser-based dispatcher dashboard. The iOS app never communicates directly with the dispatcher dashboard except for the P2P WebRTC video/audio stream — all other data (vitals, AI triage, call events) routes through the Vultr server.

```
┌─────────────────────────────────────────────────────────────────┐
│                         iOS Caller App                           │
│                                                                   │
│  Presage SDK ──► vitals JSON ──────────────────────────────────► │
│  AVAudioEngine ──► PCM audio + JPEG frames ────────────────────► │
│  WebRTC ◄──────────────────────────────── signaling ───────────► │ Vultr Server
│  CLLocationManager ──► GPS coords ─────────────────────────────► │
│                                                                   │
│  WebRTC video/audio ◄──────────────────────────P2P──────────────►│ Dispatcher
└─────────────────────────────────────────────────────────────────┘    Browser
                              │
                    ┌─────────▼─────────┐
                    │   Vultr Server     │
                    │   (server.py)      │
                    │                    │
                    │  /ws/signal        │──► forwards SDP/ICE
                    │  /ws/audio    ─────┼──► Gemini (batch)
                    │  /ws/vitals        │
                    │  /ws/dashboard ────┼──► Dispatcher Browser
                    │  /api/turn-creds   │──► TURN credentials
                    └────────────────────┘
```

---

## Components

### iOS App (Swift)

The caller-facing application. Manages the full call lifecycle and coordinates three hardware resources. Built with XcodeGen (`project.yml`).

**Presage SmartSpectra SDK** reads contactless vitals (heart rate, breathing rate) from the front camera using rPPG. Runs for up to 15 seconds when SOS is pressed, but can finish early once signals stabilize (minimum 5s, requires 2s error cooldown). Real-time SDK status codes (face not found, too dark, etc.) drive scan feedback UI. Results are shown as "last measured" on the dispatcher dashboard.

**RTCCameraVideoCapturer** (stasel/WebRTC) runs after Presage completes its scan. Captures front camera video at ~720p and streams it P2P to the dispatcher's browser via WebRTC. Audio is included in the WebRTC stream. A `FrameGrabber` (custom `RTCVideoRenderer`) retains the latest `CVPixelBuffer` for JPEG snapshot extraction.

**AVAudioEngine tap** runs in parallel with WebRTC audio using `mixWithOthers` session mode. Captures a separate 16kHz mono PCM stream and sends it over a secondary WebSocket to the server, which buffers it for Gemini. Also sends JPEG frames every 2 seconds (captured via `FrameGrabber` from the WebRTC video track).

**CLLocationManager** provides a one-shot GPS coordinate sent with the `call_initiated` message.

### Vultr Server (Python asyncio)

A single `server.py` process handles all backend logic. Uses `aiohttp` for WebSocket serving, `python-dotenv` for environment config, and asyncio tasks for concurrent call management.

Four WebSocket endpoints:
- `/ws/signal` — forwards WebRTC SDP offers, answers, and ICE candidates between caller and dispatcher
- `/ws/audio` — receives PCM audio and JPEG frames from iOS, buffers them for Gemini batch analysis
- `/ws/vitals` — receives vitals JSON from iOS, forwards to the connected dispatcher dashboard; buffers vitals that arrive before `call_initiated` registers the call
- `/ws/dashboard` — browser-facing endpoint; pushes incoming call notifications, triage updates, vitals, and call events; replays cached vitals on dispatcher join

One REST endpoint:
- `/api/turn-credentials` — generates time-limited HMAC TURN credentials for WebRTC NAT traversal

One asyncio task per active call runs **batch Gemini analysis**: every 10 seconds, it drains the audio queue, wraps accumulated PCM in a WAV container, and calls `gemini-2.5-flash` via `generateContent` (not the Live API). Previous analysis summary is carried forward as context for continuity.

A separate `coturn` process (port 3478/5349) provides STUN/TURN for WebRTC NAT traversal.

### Dispatcher Dashboard (Browser)

A single `index.html` served at `/` on Vultr. Uses the browser WebRTC API for the P2P video connection and a WebSocket to `/ws/dashboard` for all other data. No framework — plain HTML, CSS, and JavaScript.

Three-column, two-row grid layout:
- **Video** (left, full height) — `<video>` element fed by the WebRTC P2P stream, mirrored, with overlay mute/end controls
- **Vitals** (top center) — HR and breathing rate with confidence bars; labeled "From pre-call scan"
- **AI Triage** (top right) — situation summary, severity level (1–5), emotional state, recommended response type, keywords
- **Location** (bottom, spans center + right) — Leaflet.js map with GPS pin

TURN credentials are fetched dynamically from `/api/turn-credentials` on page load. Connection status indicator in the header shows WebSocket state.

---

## Data Flow

### Call Initiation

```
1. User presses SOS
2. Presage starts scanning (up to 15s, can finish early when signals stabilize)
3. Real-time SDK feedback shown (face position, lighting quality, signal stability)
4. After scan: Presage stops, results cached locally

5. iOS opens:
   - /ws/signal?call_id=<uuid>&role=caller
   - /ws/audio?call_id=<uuid>
   - /ws/vitals?call_id=<uuid>

6. iOS sends: { type: "call_initiated", call_id, location }
   (vitals sent 0.3s later to ensure call is registered server-side first)

7. Server:
   - Stores call state (including audio_queue)
   - Replays any vitals buffered before call registration
   - Notifies all connected dispatcher dashboards: { type: "incoming_call", call_id, location }

8. Dispatcher clicks Answer → sends { type: "dispatcher_joined", call_id }

9. Server:
   - Records dispatcher WebSocket
   - Starts Gemini analysis task (not at call_initiated — saves RPD quota)
   - Replays cached vitals to dispatcher
   - Sends { type: "dispatcher_ready" } to iOS

10. WebRTC handshake begins (iOS creates offer, signals through /ws/signal)

11. P2P video/audio connection established
```

### Active Call

```
Continuously (audio):
iOS AVAudioEngine → PCM chunks → /ws/audio → server audio_queue

Every 2s (frames):
iOS → JPEG snapshot (from FrameGrabber) → /ws/audio (JSON) → server audio_queue

Every 10s (Gemini batch analysis):
Server drains audio_queue → wraps PCM in WAV → generateContent(gemini-2.5-flash)
  + latest JPEG frame + previous analysis context
Gemini → triage JSON → server parses → /ws/dashboard { type: "triage_update" }

Video/audio (P2P, server not involved):
iOS RTCCameraVideoCapturer → WebRTC → dispatcher <video> element
```

### Call End

```
Either party sends { type: "call_ended", call_id }
Server:
  - Sends sentinel to audio_queue, cancels Gemini task
  - Notifies other party
  - Removes call from active_calls dict
iOS:
  - Closes all 3 WebSockets
  - Stops RTCCameraVideoCapturer
  - Thread-safe cleanup with re-entrancy guard
  - Returns to IDLE state
Dashboard:
  - Tears down WebRTC peer connection
  - Resets vitals, triage, map marker, and UI controls
```

---

## State Machine (iOS)

```
IDLE
  │ SOS pressed
  ▼
SCANNING  ← Presage running, camera exclusive, up to 15s
  │         real-time SDK feedback (face position, lighting)
  │         can finish early when HR + BR signals stabilize
  │         user can retry scan
  │ scan complete (stable signals or timeout)
  ▼
INITIATING  ← open 3 WebSockets, send call_initiated + vitals
  │ dispatcher answers (dispatcher_ready received)
  ▼
CONNECTING  ← WebRTC offer/answer/ICE exchange via /ws/signal
  │ P2P established (ICE connected/completed)
  ▼
ACTIVE  ← video, audio, vitals, Gemini all live
  │         resends cached vitals on ICE connect
  │         ICE disconnected = transient (no teardown)
  │ end call (either party) or ICE failed
  ▼
CLEANUP  ← thread-safe, re-entrant-safe teardown
  │         close WebSockets, stop capturer, nil references
  ▼
IDLE
```

---

## Ports and Protocols

| Service | Port | Protocol |
|---|---|---|
| Vultr server (HTTP/WS) | 443 | WSS/HTTPS (TLS via Let's Encrypt) |
| coturn STUN | 3478 | UDP/TCP |
| coturn TURN (TLS) | 5349 | TLS |
| WebRTC P2P | ephemeral | SRTP/DTLS (UDP) |

---

## Key Design Decisions

**Why a single Python process instead of Node.js + Python?**
Fewer moving parts for a hackathon. aiohttp handles WebSockets fine. No IPC needed between services.

**Why batch `generateContent` instead of Gemini Live API?**
The Live API (audio-only mode) has a 15-minute session limit, and the native audio preview model had reliability issues. Batch analysis every 10 seconds using `gemini-2.5-flash` with WAV audio + JPEG frames is more reliable, simpler to debug, and carries forward previous analysis context for continuity. Trade-off: ~10s latency on triage updates instead of real-time streaming.

**Why time-slice Presage and WebRTC instead of running them concurrently?**
Both need exclusive access to `AVCaptureSession` on the front camera. Two sessions on the same camera cause a hard crash (-12785). The pre-call scan gives dispatchers biometric context the moment the call connects, which is actually a better UX story than trying to show live vitals mid-call.

**Why route vitals through the server instead of RTCDataChannel?**
DataChannel is P2P — the server never sees it. Routing through `/ws/vitals` lets the server cache vitals and replay them when the dispatcher joins, and keeps the dashboard on a single data pipeline (`/ws/dashboard`) instead of two separate channels.

**Why dynamic TURN credentials via `/api/turn-credentials`?**
coturn uses time-limited HMAC credentials. The server generates them on demand so both the iOS app and dashboard can fetch fresh credentials without hardcoding secrets in client code.
