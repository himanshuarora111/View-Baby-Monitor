# View Baby Monitor — Plan V4

> Evolution of Plan.md → plan_v2.md → plan_v3.md
>
> This version consolidates architectural decisions, unresolved risks, implementation strategies, failure handling, and production considerations.

---

# 1. Vision

View Baby Monitor is a privacy-first, local-first baby monitoring system designed to:

* avoid cloud video storage
* avoid subscriptions
* minimize server dependency
* allow users to self-host if desired
* provide low-latency live monitoring
* support old Android devices as Baby Units
* support future iOS deployment
* remain lightweight and inexpensive to operate

Core principles:

* Baby Unit is the source of truth
* P2P media whenever possible
* no mandatory cloud storage
* local-first pairing
* graceful degradation on weak hardware
* optional monetization through lightweight ads only on hosted builds

---

# 2. High-Level Architecture

## 2.1 Components

### Baby Unit

Primary authority device.

Responsibilities:

* camera capture
* microphone capture
* motion detection
* cry detection
* stream quality control
* permission approvals
* pairing approval
* sensor activation state
* thermal management
* battery optimization
* WebRTC media source

### Parent Unit

Viewer/controller.

Responsibilities:

* stream viewing
* audio talkback
* stream controls
* receiving alerts
* initiating connection
* sensor toggling

### Signaling Server

Stateless signaling relay only.

Responsibilities:

* WebSocket/STOMP relay
* ICE exchange
* SDP exchange
* temporary session coordination
* optional push-notification routing

The signaling server must NEVER:

* store video
* proxy media
* persist sensitive data
* become required for LAN-only mode

---

# 3. Technology Decisions

## 3.1 Mobile Framework

### Chosen

React Native

### Reason

* future iOS support
* single shared codebase
* faster solo development
* acceptable tradeoff for MVP

### Risks

React Native becomes difficult in:

* camera lifecycle handling
* background execution
* audio routing
* WebRTC native integrations
* OEM battery optimizations

### Mitigation Strategy

Heavy/native-sensitive functionality should be isolated behind:

* native Android modules
* service abstraction layers
* minimal JS dependency for media pipelines

Potential future architecture:

* RN for UI
* native modules for media engine

---

## 3.2 Backend

### Chosen

Spring Boot

### Reason

* existing expertise
* rapid iteration
* production familiarity
* stable concurrency model

### Tradeoff

Higher memory usage compared to:

* Go
* Node.js

### Decision

Continue with Spring Boot unless:

* scaling problems appear
* free-tier hosting becomes insufficient

Since signaling traffic is lightweight and mostly stateless, Spring Boot should remain acceptable for early-stage deployment.

---

## 3.3 Signaling Protocol

### Chosen

WebSocket + STOMP

### Reason

* structured topic routing
* easier debugging
* organized event model
* simpler scaling path later

### Tradeoff

STOMP adds:

* framing overhead
* additional abstraction

### Decision

Keep STOMP initially.

Potential future migration:

* raw WebSocket JSON
* Socket.IO alternative

Only optimize if profiling shows bottlenecks.

---

# 4. WebRTC Strategy

## 4.1 Media Transport

### Chosen

WebRTC

### Features

* DTLS-SRTP encryption
* low latency
* NAT traversal
* built-in echo cancellation support
* adaptive bitrate
* mobile optimized

---

## 4.2 TURN/STUN Strategy

## Current Understanding

STUN alone is NOT enough.

Carrier NATs frequently fail direct P2P.

Without TURN:

* many remote connections will fail
* mobile-to-mobile reliability becomes poor

---

## Recommended Strategy

### Phase 1

LAN-first.

Use:

* STUN only
* local network discovery

This minimizes infrastructure cost.

---

### Phase 2

Optional self-hosted TURN.

Recommended:

* coturn

Users may:

* self-host
* use their own VPS
* use local TURN server

---

### Phase 3

Hosted TURN (optional premium feature)

Possible monetization path.

---

## Important Decision

Media must ALWAYS remain encrypted.

Even on LAN.

Reason:

* modern home networks are not fully trusted
* WebRTC already provides encryption
* little additional complexity

