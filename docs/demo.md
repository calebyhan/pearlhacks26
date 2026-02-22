# Demo

## Scenario

A high-risk patient at home presses SOS on their phone. They cannot speak — elevated heart rate and rapid breathing are visible through contactless vitals. The AI detects distress from ambient audio and the vitals reading. The dispatcher sees all of this in real time and can dispatch appropriate help.

Target duration: 3 minutes. Never go longer than 10 minutes (Gemini API budget).

---

## Roles

| Person | Role | Device |
|---|---|---|
| Teammate A | Caller | iPhone (physical device, app installed) |
| Teammate B | Dispatcher | Laptop browser (visual911.mooo.com) |
| Teammate C | Presents / narrates | Free to walk judges through what's happening |

---

## Pre-Demo Setup (15 minutes before)

### Environment
- [ ] Room has ≥60 lux lighting on Teammate A's face (overhead lights on, face visible)
- [ ] Teammate A's phone propped 1–2ft from face, front camera pointing at face
- [ ] Both devices on the same WiFi (preferred) or Teammate A on LTE
- [ ] Laptop browser has visual911.mooo.com open and dispatcher dashboard loaded
- [ ] Dashboard `/ws/dashboard` WebSocket connected (green dot in header, no reconnecting)

### Devices
- [ ] iPhone charged, screen timeout disabled (Settings → Display → Never)
- [ ] Visual911 app installed and launched — verify no crashes on open
- [ ] Presage API key confirmed working (check last test run logs)
- [ ] coturn running on Vultr: `systemctl status coturn`
- [ ] Server running on Vultr: `systemctl status visual911`
- [ ] Server logs open in terminal: `journalctl -u visual911 -f`

### Vitals Boost (optional but impactful)
Have Teammate A do 30 seconds of jumping jacks or brisk walking right before the demo. Natural HR elevation (90–130 bpm) reads much more dramatically on the dispatcher dashboard than resting HR (~65 bpm). This is the difference between a boring demo and a compelling one.

---

## Demo Script

### Opening (Teammate C narrates)

> "Visual911 is a FaceTime-style emergency call system designed for people who can't speak during a crisis — or who can't reach a phone to dial. We use the phone's front camera to measure contactless vital signs, stream live video to a dispatcher, and run AI triage in the background using Gemini."

### Step 1: Press SOS (0:00)

Teammate A taps the SOS button.

> "When SOS is pressed, the app first does a 15-second contactless vitals scan using the Presage SmartSpectra SDK — measuring heart rate and breathing rate just from the camera, no wearable required."

The app shows a scanning state. Ideally HR reads elevated from the jumping jacks.

### Step 2: Incoming Call (0:15)

The dispatcher dashboard shows the incoming call banner with GPS coordinates.

> "The dispatcher's dashboard immediately shows an incoming call with the caller's location. On the map you can see the pin placed at the caller's exact position."

Teammate B clicks ANSWER.

### Step 3: WebRTC Connects (0:25)

Live video appears in the video panel.

> "WebRTC establishes a direct peer-to-peer connection. The dispatcher now has live video and audio from the caller."

### Step 4: Vitals Appear (0:30)

The vitals panel shows HR (elevated) and breathing rate from the pre-call scan.

> "The vitals captured during the pre-call scan appear on the dispatcher's screen. Heart rate is elevated at [X] bpm. Breathing is rapid at [X] per minute. The confidence bar shows how reliable the reading is."

Point to the confidence bar.

### Step 5: AI Triage Activates (0:45–1:30)

Gemini has been receiving audio since the call connected. Every ~10 seconds, the server sends a batch of accumulated audio (as WAV) plus the latest video frame to Gemini's `generateContent` API. Triage panel starts populating.

> "In the background, Gemini analyzes audio from the call every 10 seconds — building a cumulative understanding of the situation. Watch the AI triage panel update."

Have Teammate A make some noise — cough, gasp, say a few distressed words, or just let the ambient silence speak for itself. The AI will detect the lack of speech too.

The panel should show:
- Caller emotional state: distressed / panicked
- Can speak: No (or low confidence)
- Recommended response: Medical
- Severity: 3–4

### Step 6: Severity Escalation (1:30–2:00)

As more audio accumulates, Gemini builds on its previous analysis (cumulative context). The severity score may increase as Gemini detects more distress signals.

> "Each analysis round gives Gemini the previous situation summary as context, so it builds an increasingly accurate picture over time. Watch the severity score update."

The severity dots turn from green/yellow to red as the score increases. At severity ≥ 4, the entire dashboard border flashes red.

