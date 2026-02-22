# iOS App

## Overview

The iOS app is a Swift/SwiftUI application that manages the full call lifecycle. It coordinates three hardware subsystems (Presage camera, WebRTC camera, AVAudioEngine microphone), three WebSocket connections to the Vultr server, and one P2P WebRTC connection to the dispatcher browser.

Physical device required. Simulator has no camera. Built with XcodeGen from `project.yml`.

---

## Dependencies

Add via Swift Package Manager (defined in `project.yml`):

| Package | URL | Version | Purpose |
|---|---|---|---|
| SmartSpectra | `https://github.com/Presage-Security/SmartSpectra` | `>= 1.0.0` | Contactless vitals |
| WebRTC | `https://github.com/stasel/WebRTC` | `>= 125.0.0` | P2P video/audio |

---

## Project Structure

```
ios/
├── project.yml               # XcodeGen project definition
├── Config.swift               # Server URLs — reads API key from Secrets.swift
├── Secrets.swift              # Presage API key (gitignored)
├── Secrets.swift.example      # Template for Secrets.swift
├── TURNCredentials.swift      # HMAC-based TURN credential generation
├── CallManager.swift          # Central state machine, owns all subsystems
├── PresageManager.swift       # Presage SDK wrapper with ScanFeedback
├── WebRTCManager.swift        # RTCPeerConnection, capturer, FrameGrabber
├── AudioTap.swift             # AVAudioEngine tap → /ws/audio WebSocket
├── SignalingClient.swift      # /ws/signal WebSocket
├── VitalsClient.swift         # /ws/vitals WebSocket
├── ContentView.swift          # Root view — switches on CallState
├── Visual911App.swift         # @main App entry point
└── Views/
    ├── IdleView.swift         # SOS button, greeting, location display
    ├── ScanningView.swift     # Camera preview, SDK feedback, vitals cards
    ├── InitiatingView.swift   # "Vitals Captured" confirmation, waiting for dispatcher
    ├── ActiveCallView.swift   # Full-screen local video, mute/end controls
    ├── RTCLocalVideoView.swift # UIViewRepresentable for RTCMTLVideoView
    └── EKGLogoView.swift      # EKG waveform logo matching web app
```

---

## Config.swift

```swift
enum Config {
    static let presageApiKey = Secrets.presageApiKey
    static let serverHost    = "wss://visual911.mooo.com"

    // WebSocket endpoints
    static let signalURL  = "\(serverHost)/ws/signal"
    static let audioURL   = "\(serverHost)/ws/audio"
    static let vitalsURL  = "\(serverHost)/ws/vitals"
}
```

API key is stored in `Secrets.swift` (gitignored). Copy `Secrets.swift.example` and fill in real values.

---

## CallManager.swift

Central coordinator. Owns the state machine and holds references to all subsystems. Key features beyond the basic lifecycle:

- **Smart scan completion**: Scan finishes early when both HR and BR signals stabilize (minimum 5s, 2s error cooldown), or hard-times out at 15s
- **Scan feedback**: Real-time `ScanFeedback` from Presage SDK status codes (face not found, too dark, etc.) and vitals signal stability
- **Retry scan**: User can restart the scan if conditions are poor
- **Vitals timing**: Sends `call_initiated` first, then vitals 0.3s later to avoid race condition on server
- **Mute toggle**: Controls WebRTC audio track enable/disable
- **Thread-safe cleanup**: `endCall()` guards against re-entrant calls (ICE failed + signaling race) and always runs on main thread
- **Transient ICE handling**: ICE `.disconnected` doesn't trigger teardown (may recover); only `.failed` ends the call
- **Vitals resend on connect**: Resends cached vitals when ICE reaches `.connected` state

