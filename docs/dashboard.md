# Dashboard

## Overview

The dispatcher dashboard is a single `index.html` file served from the Vultr server at `/`. It runs entirely in the browser with no framework — plain HTML, CSS, and JavaScript.

It connects to two things:
1. The Vultr server via `/ws/dashboard` WebSocket (receives triage updates, vitals, call events)
2. The iOS caller app via WebRTC P2P (receives live video and audio directly)

External dependencies: Leaflet.js for maps. No Chart.js — vitals are displayed as numeric values with confidence bars.

---

## Layout

```
┌──────────────────────────────────────────────────────────────────┐
│  ● Connected        VISUAL911                                    │
├────────────────────┬─────────────────┬──────────────────────────┤
│                    │   VITALS         │   AI TRIAGE              │
│   VIDEO FEED       │  Heart Rate      │  Severity: ████░ 4/5    │
│                    │  127 bpm [conf]  │  Medical emergency       │
│   (live WebRTC)    │                 │  Caller: Panicked        │
│   [🔇] [END]      │  Breathing Rate  │  Can speak: No           │
│   (overlay ctrls)  │  22 /min [conf] │  Keywords: chest, pain   │
│                    │                 │  Response: Medical        │
├────────────────────┼─────────────────┴──────────────────────────┤
│                    │   📍 MAP — GPS pin (Leaflet.js)              │
│   (video cont.)   │                                              │
└────────────────────┴─────────────────────────────────────────────┘
```

Key layout details:
- 3-column, 2-row CSS grid
- Video panel spans the entire left column (both rows)
- Vitals: top-center panel
- Triage: top-right panel
- Map: bottom row, spans center + right columns
- All panels have `border-radius: 22px` and `box-shadow`
- Video is mirrored (`transform: scaleX(-1)`) since front camera is used

---

## Key Implementation Details

### HTML Structure

```html
<head>
  <link rel="icon" href="/static/logo.svg">
  <title>Visual911 — Dispatcher</title>
  <!-- Leaflet.js for map (no Chart.js) -->
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css">
  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
</head>
```

```html
<div id="header">
  <span id="conn-status" class="status-dot disconnected"></span>
  <div id="incoming-banner"><!-- pulse dot + ANSWER button --></div>
  <h1>VISUAL911</h1>
</div>

<div id="main-grid">
  <div class="panel" id="video-panel">
    <!-- grid-row: 1 / 3 (spans both rows) -->
    <video id="remote-video" autoplay playsinline muted
           style="transform: scaleX(-1);"></video>
    <div id="video-controls">
      <button id="mute-btn"><!-- SVG mic icon --></button>
      <button id="end-call-btn">End Call</button>
    </div>
  </div>

  <div class="panel" id="vitals-panel"><!-- top center --></div>
  <div class="panel" id="triage-panel"><!-- top right --></div>
  <div class="panel" id="map-panel">
    <!-- grid-column: 2 / 4 (spans center + right) -->
    <div id="map"></div>
  </div>
</div>
```

### CSS Grid

```css
#main-grid {
  flex: 1;
  display: grid;
  grid-template-columns: 1.2fr 1fr 1fr;
  grid-template-rows: 1fr 1fr;
  gap: 14px;
  padding: 14px;
  overflow: hidden;
}
.panel {
  background: #12121a;
  padding: 20px;
  border-radius: 22px;
  box-shadow: 0 2px 16px rgba(0,0,0,0.24);
}
#video-panel { grid-row: 1 / 3; }
#map-panel   { grid-column: 2 / 4; }
```

### Connection Status

Green/red dot in the header showing WebSocket connection state:

```javascript
ws.onopen = () => {
  connStatus.className = 'status-dot connected';  // green
};
ws.onclose = () => {
  connStatus.className = 'status-dot disconnected'; // red
  setTimeout(connectDashboard, 2000); // auto-reconnect
};
```

### TURN Credentials (Dynamic)

No hardcoded TURN credentials. Fetched from the server's REST endpoint:

```javascript
async function fetchTurnCredentials() {
  const resp = await fetch('/api/turn-credentials');
  const data = await resp.json();
  return [
    { urls: 'stun:stun.l.google.com:19302' },
    { urls: 'stun:stun1.l.google.com:19302' },
    {
      urls: data.uris,
      username: data.username,
      credential: data.credential,
    }
  ];
}
```

ICE servers include Google's public STUN servers plus the project's coturn TURN server with HMAC-SHA1 time-limited credentials.

### WebSocket Protocol Auto-Detection