Therefore:

* no unencrypted transport mode
* all media flows through secure WebRTC channels

---

# 5. Authentication & Pairing

## 5.1 Design Philosophy

Since this is primarily local-first:

* avoid complex account systems
* avoid mandatory cloud identity
* avoid persistent user tracking

---

## 5.2 Pairing Flow

### Recommended Final Design

1. Parent requests pairing
2. Baby Unit generates:

   * one-time session token
   * short expiration
   * device nonce
3. QR code displayed
4. Parent scans QR
5. Baby Unit explicitly approves request
6. Parent becomes trusted device
7. Token immediately invalidated

---

## 5.3 Token Strategy

### Recommendation

Use short-lived signed tokens.

Not full auth infrastructure.

Reason:

* simplifies connection security
* prevents malformed session requests
* easier future expansion

---

## 5.4 Token Storage

### Android

Use Android Keystore.

Store:

* signing keys
* trusted device IDs
* pairing secrets

---

## 5.5 Replay Protection

### Required

* one-time-use tokens
* nonce validation
* short expiry (5 minutes)
* manual Baby Unit approval

This prevents:

* screenshot replay attacks
* reused pairing attempts

---

## 5.6 Trusted Device Model

After first approval:

* Parent device becomes trusted
* future reconnects skip QR scan
* Baby Unit may revoke trust manually

Recommended UI:

* "Trusted Devices" screen
* revoke access option

---

# 6. Mobile Media Stack

## 6.1 Recommended React Native WebRTC Library

### Recommendation

react-native-webrtc

Reason:

* most mature RN WebRTC option
* widely used
* active enough ecosystem

### Important Note

Keep WebRTC integration isolated.

Create:

* media service layer
* signaling abstraction
* native wrappers

Avoid spreading direct WebRTC calls throughout the app.

---

## 6.2 Camera Stack

### Recommendation

CameraX on Android.

Avoid low-level Camera2 initially.

Reason:

* lifecycle handling easier
* fewer OEM issues
* better compatibility
* simpler threading

---

## 6.3 Background Execution Strategy

This is the hardest technical challenge.

### Required Components

* Foreground Service
* Wake Locks (minimal)
* Battery Optimization guidance
* Reconnection service
* Camera lifecycle recovery

---

## 6.4 OEM Battery Killers

Major problem vendors:

* Xiaomi
* Oppo
* Vivo
* Realme
* Samsung

### Required UX

Guided setup:

* disable battery optimization
* allow background execution
* auto-start permissions

Future feature:

Vendor-specific onboarding instructions.

---

# 7. Audio System

## 7.1 Goal

Simple reliable two-way audio.

Avoid overly complex DSP systems.

---

## 7.2 Recommended Strategy

Rely primarily on:

* WebRTC built-in AEC
* built-in AGC
* built-in Noise Suppression

Avoid custom DSP initially.

---

## 7.3 Audio Constraints

Recommended:

* mono audio
* low bitrate
* voice optimized profile

---

# 8. Sensor System

## 8.1 Motion Detection

### Recommended MVP

Simple luminance-based frame differencing.

Process:

1. downscale frame
2. convert to grayscale/luminance
3. compare pixel deltas
4. adaptive threshold
5. debounce events

Advantages:

* lightweight
* offline
* CPU friendly
* battery friendly

---

## 8.2 False Positive Mitigation

Required:

* adaptive thresholds
* motion cooldown window
* ignore tiny pixel changes
* lighting normalization

---

## 8.3 Cry Detection

### Important Reality

Accurate cry detection is difficult.

### Recommended MVP

Do NOT use ML initially.

Use heuristic approach:

* audio amplitude monitoring
* frequency band analysis
* sustained sound duration
* repeated pattern detection

Goal:

* good-enough alerts
* not medical-grade accuracy

---

## 8.4 Sensor Activation Model

Clarification:

Sensors are NOT always active.

Parent manually enables:

* sleep monitoring mode
* cry detection
* motion detection

Reason:

* battery savings
* reduced false alerts
* intentional monitoring periods

---

# 9. Streaming Strategy

## 9.1 Video Quality