```swift
import SwiftUI
import Combine
import CoreLocation
import WebRTC

enum CallState {
    case idle
    case scanning          // Presage running
    case initiating        // WebSockets open, waiting for dispatcher
    case connecting        // WebRTC handshake in progress
    case active            // All systems live
    case cleanup
}

class CallManager: ObservableObject {
    @Published var state: CallState = .idle
    @Published var lastVitals: VitalsReading?
    @Published var lastLocation: CLLocationCoordinate2D?
    @Published var isMuted: Bool = false
    @Published var scanFeedback: ScanFeedback?
    @Published var scanComplete: Bool = false

    private var callId: String?
    private let presage = PresageManager()
    private var webrtc: WebRTCManager?
    private var audioTap: AudioTap?
    private var signalingClient: SignalingClient?
    private var vitalsClient: VitalsClient?
    private var locationManager = CLLocationManager()
    private var cancellables = Set<AnyCancellable>()
    private var scanTimeoutWork: DispatchWorkItem?
    private var scanStartTime: Date?
    private var lastErrorTime: Date?
    private let minScanDuration: TimeInterval = 5.0
    private let errorCooldown: TimeInterval = 2.0

    func onSOSPressed() { ... }
    private func startPresageScan() { ... }
    private func tryFinishScan() { ... }
    private func finishScan() { ... }
    func retryScan() { ... }
    private func initiateCall() { ... }
    func onDispatcherReady() { ... }
    func toggleMute() { ... }
    func renderLocalVideo(to renderer: RTCVideoRenderer) { ... }
    func endCall(reason: String = "caller_ended") { ... }
}
```

### Scan Logic

The scan has three ways to finish:
1. **Early finish** (`tryFinishScan`): Both `hrStable` and `brStable` are true, minimum 5s has passed, no SDK error active, no error in last 2s
2. **Hard timeout**: 15 seconds regardless of quality
3. **User retry** (`retryScan`): Cancels and restarts the scan

SDK status codes (face position, lighting) take priority over vitals-based feedback. Positive "Signal stable" feedback is only shown when the SDK isn't reporting an error.

### Call Lifecycle

On `initiateCall()`:
1. Opens signaling, vitals, and audio WebSockets
2. Sends `call_initiated` with location
3. After 0.3s delay, sends cached vitals (if any)

On `onDispatcherReady()`:
1. Transitions to `.connecting`
2. Creates `WebRTCManager`, links it to `AudioTap` for frame capture
3. Creates WebRTC offer

On `endCall()`:
1. Guards: must be on main thread, must not already be in `.cleanup`/`.idle`
2. Captures and nils all subsystem references atomically
3. Sends `call_ended`, disconnects everything, resets all state

### Delegate Extensions

```swift
// SignalingClientDelegate — handles dispatcher_ready, call_ended, SDP, ICE
// WebRTCManagerDelegate — handles ICE candidates, connection state changes, SDP
```

ICE `.disconnected` is treated as transient (logged but not acted on). Only `.failed` triggers `endCall()`. On `.connected`/`.completed`, vitals are resent to ensure the dispatcher gets them.

---

## PresageManager.swift

Wraps `SmartSpectraSwiftSDK.shared` and `SmartSpectraVitalsProcessor.shared`. Publishes readings and scan feedback.

```swift
import SmartSpectraSwiftSDK
import Combine

struct VitalsReading {
    let hr: Double
    let hrStable: Bool
    let breathing: Double
    let brStable: Bool
    let timestamp: Date
}

struct ScanFeedback: Equatable {
    let icon: String      // SF Symbol name
    let message: String   // User-facing text
    let isError: Bool     // true = problem, false = informational

    static func from(_ code: StatusCode) -> ScanFeedback? { ... }

    // Vitals-driven feedback (not from SDK status)
    static let signalStable = ScanFeedback(icon: "checkmark.circle.fill", message: "Signal stable — measuring vitals", isError: false)
    static let signalWeak = ScanFeedback(icon: "exclamationmark.triangle.fill", message: "Weak signal — hold still in good lighting", isError: true)
}

class PresageManager: ObservableObject {
    @Published var latestReading: VitalsReading?
    @Published var scanFeedback: ScanFeedback?
    private let sdk = SmartSpectraSwiftSDK.shared
    private let processor = SmartSpectraVitalsProcessor.shared

    init() {
        sdk.setApiKey(Config.presageApiKey)
        sdk.setSmartSpectraMode(.continuous)
        sdk.setCameraPosition(.front)
        sdk.setImageOutputEnabled(true)  // enables camera preview via processor.imageOutput
    }

    func startMeasuring() {
        processor.startProcessing()
        processor.startRecording()
        // Observes processor.$lastStatusCode → scanFeedback
        // Observes sdk.$metricsBuffer → latestReading (with .dropFirst() to skip stale values)
    }

    func stopMeasuring() {
        processor.stopRecording()
        processor.stopProcessing()
    }
}
```

