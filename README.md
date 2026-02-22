# Visual911

A FaceTime-style emergency call system that gives dispatchers live video, real-time contactless biometrics, and AI-generated triage — so callers who cannot speak can still communicate.

---

## The Problem

Millions of 911 calls come from people who can't speak: domestic violence victims hiding from an intruder, cardiac patients mid-event, people with throat injuries. Traditional 911 is audio-only — dispatchers hear silence and have no way to assess the situation.

## What Visual911 Does

A caller taps **SOS** on their iPhone. The app:

1. Runs a **15-second contactless vitals scan** — heart rate and breathing rate measured from the front camera alone (Presage SmartSpectra SDK, no wearable needed)
2. Opens a **live WebRTC video call** directly to the dispatcher's browser
3. Streams call audio to **Gemini AI** for real-time triage analysis every 10 seconds
4. Sends **GPS coordinates** and vitals to the dispatcher dashboard

The dispatcher sees: live video, biometric readings, an AI-generated situation summary with severity score, and a location pin — all in one screen.

---

## Architecture

```
┌──────────────┐        WebRTC P2P         ┌──────────────────┐
│   iPhone     │◄────── video/audio ──────►│  Dispatcher      │
│   (caller)   │                           │  Browser         │
└──────┬───────┘                           └────────┬─────────┘
       │ WebSocket                                  │ WebSocket
       │ (signal, audio, vitals)                    │ (dashboard)
       ▼                                            ▼
┌──────────────────────────────────────────────────────────────┐
│                    Vultr VPS (server.py)                     │
│  WebSocket hub · Gemini batch analysis · TURN credentials    │
└──────────────────────────────────────────────────────────────┘
```

- **iOS → Server**: 3 WebSockets (signaling, audio PCM + JPEG frames, vitals JSON)
- **Server → Gemini**: Batch `generateContent` every 10s with buffered WAV audio + latest video frame
- **Server → Dashboard**: Triage updates, vitals relay, call lifecycle events
- **iOS ↔ Dashboard**: Direct WebRTC peer-to-peer video/audio (TURN-assisted if needed)

---

## Stack

| Layer | Technology |
|---|---|
| iOS App | Swift/SwiftUI, Presage SmartSpectra SDK, stasel/WebRTC, AVAudioEngine, CoreLocation |
| Backend | Python 3.11+, asyncio, aiohttp, google-genai, python-dotenv |
| AI | Gemini 2.5 Flash (batch generateContent with cumulative context) |
| Infrastructure | Vultr Ubuntu VPS, coturn TURN server, Let's Encrypt SSL |
| Dispatcher UI | Plain HTML/CSS/JS, Leaflet.js, WebRTC browser API |
| Domain | visual911.mooo.com |

---

## Repo Structure

```
/
├── ios/                        # Swift iOS app (XcodeGen project)
│   ├── project.yml             # XcodeGen spec → generates .xcodeproj
│   ├── Config.swift            # Server URLs
│   ├── Secrets.swift           # API keys (gitignored)
│   ├── CallManager.swift       # Call lifecycle state machine
│   ├── PresageManager.swift    # Presage SDK wrapper
│   ├── WebRTCManager.swift     # WebRTC peer connection
│   ├── SignalingClient.swift   # /ws/signal WebSocket
│   ├── AudioTap.swift          # AVAudioEngine → /ws/audio
│   ├── VitalsClient.swift      # /ws/vitals WebSocket
│   ├── TURNCredentials.swift   # HMAC-SHA1 TURN credential generation
│   └── Views/                  # SwiftUI views per call state
├── server/
│   ├── server.py               # Single-file Python backend
│   ├── requirements.txt        # Pinned dependencies
│   └── static/index.html       # Dispatcher dashboard
└── docs/                       # Detailed implementation docs
```

---

## Getting Started

See [CONTRIBUTING.md](CONTRIBUTING.md) for full setup instructions (iOS, server, dashboard).

Local version:

```bash
# Virtual environment for server
cd server
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Server
echo 'GEMINI_API_KEY=your-key' > .env
python server.py

# iOS — open ios/ in Xcode, copy Secrets.swift.example → Secrets.swift, run on device
```

## License

[MIT](LICENSE)