Adaptive quality required.

Parameters:

* resolution
* FPS
* bitrate

---

## 9.2 Sentry Mode

### Decision

Do NOT fully stop video.

Instead:

* reduce FPS aggressively
* reduce bitrate
* optionally freeze preview

Reason:

* faster reconnect
* lower renegotiation risk
* reduced latency on wake-up

Recommended sentry profile:

* 1 FPS
* low bitrate
* audio optional

---

## 9.3 Multi-Viewer Support

### Initial Recommendation

Maximum 2 viewers.

### Quality Policy

Baby Unit controls quality globally.

Avoid per-viewer transcoding initially.

Reason:

* simpler implementation
* lower CPU usage
* avoids multiple encoders

Future possibility:

* simulcast
* layered streams

---

# 10. State & Recovery

## 10.1 Persistence

### Recommendation

Persist lightweight session state only.

Store:

* trusted devices
* sensor settings
* active mode
* quality preferences
* last connection state

### Storage Recommendation

* encrypted shared preferences
* AsyncStorage for non-sensitive state

---

## 10.2 Crash Recovery

### Goal

Automatic recovery after:

* app restart
* temporary network loss
* signaling reconnect

---

## 10.3 Session TTL

Recommended:

30-second reconnect window.

If reconnect succeeds:

* reuse session
* ICE restart
* avoid full renegotiation if possible

---

## 10.4 Ghost Session Handling

If timeout exceeded:

* old session destroyed
* new session created cleanly
* stale viewers evicted

This avoids duplicate sessions.

---

# 11. Thermal & Battery Management

## 11.1 Display Strategy

Use BOTH:

* black screen overlay
* brightness reduction

Reason:

* OLED savings
* LCD compatibility

---

## 11.2 Thermal Monitoring

Required for long-term stability.

### Recommended Reactions

On thermal increase:

1. reduce FPS
2. reduce bitrate
3. reduce resolution
4. disable optional sensors
5. notify parent

---

## 11.3 Battery Management

Critical thresholds:

* Low Battery Warning
* Battery Saver Mode
* Critical Shutdown Mode

Critical mode:

* audio-only fallback
* sensor-only mode
* emergency disconnect warning

---

# 12. Failure Modes & Recovery

This section is mandatory for production readiness.

---

## 12.1 Network Switching

Handle:

* WiFi → Mobile
* Mobile → WiFi
* IP changes
* temporary disconnects

Use:

* ICE restart
* reconnect backoff

---

## 12.2 Camera Failure

Possible causes:

* camera in use
* OEM crash
* permission revoked
* thermal shutdown

Recovery:

* attempt restart
* notify parent
* fallback to audio-only

---

## 12.3 App Killed by OEM

Detection:

* heartbeat timeout
* service monitoring

Recovery:

* restart service
* notify user
* onboarding instructions

---

## 12.4 TURN Unavailable

Fallback:

* LAN-only mode
* retry STUN-only connection
* notify parent

---

## 12.5 Overheating

Automatic degradation:

* FPS reduction
* resolution reduction
* optional stream disable

---

## 12.6 Permission Revoked

If camera/microphone revoked:

* immediate stream stop
* clear UI state
* permission recovery flow

---

# 13. Monetization Strategy

## Goals

* remain privacy-first
* avoid subscriptions for core use
* avoid aggressive monetization

---

## Proposed Model

### Self-Hosted / Local Mode

Completely free.

No ads.

---

### Hosted Convenience Mode

Optional lightweight ads.

Possible premium features later:

* hosted TURN
* remote relay
* push relay reliability
* cloud backup configs

---

# 14. License Strategy

## Current Intent

Protect against:

* commercial cloning
* app-store redistribution
* hosted competitors

While allowing:

* personal usage
* learning
* self-hosting

---

## Important Clarification

This is NOT fully open-source.

It is source-available.

---

## Future Possibility

Dual-license model.

Example:

* community/self-host license
* commercial license

This may improve:

* contributions
* ecosystem growth
* legal clarity

---

# 15. Recommended Development Order

## Phase 1 — Core Connectivity

Build:

* signaling server
* QR pairing
* Baby/Parent roles
* basic WebRTC stream
* local network support

---

## Phase 2 — Stability

Add:

* reconnect logic
* background service
* battery handling
* thermal handling
* trusted devices

---

## Phase 3 — Sensors

Add:

* motion detection
* cry detection
* sleep mode
* notifications

---

## Phase 4 — Production Hardening

Add:

* TURN support
* failure recovery
* metrics
* diagnostics
* onboarding UX

---

## Phase 5 — Cross Platform

Add:

* iOS support
* native optimizations
* advanced audio tuning

---

# 16. Biggest Engineering Risks

## Highest Risk

Android OEM background execution reliability.

This is harder than:

* WebRTC
* signaling
* streaming

---

## Other Major Risks

* thermal throttling
* RN native integration maintenance
* TURN infrastructure costs
* false positive sensor alerts
* battery drain on old devices

---

# 17. Biggest Architectural Strength

"Baby Unit is the source of truth."

This simplifies:

* security
* authority
* synchronization
* privacy
* recovery
* scaling
* local-first operation

This should remain a foundational principle.

---

# 18. Final Engineering Guidance

Do NOT over-engineer V1.

The hardest part is:

* reliability
* lifecycle recovery
* OEM compatibility
* thermal stability

Not AI.
Not ML.
Not advanced analytics.


# 19. AI Build Directives (Executive Constraints)

The following directives are strict architectural constraints. When implementing the features below, prioritize these approaches to ensure stability, simplicity, and successful deployment without relying on overly complex native workarounds.

### 19.1 Background Execution & Camera Lifecycle
* **Decision:** Keep the Baby Unit in the Foreground (OLED Blackout).
* **AI Directive:** Do not attempt to write custom Java/Kotlin native modules to run the camera and WebRTC in a background service. Instead, keep the Baby Unit app in the foreground at all times. Implement an OLED "Stealth Mode" (a `#000000` full-screen UI overlay) to turn off the screen pixels, and use Android's `Wakelock` to prevent the device CPU from sleeping.

### 19.2 Audio Echo & Feedback Loops
* **Decision:** Software Mic Muting (Walkie-Talkie Mode).
* **AI Directive:** Do not rely solely on hardware Acoustic Echo Cancellation (AEC). Implement a software-level "Mic Mute" state on the Baby Unit. When the Parent Unit holds the "Talk" button, send a Data Channel message to the Baby Unit to mute its microphone. When the Parent releases the button, send a payload to unmute the Baby Unit.

### 19.3 Thermal Management Triggers
* **Decision:** Battery Temperature Proxy.
* **AI Directive:** Use a React Native device info library to read the Android battery temperature. If the battery temperature exceeds 40°C (104°F), trigger the thermal degradation protocol (lower the WebRTC bitrate and drop FPS). Do not use a time-based guess for thermal throttling.

### 19.4 Pairing JWT "Time Drift"
* **Decision:** 2-Minute Leeway.
* **AI Directive:** When validating the pairing JWT on the Parent Unit, implement a 120-second (2-minute) "clock skew" leeway. Accept tokens that appear up to 2 minutes expired or 2 minutes from the future to account for normal Android system clock drift.

### 19.5 Sentry Mode Video Throttling
* **Decision:** WebRTC Parameter Throttling.
* **AI Directive:** To achieve the 1 FPS Sentry mode, use the WebRTC `RTCRtpSender.setParameters()` API to dynamically drop the `maxFramerate` to 1. Do not implement manual canvas drawing or custom frame capturing routines to achieve this.

### 19.6 The Local LAN Disconnect
* **Decision:** Proactive Reconnect.
* **AI Directive:** Use React Native `NetInfo` on the Parent Unit. If the Parent Unit detects a network switch (e.g., WiFi dropping to 4G), it must immediately send a 'Network Changed' payload to the Signaling Server, instructing the Baby Unit to restart ICE candidates. Do not wait for the 30-second Ghost timeout to realize the local UDP socket has died.

A reliable low-latency monitor with:

* stable reconnects
* decent audio
* reasonable battery usage
* secure local pairing

is already a strong product.