**Key SDK details:**
- `.ok` status is NOT used for positive feedback — it's unreliable (can report `.ok` while C++ layer logs "too dark")
- Positive feedback comes from vitals signal stability (`hrStable`/`brStable`) instead
- Status codes handled: `imageTooDark`, `imageTooBright`, `noFacesFound`, `moreThanOneFaceFound`, `faceNotCentered`, `faceTooBigOrTooSmall`, `chestTooFarOrNotEnoughShowing`
- Uses `.dropFirst()` on metricsBuffer to skip stale initial value from previous session
- `setImageOutputEnabled(true)` provides camera preview via `processor.imageOutput` (UIImage)

---

## WebRTCManager.swift

Manages `RTCPeerConnection` and `RTCCameraVideoCapturer`. After Presage stops, this takes over the front camera.

```swift
import WebRTC
import UIKit

class WebRTCManager: NSObject {
    weak var delegate: WebRTCManagerDelegate?
    private var peerConnection: RTCPeerConnection!
    private var capturer: RTCCameraVideoCapturer?
    private var localVideoTrack: RTCVideoTrack?
    private let frameGrabber = FrameGrabber()

    // ... setup, offer/answer, ICE handling ...

    func captureJPEG() -> Data? { frameGrabber.captureJPEG() }
    func setAudioMuted(_ muted: Bool) { ... }
    func renderLocalVideo(to renderer: RTCVideoRenderer) { ... }
}
```

### Key differences from docs template:

**ICE Servers** use Google STUN servers plus the project TURN server:
```swift
config.iceServers = [
    RTCIceServer(urlStrings: ["stun:stun.l.google.com:19302"]),
    RTCIceServer(urlStrings: ["stun:stun1.l.google.com:19302"]),
    RTCIceServer(
        urlStrings: ["turn:visual911.mooo.com:3478", "turns:visual911.mooo.com:5349"],
        username: generatedTURNUsername(),
        credential: generatedTURNCredential()
    ),
]
```

**Camera resolution**: Picks format closest to 1280x720 (not highest available) to avoid overwhelming the WebRTC encoder.

**Camera retry**: Retries camera start up to 3 times with 1s delay on failure.

**`createOffer`** takes a `callId` parameter (included in the SDP message).

**FrameGrabber**: A private `RTCVideoRenderer` class that retains the latest `CVPixelBuffer` from the video track and converts it to JPEG on demand:
```swift
private class FrameGrabber: NSObject, RTCVideoRenderer {
    func renderFrame(_ frame: RTCVideoFrame?) { /* retain CVPixelBuffer */ }
    func captureJPEG(compressionQuality: CGFloat = 0.5) -> Data? { /* CIImage → UIImage → JPEG */ }
}
```

---

## AudioTap.swift

Runs parallel to WebRTC audio. Taps the microphone via AVAudioEngine and streams 16kHz mono PCM to `/ws/audio`. Also sends periodic JPEG frames for Gemini scene awareness using a weak reference to `WebRTCManager`.

```swift
class AudioTap {
    private let callId: String
    private var engine = AVAudioEngine()
    private var webSocket: URLSessionWebSocketTask?
    private var frameTimer: Timer?
    weak var webRTCManager: WebRTCManager?  // Set by CallManager after WebRTC init

    func start() {
        setupAudioSession()   // .playAndRecord + .mixWithOthers
        connectWebSocket()    // /ws/audio?call_id=...
        setupEngine()         // AVAudioEngine tap → 16kHz PCM → WebSocket
        scheduleFrames()      // Timer every 2s → captureAndSendFrame()
    }

    private func captureAndSendFrame() {
        guard let jpeg = webRTCManager?.captureJPEG() else { return }
        // Send as JSON: { "type": "frame", "data": "<base64>" }
    }
}
```

**Note:** WebSocket requests include `ngrok-skip-browser-warning: true` header for development via ngrok tunnels.

---

## SignalingClient.swift

WebSocket wrapper for `/ws/signal`. Forwards raw JSON between iOS and the server.

```swift
class SignalingClient: NSObject {
    weak var delegate: SignalingClientDelegate?
    private let callId: String
    private var webSocket: URLSessionWebSocketTask?

    func connect() {
        // URL: Config.signalURL?call_id=...&role=caller
        // Includes ngrok-skip-browser-warning header
    }
    func send(_ dict: [String: Any]) { ... }
    func disconnect() { ... }
}
```

---

## VitalsClient.swift