### Step 7: Location Map (2:00)

Point to the Leaflet.js map.

> "The caller's GPS coordinates are pinned on the map. A dispatcher can share this with responding units immediately."

### Step 8: End Call (2:30)

Teammate B clicks END CALL.

> "Either party can end the call. The AI session closes, all connections terminate cleanly."

### Closing (Teammate C)

> "Visual911 uses Presage for contactless vitals — no wearable. Gemini for AI triage — understanding not just words but the absence of words, tone, and stress. WebRTC for low-latency video. And Vultr for the infrastructure holding it all together. This is emergency response for the 21st century."

Total time: ~3 minutes.

---

## Credit Budget

### Presage
300 credits total. 1 credit = 30s of measurement.

| Activity | Credits Used |
|---|---|
| Each full demo run | ~6 (15s scan + buffer) |
| Each integration test | ~4 |
| Each unit test of Presage alone | ~2 |
| Buffer for failures/retries | 50 |

Estimated capacity: ~40 demo/test sessions. Do not run Presage in idle. Confirm credit balance at physiology.presagetech.com before the hackathon starts.

### Gemini
250 requests per day on free tier. Each batch `generateContent` call = 1 request. With 10-second intervals, a 3-minute call uses ~18 requests.

| Activity | Requests |
|---|---|
| Each demo run (3 min) | ~18 |
| Each demo run (10 min) | ~60 |
| Daily testing budget | ~50 |

250 RPD is adequate for ~4 full 10-min demo runs per day plus testing. The risk is RPM (10/minute) — with 10-second intervals, one call uses ~6 RPM, leaving room for only one simultaneous call.

---

## What to Do If Things Break

**Presage not reading:**
- Check lighting (≥60 lux). Turn on overhead lights, move near a window.
- Check distance (1–2ft from camera).
- Check credit balance at physiology.presagetech.com.
- Restart the app and try again — sometimes a fresh session clears stuck state.

**WebRTC video not appearing:**
- Check that coturn is running: `systemctl status coturn`
- TURN credentials are fetched dynamically from `/api/turn-credentials` — verify: `curl https://visual911.mooo.com/api/turn-credentials`
- Check that `TURN_SECRET` env var on server matches coturn `static-auth-secret`
- Try connecting both devices on the same WiFi (avoids TURN entirely)
- Open browser devtools → Console for WebRTC errors

**Gemini not producing output:**
- Check server logs: `journalctl -u visual911 -f`
- Look for 429 errors (rate limit) — wait 1 minute
- Check GEMINI_API_KEY is set in `.env` file
- Gemini batch analysis runs every 10 seconds — first triage update appears ~10s after dispatcher joins
- If no audio is being captured, check iOS AudioTap logs

**Dashboard WebSocket not connecting:**
- Refresh the browser
- Check server is running: `systemctl status visual911`
- Check firewall: `ufw status` — port 443 should be ALLOW

**Demo time and something is broken:**
- Fallback: screencast a recording of a working demo run if you made one
- Have a slide showing the architecture and key technical decisions — judges often care more about the engineering choices than a live demo
- Know your story cold: Presage rPPG + Gemini affective audio + WebRTC P2P + Vultr infrastructure. That's compelling even without a perfect live demo.

---

## Judging Talking Points

**Why this matters:**
- 911 callers who can't speak are common: domestic violence, throat injury, hiding from an intruder, cardiac event mid-call
- Contactless vitals give dispatchers medical data before any responder arrives
- AI triage reduces cognitive load on dispatchers handling multiple calls

**Technical depth to highlight:**
- Presage clinical accuracy (<1.62% RMSD for HR vs hospital equipment)
- Gemini batch analysis with cumulative context — each 10s round builds on the previous summary
- The camera conflict problem and why we time-slice instead of run concurrently
- Why we use batch `generateContent` with WAV audio + JPEG frames instead of the Live API (rate limits, reliability)
- Dynamic TURN credentials with HMAC-SHA1 (matching coturn's auth mechanism)
- The single-server asyncio design and why it's appropriate for this use case

**Sponsor track connections:**
- Triad STEM: This is literally an emergency 911 use case
- Presage: We're using their SDK in the exact scenario they described on the use cases page ("Emergency responders can now assess caller's heart rate & breathing through smartphone cameras during 911 calls")
- Gemini: Batch analysis with `generateContent`, native audio understanding, multimodal (audio + video frames) — showcasing practical AI in emergencies
- Vultr: Real deployed infrastructure, not localhost