```javascript
const protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
const wsUrl = `${protocol}//${window.location.host}/ws/dashboard`;
```

Automatically uses `wss:` in production and `ws:` for local development.

### WebRTC Flow

The dashboard acts as the WebRTC **answerer** (never the offerer). The iOS app always creates the offer. Signaling routes through `/ws/dashboard` ↔ server ↔ `/ws/signal`.

```javascript
async function handleOffer(offerData) {
  const iceServers = await fetchTurnCredentials();
  peerConnection = new RTCPeerConnection({ iceServers });
  // ... onicecandidate, ontrack handlers
  await peerConnection.setRemoteDescription({ type: 'offer', sdp: offerData.sdp });
  const answer = await peerConnection.createAnswer();
  await peerConnection.setLocalDescription(answer);
  ws.send(JSON.stringify({ type: 'answer', call_id: currentCallId, sdp: answer.sdp }));
}
```

### Video Controls Overlay

Mute and end-call buttons float over the video panel:

```css
#video-controls {
  position: absolute;
  bottom: 24px;
  left: 50%;
  transform: translateX(-50%);
  display: none; /* shown during active call */
}
```

Mute button toggles between mic-on and mic-off SVG icons. End-call button is red with white text.

---

## Panel Behavior Details

### Vitals Panel

- Displays HR and breathing rate from the pre-call Presage scan
- **Always shows values regardless of confidence** — dispatchers need all data even if uncertain
- Confidence bars: green when confidence ≥ 0.4, **orange** when < 0.4
- "Signal lost / low confidence" warning only if **both** readings have confidence < 0.4
- No Chart.js HR graph — just numeric display with confidence indicators
- Labeled "Contactless vitals • 15s pre-call scan"

```javascript
function onVitals(data) {
  const hrConf = data.hrConfidence ?? 0;
  const brConf = data.breathingConfidence ?? 0;

  // Always show values (dispatchers need all data)
  document.getElementById('hr-value').textContent = Math.round(data.hr);
  document.getElementById('br-value').textContent = Math.round(data.breathing);

  // Confidence bars
  document.getElementById('hr-conf').style.width = `${Math.round(hrConf * 100)}%`;
  document.getElementById('hr-conf').style.background = hrConf >= 0.4 ? '#00cc44' : '#ff8800';

  // Signal lost warning at threshold 0.4 (not 0.7)
  signalLost.style.display = (hrConf < 0.4 && brConf < 0.4) ? 'block' : 'none';
}
```

### Triage Panel

- Updates on every `triage_update` from the server (~every 10 seconds)
- Severity dots (1–5) color-coded: green (1–2), yellow (3), red (4–5)
- Full body red flash animation when severity ≥ 4
- Displays: situation summary, emotional state, response type, can_speak, keywords
- No separate `critical_flag` alert — severity communicated via the `severity` field

### Location Map

- Leaflet.js on OpenStreetMap tiles (no API key needed)
- Pin placed on `incoming_call` and updated if location changes
- Zoom level 15 (street level) on pin placement
- Map panel spans center + right columns in the bottom row

---

## Call Lifecycle in Dashboard

### Incoming Call
1. `incoming_call` message received via WS
2. Red pulse banner appears with ANSWER button
3. Map pin placed at caller GPS coordinates
4. Connection status shows green dot

### Answer
1. Dispatcher clicks ANSWER
2. Sends `dispatcher_joined` to server
3. Server starts Gemini analysis task, replays cached vitals
4. iOS sends WebRTC offer → server → dashboard
5. Dashboard creates answer, sends back
6. Video appears, controls overlay shown

### Active Call
- Video: live P2P WebRTC stream (mirrored, starts muted, click to unmute)
- Vitals: displayed from pre-call scan data
- Triage: updates every ~10 seconds from Gemini batch analysis
- Mute button: toggles dashboard mic icon (visual only — mute is local)

### End Call
`onCallEnded()` performs a full reset:
```javascript
function onCallEnded(data) {
  // Close WebRTC
  if (peerConnection) { peerConnection.close(); peerConnection = null; }
  document.getElementById('remote-video').srcObject = null;

  // Reset vitals
  document.getElementById('hr-value').textContent = '--';
  document.getElementById('br-value').textContent = '--';
  document.getElementById('hr-conf').style.width = '0%';
  document.getElementById('br-conf').style.width = '0%';

  // Reset triage
  document.getElementById('situation').textContent = 'Waiting for analysis...';
  document.getElementById('severity-label').textContent = '--';
  // ... reset all severity dots, emotional-state, etc.

  // Reset map marker
  if (mapMarker) { map.removeLayer(mapMarker); mapMarker = null; }

  // Reset mute button to unmuted state
  // Hide video controls, incoming banner
  currentCallId = null;
}
```

---

## Styling Notes

- Dark theme: `#0a0a0f` body, `#12121a` panels
- Font: `'SF Pro', system-ui, sans-serif`
- Panel border-radius: `22px` with `box-shadow: 0 2px 16px rgba(0,0,0,0.24)`
- Header: `#111118` background, bold "VISUAL911" title
- Incoming call banner: red border on `#1a0000` background with pulse animation
- Answer button: green `#00cc44`
- End call button: red `#cc0000`
- Severity dots colored dynamically (green/yellow/red)
- Video mirrored for front-camera view: `transform: scaleX(-1)`