Sends vitals JSON to `/ws/vitals`. Called once at call start with cached vitals from the Presage scan. Resent on WebRTC connect to ensure dispatcher receives them.

```swift
class VitalsClient {
    func send(reading: VitalsReading) {
        // Converts hrStable/brStable to confidence values:
        // stable → 1.0, unstable → 0.5 (not 0.0, so dashboard still shows the value)
        let payload: [String: Any] = [
            "type": "vitals",
            "call_id": callId,
            "hr": reading.hr,
            "hrConfidence": reading.hrStable ? 1.0 : 0.5,
            "breathing": reading.breathing,
            "breathingConfidence": reading.brStable ? 1.0 : 0.5,
            "timestamp": reading.timestamp.timeIntervalSince1970
        ]
    }
}
```

---

## Views

### ContentView.swift

Root view that switches based on `CallState`:

| State | View |
|---|---|
| `.idle` | `IdleView` — SOS button, greeting, reverse-geocoded location |
| `.scanning` | `ScanningView` — Camera preview, SDK feedback banner, vitals cards, progress bar |
| `.initiating`, `.connecting` | `InitiatingView` — Checkmark, captured vitals summary, location, "Waiting for dispatcher" |
| `.active` | `ActiveCallView` — Full-screen local video, mute/end call buttons |
| `.cleanup` | ProgressView "Ending call…" |

### ScanningView.swift

Uses `SmartSpectraVitalsProcessor.shared.imageOutput` for the camera preview (UIImage from the Presage SDK). Displays:
- Mirrored camera preview
- Scan feedback banner (red for errors, green for stable signal) — driven by `callManager.scanFeedback`
- Progress bar (0–15s with snap-to-full on early completion via `scanComplete`)
- Vitals cards showing HR and BR with stability indicators

### InitiatingView.swift

Shows captured vitals in compact side-by-side cards, GPS coordinates, and a cancel button. Displays "Waiting for dispatcher to answer..." with an amber status dot.

### ActiveCallView.swift

Full-screen `RTCLocalVideoView` (Metal renderer) with overlay controls:
- Mute/Unmute button
- End call button
- "Dispatcher can see your video and location" label

---

## TURNCredentials.swift

Generates time-limited HMAC-SHA1 credentials for coturn:

```swift
import CryptoKit

func generateTURNCredentials(secret: String) -> (username: String, credential: String) {
    let expiry = Int(Date().timeIntervalSince1970) + 3600  // 1 hour
    let username = "\(expiry):visual911user"
    let key = SymmetricKey(data: Data(secret.utf8))
    let mac = HMAC<Insecure.SHA1>.authenticationCode(for: Data(username.utf8), using: key)
    let credential = Data(mac).base64EncodedString()
    return (username, credential)
}

// Convenience accessors used by WebRTCManager
func generatedTURNUsername() -> String { ... }
func generatedTURNCredential() -> String { ... }
```

For the hackathon, the coturn shared secret is hardcoded in this file. For production, credentials would be fetched from the server's `/api/turn-credentials` endpoint.

---

## Info.plist (via project.yml)

```yaml
NSCameraUsageDescription: "Visual911 needs camera access to measure your vitals and stream video to the dispatcher."
NSMicrophoneUsageDescription: "Visual911 needs microphone access so the dispatcher can hear you during the emergency call."
NSLocationWhenInUseUsageDescription: "Visual911 sends your location to the dispatcher so emergency services can find you."
NSLocalNetworkUsageDescription: "Visual911 connects to the dispatch server on your local network."
NSAppTransportSecurity:
  NSAllowsLocalNetworking: true
```

---

## Build Checklist

- [ ] Physical iOS device connected (not simulator)
- [ ] `Secrets.swift` created from `Secrets.swift.example` with real Presage API key
- [ ] Server URL in `Config.swift` points to deployed server (`wss://`)
- [ ] TURN secret in `TURNCredentials.swift` matches coturn config
- [ ] Run `xcodegen generate` from `ios/` directory to regenerate Xcode project
- [ ] Both SPM packages resolved (SmartSpectra, stasel/WebRTC)
- [ ] iOS deployment target ≥ 17.0
- [ ] Paid Apple developer account for signing
- [ ] Test Presage alone first (confirm HR readings before integrating WebRTC)
- [ ] Test WebRTC alone second (confirm video reaches browser before adding audio tap)
- [ ] Test full pipeline last
