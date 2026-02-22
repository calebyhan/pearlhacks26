# Contributing — Visual911

End-to-end onboarding guide. Follow these steps to get the full stack running locally and deploying to production.

---

## Prerequisites

| Tool | Version | Check |
|---|---|---|
| Xcode | 16.0+ | `xcode-select -p` |
| Physical iPhone | iOS 17.0+ | Presage + WebRTC require a real device (no simulator) |
| Python | 3.11+ | `python3 --version` |
| XcodeGen | Latest | `brew install xcodegen` |
| Gemini API key | Free tier | [aistudio.google.com](https://aistudio.google.com) |
| Presage API key | From team | physiology.presagetech.com |

---

## 1. Clone & Orient

```bash
git clone https://github.com/calebyhan/pearlhacks26.git
cd pearlhacks26
```

Key directories:
- `ios/` — Swift iOS app
- `server/` — Python backend + dashboard HTML
- `deploy/` — coturn config template
- `docs/` — detailed implementation guides

---

## 2. Server Setup

### Install dependencies

```bash
cd server
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### Configure environment

```bash
cp .env.example .env   # if .env.example exists, otherwise:
cat > .env << 'EOF'
GEMINI_API_KEY=your-gemini-api-key
TURN_SECRET=your-coturn-shared-secret
```

| Variable | Required | Description |
|---|---|---|
| `GEMINI_API_KEY` | Yes | From Google AI Studio |
| `TURN_SECRET` | Production only | Must match coturn `static-auth-secret` |
| `PORT` | No | Defaults to 8080 |
| `SSL_CERT` | Production only | Path to fullchain.pem |
| `SSL_KEY` | Production only | Path to privkey.pem |

### Run

```bash
python server.py
```

Server starts on `http://localhost:8080`. The dashboard is at `http://localhost:8080/`.

### Run tests

```bash
pytest tests/
```

---

## 3. iOS Setup

### Generate Xcode project

The repo uses [XcodeGen](https://github.com/yonaskolb/XcodeGen) — there's no committed `.xcodeproj` (it's gitignored). Generate it from `project.yml`:

```bash
cd ios
xcodegen generate
```

This creates `Visual911.xcodeproj` and resolves SPM packages (SmartSpectra, WebRTC).

### Create Secrets.swift

```bash
cp Secrets.swift.example Secrets.swift
```

Edit `Secrets.swift` and fill in real values:

```swift
enum Secrets {
    static let presageApiKey = "YOUR_PRESAGE_API_KEY"
    // Optional: add turnSecret here if not in TURNCredentials.swift
}
```

`Secrets.swift` is gitignored — never commit API keys.

### Configure server URL

Edit `Config.swift` if you need to point to a different server:

```swift
enum Config {
    static let presageApiKey = Secrets.presageApiKey
    static let serverHost    = "wss://visual911.mooo.com"  // production
    // For local dev: "ws://YOUR_MAC_IP:8080"
}
```

For local development, use your Mac's local IP (not `localhost` — the phone is a separate device). Find it with:
```bash
ipconfig getifaddr en0
```

### Build & Run

1. Open `ios/Visual911.xcodeproj` in Xcode
2. Select your physical iPhone as the target device
3. Set your development team in Signing & Capabilities
4. Build & Run (⌘R)

**Requirements for Presage vitals scanning:**
- Physical device only (no simulator)
- Face 1–2 feet from front camera
- ≥60 lux ambient lighting (overhead lights on)
- Subject reasonably still during 15-second scan

---

## 4. Dashboard (Dispatcher UI)

The dashboard is served by the Python server — no separate build step. Open your browser to wherever the server is running:

- Local: `http://localhost:8080`
- Production: `https://visual911.mooo.com`

The dashboard auto-connects via WebSocket and auto-reconnects on disconnect. A green dot in the header indicates a live connection.

---

## 5. Local Development Workflow

### Full stack locally

1. Start the server: `cd server && python3 server.py`
2. Open dashboard in browser: `http://localhost:8080`
3. Update `Config.swift` to point to `ws://YOUR_MAC_IP:8080`
4. Run the iOS app on your phone
5. Press SOS → vitals scan → incoming call appears on dashboard → click ANSWER

### What to expect

- **Presage scan**: 15 seconds, shows feedback states (measuring, hold still, etc.)
- **WebRTC video**: appears in dashboard after ANSWER, may take a few seconds for ICE negotiation
- **Gemini triage**: first update ~10 seconds after dispatcher joins (needs audio to analyze)
- **Vitals**: appear on dashboard immediately from the pre-call scan

### Common issues during local dev

| Issue | Cause | Fix |
|---|---|---|
| iOS can't connect to server | Wrong IP or port | Check `Config.swift`, verify with `curl http://YOUR_IP:8080/` from phone browser |
| WebRTC video doesn't appear | TURN not available locally | Both devices on same WiFi usually works without TURN |
| Gemini triage stays "Waiting..." | No `GEMINI_API_KEY` set | Check `.env` file |
| Presage shows no readings | Too dark / too far / simulator | Use physical device, good lighting, face 1-2ft away |
| Xcode can't find packages | SPM resolution failed | File → Packages → Reset Package Caches |

---

## 6. Production Deployment

Full instructions in [docs/deployment.md](docs/deployment.md). Summary:

1. Provision Vultr Ubuntu 24.04 VPS
2. Point `visual911.mooo.com` A record to VPS IP
3. Install dependencies: `apt install python3 python3-pip coturn certbot`
4. Get SSL cert: `certbot certonly --standalone -d visual911.mooo.com`
5. Configure coturn with `static-auth-secret` matching your `TURN_SECRET`
6. Deploy server code to `/opt/visual911/`
7. Set environment variables in `/opt/visual911/.env`
8. Create systemd service for auto-restart
9. Update `Config.swift` to `wss://visual911.mooo.com` and rebuild iOS app

### Verify deployment

```bash
# Dashboard loads
curl https://visual911.mooo.com/

# WebSocket connects
wscat -c wss://visual911.mooo.com/ws/dashboard

# TURN credentials work
curl https://visual911.mooo.com/api/turn-credentials

# coturn running
systemctl status coturn

# Server logs
journalctl -u visual911 -f
```

---

## 7. Project Conventions

### Code style
- **Swift**: Standard Swift conventions, SwiftUI for views
- **Python**: Standard library style, single `server.py` file (hackathon simplicity)
- **Dashboard**: Plain HTML/CSS/JS, no framework, single `index.html`

### Secrets management
- iOS: `Secrets.swift` (gitignored) — copy from `Secrets.swift.example`
- Server: `.env` file (gitignored) — loaded by `python-dotenv`
- Never commit API keys, TURN secrets, or credentials

### Git
- Don't commit `Secrets.swift`, `.env`, or `*.xcodeproj` (generated by XcodeGen)
- The `.xcodeproj` is generated from `project.yml` — edit `project.yml` for project config changes, then run `xcodegen generate`

### Presage credit budget
300 total credits. 1 credit ≈ 30s of measurement. **Never run Presage in idle mode.** Only start scanning when the user presses SOS, stop when the scan completes or the call ends.

### Gemini rate limits (free tier)
- 10 requests per minute
- 250 requests per day
- One active call uses ~6 RPM (analysis every 10s)
- One call at a time is safe; two simultaneous calls approach the RPM limit

---

## 8. Key Files Reference

| File | What it does |
|---|---|
| `server/server.py` | Entire Python backend — WebSocket handlers, Gemini analysis, TURN credentials |
| `server/static/index.html` | Dispatcher dashboard UI |
| `ios/CallManager.swift` | Call lifecycle state machine (idle → scanning → initiating → connecting → active) |
| `ios/PresageManager.swift` | Presage SDK wrapper — manages scan lifecycle and vitals extraction |
| `ios/WebRTCManager.swift` | WebRTC peer connection, ICE handling, camera capture |
| `ios/SignalingClient.swift` | /ws/signal WebSocket client |
| `ios/AudioTap.swift` | AVAudioEngine tap → PCM audio to /ws/audio |
| `ios/VitalsClient.swift` | /ws/vitals WebSocket client — sends vitals JSON to server |
| `ios/TURNCredentials.swift` | Generates HMAC-SHA1 time-limited TURN credentials |
| `ios/Config.swift` | Server URLs and API key references |
| `ios/Secrets.swift` | API keys (gitignored) |
| `ios/project.yml` | XcodeGen project definition |
| `docs/` | Detailed implementation guides for each component |
