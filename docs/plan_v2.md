# View Baby Monitor (Viyu) - Plan v2 📋

**Last Updated:** 2026-05-12

---

## 📑 Table of Contents

1. [Core Architecture & Design](#core-architecture--design)
2. [Mobile App (React Native)](#mobile-app-react-native)
3. [Security & Privacy](#security--privacy)
4. [Battery & Power Management](#battery--power-management)
5. [Sensors & Detection](#sensors--detection)
6. [Networking & Connectivity](#networking--connectivity)
7. [Notifications & Alerts](#notifications--alerts)
8. [Data & Logging](#data--logging)
9. [Deployment & Operations](#deployment--operations)
10. [User Experience & Workflows](#user-experience--workflows)
11. [Edge Cases & Error Handling](#edge-cases--error-handling)
12. [Future Roadmap](#future-roadmap)

---

## Core Architecture & Design

### 1. P2P vs Signaling Trade-offs

**Decision:** Auto-switch to local network when available, fallback to WebRTC via signaling server.

- **Target:** Achieve optimal failover speed (TBD after app development).
- **Primary Connectivity:** Local network (UDP Broadcast) - zero-latency, zero-data usage.
- **Secondary Connectivity:** WebRTC via signaling server - for 4G/5G/remote monitoring.
- **Adaptive ICE:** Auto-restart ICE when network switches (WiFi ↔ 4G).
- **Status:** Will be refined and benchmarked during development.

---

### 2. Mesh Networking Limitations

**Decision:** Support up to **2 simultaneous viewers** (design choice, not technical limit).

- **Rationale:** Battery efficiency on older devices; they may not handle higher load.
- **Scalability:** Future versions can scale to 3-4+ viewers if device performance allows.
- **New Viewer Behavior:** When a 3rd person tries to connect:
  - They see a clear error message: "Maximum 2 viewers connected. Please ask one viewer to disconnect."
  - They're given the option to see who is currently watching.

---

### 3. Master Node Authority & Device Theft

**Decision:** Baby Unit (physical device with baby) has full authority.

- **Normal Operation:** Baby Unit can kick any viewer instantly.
- **Device Theft/Loss:** Parent can unpair the stolen device from their parent app with an "Unpair" button.
- **Security Model:** Once unpairedから the lost device cannot reconnect even if recovered (requires Baby Unit re-initialization).
- **Rationale:** Prevents misuse from unauthorized users connecting to the monitor.

---

## Mobile App (React Native)

### 4. Platform & SDK Strategy

**Decision:** Start with **Android**, plan for iOS.

- **Framework:** React Native (JavaScript/TypeScript).
- **Primary Target:** Android (easier to develop, test, and deploy via GitHub).
- **Minimum SDK:** TBD - will be determined during development (recommend targeting Android 8+).
- **iOS Support:** Planned for future releases (once Android is stable).
- **Rationale:** React Native allows code sharing; easier to add iOS later.

---

### 5. Background Execution & Notifications

**Decision:** Hybrid notification strategy based on app availability.

**For Play Store Builds:**
- Prefer **Firebase Cloud Messaging (FCM)** when available.
- Implement foreground services for persistent background listening.
- Fallback: If FCM fails, rely on foreground service.

**For GitHub APK Builds (Personal Use):**
- Option to keep connection active even when app is backgrounded/killed.
- If app is killed: Loss of notifications is acceptable.
- User must consciously enable this mode (not default).

**Implementation Details:**
- Use Android foreground services for persistent WebSocket connections.
- FCM is preferred but optional (for privacy-first users).
- Document the trade-off: always-on = better notifications but higher battery drain.

---

### 6. State Management Strategy (Redux-like Pattern)

**Decision:** Implement state machine for Active/Sentry/Sleep transitions.

**State Machine Design:**

| State | Video | Audio | Data Channel | When Used |
|-------|-------|-------|--------------|-----------|
| **Active (Live)** | ON (HD) | ON | ON | Parent actively watching |
| **Sentry (Bg)** | OFF | User Toggle | ON | Parent app minimized |
| **Sleep (Idle)** | OFF | OFF | Listener | Baby unit deactivated, awaits wake ping |

**State Sync Mechanism:**
- Baby Unit maintains local "last known state" in persistent storage.
- Parent Unit sends periodic "sync requests" to confirm Baby Unit state.
- **Reconnection within 10 seconds:** Resume seamlessly with stored state.
- **Reconnection after 10 seconds:** Treat as new session, require re-activation.
- **Conflict Resolution:** If parent thinks Active but Baby is Sleep, show error and request re-activation.

**Implementation:**
- Use **Redux, MobX, or Zustand** for centralized state management.
- Store state locally using AsyncStorage or SQLite.
- Sync state via WebRTC Data Channel (Active/Sentry) or Firebase FCM (Sleep wake-up).

---

## Security & Privacy

### 7. QR Code Expiration & Security

**Decision:** QR codes expire after **5 minutes**.

- **QR Code Contents:** Baby Unit IP, device ID, temporary token (5-min TTL).
- **Expiration:** Token is invalidated after 5 minutes, forcing parent to generate new QR.
- **Prevents:** Old screenshots being used to gain unauthorized access.
- **Recovery:** Parent simply scans a fresh QR code from the Baby Unit.

---

### 8. WebRTC Encryption

**Decision:** Use **DTLS-SRTP (Option A - Industry Standard)**.

- **What it does:** WebRTC natively encrypts media via DTLS handshake + SRTP real-time encryption.
- **Pros:** 
  - Industry standard, automatic in WebRTC.
  - Perfect forward secrecy.
  - No manual implementation needed.
  - Proven security.
- **Implementation:** Enable DTLS in PeerConnection config (automatic by default in modern WebRTC).
- **No Double Encryption Needed:** DTLS-SRTP is sufficient for home monitoring.

---

### 9. Signaling Server Trust & Stream Storage

**Decision:** Signaling server facilitates connection only; never stores media.

**Server Responsibilities:**
- Exchange encrypted SDP (Session Description Protocol) offers/answers.
- Store ICE candidates (non-sensitive routing info).
- Maintain connection logs and viewer lists for authority checks.
- **NO storage of:** media frames, audio samples, recordings, or stream metadata.

**How Media Flows:**
1. Parent/Baby exchange SDP via signaling server (STOMP/WebSocket).
2. WebRTC connection established with DTLS-SRTP encryption.
3. All media flows **directly P2P** (never touches signaling server).
4. Session data cleared after 30-second timeout.

**User Privacy:**
- Trust model: Signaling server is infrastructure you control.
- For absolute paranoia: Users can self-host signaling server on their own infrastructure.

---

### 10. Local Network Security

**Decision:** Keep it simple with QR code-based discovery.

- **Local Network Transport:** UDP or TCP connections unencrypted (trusted home network).
- **Rationale:** Home WiFi is assumed trusted; adding encryption adds complexity without proportional security gain.
- **Fallback Security:** If someone gains WiFi access, they still need the QR code (5-min token) to pair.

---

## Battery & Power Management

### 11. Battery Drain & Optimization

**Status:** Testing postponed until app development complete.

- Plan to measure battery drain per hour in each state:
  - Active (Live): Expected high drain.
  - Sentry: Medium drain (motion/cry detection running).
  - Sleep: Minimal drain (awaiting FCM wake ping).
- Optimization strategies to explore:
  - Hardware-accelerated video encoding.
  - Efficient motion detection on low-resolution stream.
  - CPU throttling when possible.

---

### 12. Sleep Mode Wake-up Mechanism

**Decision:** Use **Firebase Cloud Messaging (FCM)** for persistent wake-up.

- **Sleep State:** Camera process killed, minimal power draw.
- **Wake Signal:** Parent's request triggers FCM notification to Baby Unit.
- **Wake Latency:** Depends on FCM delivery (typically <5 seconds).
- **Fallback:** Persistent WebSocket listener (higher battery drain) for GitHub APK users without FCM.

---

### 13. Adaptive Bitrate Strategy

**Decision:** Scale bitrate (up to 5Mbps cap) based on network conditions.

- **Default:** 1080p (highest hardware-supported resolution).
- **Manual Quality Control:** Parent can force 1080p, 720p, 480p, or Auto mode via YouTube-style gear icon.
- **Poor Network (3G/Low Signal):**
  - Bitrate scales down aggressively (500Kbps - 2Mbps range).
  - Video quality degrades but remains usable.
  - Audio prioritized over video for two-way communication.
- **WebRTC Stats:** Monitor bitrate via `getStats()` for real-time adaptation.
- **Stealth Mode (OLED):** True-black overlay widget (#000000) shuts off pixels while keeping camera/sensors operational.

---

## Sensors & Detection

### 14. Motion Detection Algorithm

**Status:** Implementation details TBD until app development.

- **Approach:** Frame-comparison on low-resolution hidden stream.
- **Resolution:** TBD (likely 240p or 360p to balance accuracy vs CPU overhead).
- **Metrics to Measure:**
  - CPU usage during continuous motion detection.
  - Battery drain per hour.
  - False positive rate (lighting changes, shadows).
- **Threshold:** Configurable sensitivity (low/medium/high) based on testing.

---

### 15. Cry Detection Algorithm

**Decision:** Frequency-based analysis targeting 2kHz - 4kHz resonance.

- **Approach:** FFT (Fast Fourier Transform) on audio stream.
- **Target Band:** 2kHz - 4kHz (typical infant distress frequencies).
- **False Positives Expected:** Yes (babbling, coughing, other sounds).
- **Parent Behavior:** Alert every time (even false positives) so parent checks feed immediately.
- **Rationale:** Better to alert on false positive than miss real cry. Parent can quickly see it's not urgent.
- **No Suppression:** Parents cannot disable cry detection; all alerts are sent.

---

### 16. Motion & Cry Alert Cooldown

**Decision:** 60-second cooldown (non-configurable by parent).

- **Cooldown Logic:** 
  - First motion/cry detected → Alert sent immediately.
  - Next 60 seconds: No new alerts (cooldown).
  - After 60 seconds: Next motion/cry triggers new alert.
- **Rationale:** Prevents notification fatigue while ensuring frequent check-ins on baby.
- **Parent Workflow:** 
  - Baby cries → Alert.
  - Parent checks feed and comforts baby.
  - Baby cries again after 2 minutes → New alert (parent already alerted once).

---

### 17. Sensor Overhead & CPU Impact

**Status:** TBD until app development.

- Metrics to benchmark:
  - Motion detection CPU usage (% per thread).
  - Cry detection CPU usage (FFT computation cost).
  - Combined overhead with video encoding active.
  - Battery drain impact (% per hour).

---

## Networking & Connectivity

### 18. NAT Traversal & STUN/TURN Strategy

**Decision:** Use Google Public STUN servers; plan fallbacks.

**STUN Configuration:**
- Primary: Google Public STUN (`stun.l.google.com:19302`).
- Fallback: 
  - `stun1.l.google.com:19302`
  - `stun2.l.google.com:19302`
  - Open-source STUN servers (if available).

**TURN Relay (Future Consideration):**
- If STUN fails and P2P direct connection impossible:
  - Option A: No relay → Media not transmitted (acceptable for home use).
  - Option B: Deploy low-cost TURN server (e.g., coturn on Oracle Always Free).
  - Decision: Defer TURN implementation until user feedback indicates need.

**No Manual Port Forwarding:** Users don't need to configure router (complexity too high for target audience).

---

### 19. ICE Auto-Restart on Network Switch

**Status:** TBD after app development.

- Behavior: When network switches (WiFi → 4G), ICE restarts to find new connection path.
- Metric to measure: Time until audio/video resumes (target: <5 seconds).
- Implementation: WebRTC handles ICE auto-restart automatically; ensure it's enabled.

---

### 20. Heartbeat & Ghost Timeout

**Decision:** 30-second "Ghost" timeout (non-configurable, revisit if needed).

- **Heartbeat Protocol:** 
  - Parent sends keep-alive every 15 seconds (over data channel or WebSocket).
  - Baby Unit expects heartbeat.
- **Timeout:** If no heartbeat for 30 seconds, Baby Unit assumes parent disconnected (without graceful close).
- **Cleanup:** Baby Unit clears viewer slot after timeout.
- **Revisit:** If testing shows 30s too aggressive on high-latency networks, adjust during beta.

---

### 21. Local Network Discovery (QR Code Only)

**Decision:** Keep it simple; use only **QR code-based discovery (Option 3)**.

**Process:**
1. Baby Unit generates QR code with:
   - Local IP address (192.168.x.x).
   - Device ID.
   - 5-minute expiration token.
2. Parent scans QR code with parent phone.
3. Parent enters name (e.g., "Father", "Mother").
4. Baby Unit displays pending approval with parent's name.
5. Baby Unit user (parent holding phone) approves or rejects.
6. Connection established.

**Advantages:**
- Simple implementation (no UDP broadcast, no mDNS).
- Works cross-network if IP is public (not just local).
- Very secure (requires physical access or screenshot).
- No network discovery complexity.

---

## Notifications & Alerts

### 22. Firebase Cloud Messaging (FCM) Reliability

**Decision:** Implement fallback strategy for failed FCM delivery.

**FCM Limitations:**
- No guaranteed delivery (Google's design).
- Typical delivery rate: 95-99% (can fail silently).

**Fallback Strategy:**
1. **Primary:** Send alert via FCM.
2. **If App Open/Backgrounded:** Also send alert via WebRTC Data Channel (instant).
3. **If App Killed:** Rely on FCM (with poor reliability accepted).
4. **Retry Logic:** Signaling server retries FCM delivery 3 times over 5 minutes.
5. **User Expectation:** Document that app must stay open or enable foreground service for reliable notifications.

**Future Option:** Self-hosting TURN/relay server to enable persistent connection (replaces FCM dependency).

---

### 23. Low Battery Notification (15%)

**Decision:** Informational alert only (no auto-disconnect).

- **Trigger:** When Baby Unit battery drops to 15%.
- **Notification:** Sent to all connected parent viewers.
- **Action:** Informational only; parent decides if they need to charge.
- **Rationale:** Gives parent time to make decision; disconnecting automatically is frustrating.

---

### 24. Connection Lost Alarm

**Decision:** Alert parent if heartbeat fails for >30 seconds.

- **Trigger:** Baby Unit sends heartbeat every 15s; if none received for 30s, parent unit triggers alarm.
- **Alarm Behavior:** 
  - Notification + sound/vibration.
  - Parent can silence alarm.
  - Alarm stops once connection reestablished.
- **Use Case:** Parent detects baby unit lost power, network crashed, or device stolen.

---

### 25. Alert Volume & DND Compliance

**Decision:** Respect system notification volume and Do Not Disturb settings.

- **Volume:** Alerts use system notification channel (respects volume setting).
- **DND:** If system DND is active, alerts are silent (users can override per-app).
- **No Override:** Baby Monitor does NOT bypass DND by default.
- **Future Option:** Allow users to enable "bypass DND" for critical alerts (motion/cry).

---

## Data & Logging

### 26. Local Event History Retention

**Decision:** Start with **1-month retention** (configurable in future).

**Events to Log:**
- Motion detection (timestamp, confidence).
- Cry detection (timestamp, confidence).
- Connection/disconnection (timestamp, viewer name).
- Battery level changes.
- Network switches (WiFi ↔ 4G).

**Storage:**
- Local SQLite database on parent phone.
- AsyncStorage for simple event metadata.
- Retention: 1 month by default; oldest entries auto-deleted.
- User Control: Future version allows configurable retention (1 week, 1 month, 3 months, unlimited).

---

### 27. Baby Unit Logging (Issue-Only)

**Decision:** Keep issue-only logs on Baby Unit for debugging.

**What to Log:**
- Errors/exceptions (with stack trace).
- Connection failures (IP, reason).
- Sensor failures (camera, microphone).
- Battery/power anomalies.
- State machine violations (e.g., unexpected state transition).

**What NOT to Log:**
- Media frames or audio samples.
- Individual viewer names or connection counts (privacy).

**Storage:**
- Local file on Baby Unit (~10MB max).
- Older logs overwritten (circular buffer).
- User can export logs for debugging (future feature).

**Privacy:** Logs contain no PII or media; safe to send to developer for issue diagnosis.

---

### 28. Event History for Sleep Analytics

**Decision:** Comprehensive logging for sleep/activity mapping.

**Events to Track:**
- Motion detected (timestamp, intensity).
- Cry detected (timestamp, duration, frequency).
- Connection started (timestamp, parent name).
- Connection ended (timestamp, reason).
- Battery level (hourly snapshot).
- Network switches (WiFi ↔ 4G).

**Analytics (Future Feature):**
- Overlay motion/cry events on a timeline.
- Estimate baby sleep duration (low motion, no cry for X minutes).
- Generate daily/weekly reports (e.g., "Baby slept 12.5 hours yesterday").
- Heat map of active hours.

**Initial MVP:** Collect all events; analytics UI can be added later.

---

## Deployment & Operations

### 29. Signaling Server Reliability & SLA

**Current Plan:**
- Deploy to Oracle Cloud Always Free (ARM Ampere).
- No formal SLA (hobbyist/personal use project).
- No monitoring/alerting infrastructure initially.

**Future Considerations (if project scales):**
- Implement health checks (e.g., /health endpoint).
- Add basic error logging (to file or external service).
- Plan redundancy (backup signaling server on different infra).
- SLA target: 99% uptime (but no guarantees initially).

---

### 30. Self-Hosting Signaling Server

**Decision:** YES - Code will be available for self-hosting.

**User's Option:**
1. Use Viyu's official signaling server (free, always-free tier).
2. Or: Deploy signaling server code on personal infrastructure:
   - AWS, Azure, DigitalOcean, home server, etc.
   - Modify client app config to point to custom server.
   - Full control over data/privacy.

**Benefits:**
- Ultimate privacy (no data reaches Viyu infrastructure).
- Control over uptime & reliability.
- Educational value.

**Implementation:**
- Document deployment steps (Docker, systemd, etc.).
- Provide signaling server code (Spring Boot) with clear comments.
- Include configuration guide.

---

### 31. App Distribution & Updates

**Distribution Channels:**
1. **Google Play Store:** Official releases, auto-updates, verified builds.
2. **GitHub Releases:** APK for personal/open-source community.
3. **APK Direct Download:** From GitHub releases page.

**Update Strategy:**
- Play Store: Google handles updates (incremental, signed).
- GitHub APK: Manual download + reinstall (or in-app update checker TBD).
- Signaling Server: Manual deployment (user pulls new code, rebuilds, redeploys).

**Versioning:** Semantic versioning (v1.0.0, v1.1.0, etc.).

---

### 32. License Enforcement

**Decision:** No active enforcement planned initially.

**License Model:** Source Available (personal/educational use only).
- ✅ **Allowed:** Personal learning, private modification, experimentation.
- ❌ **Prohibited:** Redistribution as product, commercial use, App Store uploads.

**Enforcement:**
- Rely on community trust & legal terms.
- If violations reported, address case-by-case.
- Future: Consider license key system (if project becomes popular).

---

## User Experience & Workflows

### 33. Initial Setup Flow (First-Time Pairing)

**Complete Workflow:**

**Phase 1: Baby Unit Setup (1-2 minutes)**
1. Install app on old phone (Baby Unit).
2. Open app → Select "Baby Unit" role.
3. Grant camera/microphone permissions.
4. Choose camera (front or back).
5. App generates QR code (5-min expiration).
6. QR code displayed prominently on screen.

**Phase 2: Parent Unit Setup (1-2 minutes)**
1. Install app on parent's phone (Parent Unit).
2. Open app → Select "Parent Unit" role.
3. Grant camera/microphone permissions (for two-way audio).
4. Tap "Scan QR Code".
5. Camera opens; scan Baby Unit's QR.
6. Enter parent's name (e.g., "Father", "Mother").
7. Connection request sent to Baby Unit.

**Phase 3: Approval (Baby Unit)**
1. Baby Unit shows pending approval:
   - "Parent Unit is requesting access: **Father**"
   - Approve / Reject buttons.
2. User (holding Baby Unit) taps Approve.
3. Connection established.

**Total Setup Time:** 5-10 minutes (first time), <1 minute (additional parents).
**Failure Points to Address:**
- QR code expires → Regenerate new code.
- Camera permission denied → Clear instructions to enable.
- Network issues → Retry logic + clear error messages.

---

### 34. Multiple Baby Units (Future Plan)

**Current MVP:** Single Baby Unit per parent.

**Future Roadmap (Post-Launch):**
- Tab-based UI to switch between Baby Units.
- Stored pairing list (Baby Unit A, Baby Unit B, etc.).
- Quick-switch interface.

---

### 35. Multi-Parent Scenario

**Decision:** Support multi-parent monitoring with parent-specific permissions.

**Permissions:**
- **Baby Unit (Master):** Can kick ANY parent/viewer instantly.
- **Parent Units:** Can view feed and communicate with baby, but:
  - Cannot kick other parents.
  - Can unpair themselves.
- **Conflict Resolution:** If two parents disagree about disconnecting:
  - Either parent requests disconnection → baby unit decides.
  - No parent-to-parent kicking (enforced).

**UI on Baby Unit:**
- Shows list of connected viewers (name + connection time).
- Approve/Reject buttons for each parent requesting access.
- One-tap "Kick" to remove any viewer.

**UI on Parent Unit:**
- Shows who else is currently watching ("Father & Mother viewing").
- Shows viewer count in header.

---

### 36. Offline Monitoring on Home Network

**Decision:** Use local network when both devices on same WiFi (bandwidth priority).

**Workflow:**
1. Parent & Baby on same WiFi → Auto-use local network (direct connection).
2. Parent leaves home (4G only) → Auto-switch to WebRTC via signaling server.
3. Parent returns home (WiFi) → Auto-switch back to local network.
4. Internet down at home (WiFi + 4G both fail):
   - If both on WiFi: Still connected locally, full monitoring.
   - If parent on 4G: No connection (no fallback without internet).

**Status:** Auto-switching logic TBD during development.

---

## Edge Cases & Error Handling

### 37. Connection Loss Recovery

**Scenario:** Parent and Baby Unit lose connection mid-stream.

**Recovery Process:**
1. Both devices detect disconnect (no heartbeat).
2. Baby Unit clears parent slot after 30-second timeout.
3. Parent sees "Connection Lost" notification.
4. Parent taps "Reconnect":
   - Baby Unit checks if reconnection is allowed (within session window? TBD).
   - If allowed: Attempt new P2P connection; if fails, use signaling server.
   - If denied: Show "Session expired" → require new QR code.
5. Re-establishment: Clean state, fresh audio/video stream.

**State Handling:** No resume mid-stream; treat as new connection.

---

### 38. Role Switching (Unit from Baby to Parent)

**Scenario:** Old phone used as Baby Unit now needed as Parent Unit.

**Process:**
1. On old phone, unpair everything (Baby Unit mode).
2. Clear all app data / reinstall app.
3. Open app → Select "Parent Unit" role.
4. Start fresh pairing with another Baby Unit.

**Automatic Reset:** Unpairing clears all state; no manual reset needed.

---

### 39. Concurrent Viewer Limit (3rd Viewer Rejected)

**Scenario:** 2 viewers connected; 3rd tries to join.

**Behavior:**
1. 3rd parent scans QR → Sends connection request.
2. Baby Unit receives request.
3. Baby Unit **rejects connection**:
   - Send notification: "Maximum 2 viewers reached."
   - Option: "View who's watching" → Show current viewers.
4. 3rd parent sees error: "Cannot connect: Max viewers (2) reached. Current viewers: Father, Mother."
5. 3rd parent must ask one of them to disconnect.

---

### 40. Extreme Bandwidth Degradation

**Scenario:** Network drops to <500Kbps.

**Expected Behavior:**
1. WebRTC detects low bitrate via `getStats()`.
2. Bitrate scales down aggressively (target: 256-500Kbps).
3. Video quality degrades:
   - Resolution drops to 240p or 360p.
   - Framerate reduces to 10-15 fps.
   - Pixelation/artifacts visible.
4. Audio remains priority (full quality preserved).
5. Two-way communication still works (parent can talk to baby).

**If <256Kbps:**
- Video may stop altogether.
- Audio-only mode active.
- Parent sees: "Low bandwidth: Video paused. Audio only."
- Parent can manually retry video or accept audio-only.

**No Auto-Disconnect:** Connection persists; user decides to disconnect.

---

## Future Roadmap

### 41. iOS Support

**Plan:** After Android MVP is stable and tested.

**Approach:**
- Reuse React Native codebase.
- Port native modules (camera, microphone) to iOS equivalents.
- Adapt UI for iPhone/iPad form factors.
- Test on various iOS versions (target iOS 13+).

**Timeline:** 3-6 months post-Android launch.

---

### 42. Cloud Storage (Encrypted Optional Backup)

**Current Decision:** No cloud storage in MVP.

**Future Consideration (Post-Launch):**
- Allow opt-in encrypted cloud backup for event history.
- User controls:
  - Enable/disable backup.
  - Encryption key stored locally (user never shares key).
  - Backup frequency (daily, weekly, monthly).
- Use case: Backup history if phone lost/reset.
- Privacy: Viyu infrastructure stores encrypted blob only (cannot read).

**Timeline:** Post-MVP (maybe v2.0 or later).

---

### 43. Multiple Cameras / Rooms

**Current Decision:** Single camera per Baby Unit (MVP).

**Future Plan:**
- Support multiple cameras on same Baby Unit (front, back, external USB).
- Tab-based camera switcher.
- Use case: Monitor baby room + nursery.

**Timeline:** v1.5 or later.

---

### 44. Sleep & Activity Analytics

**Current Decision:** Collect event data now; UI analytics later.

**Future Analytics Dashboard:**
- Sleep duration estimates (low motion + no cry = sleep).
- Activity heatmap (when is baby most active?).
- Cry frequency chart (cries per day, by time).
- Daily/weekly reports.
- Notifications if patterns change (e.g., unusual activity).

**Implementation:** Add analytics screen post-MVP when data accumulation is complete.

---

## Implementation Priorities

### Phase 1 (MVP)
- [x] QR code generation & expiration (5 min).
- [x] Parent approval workflow.
- [x] WebRTC P2P connection (DTLS-SRTP).
- [x] Signaling server (STOMP + Java Spring Boot).
- [x] Three-state power management (Active/Sentry/Sleep).
- [x] Motion & cry detection (frame-comparison + FFT).
- [x] Local event history (SQLite, 1-month retention).
- [x] FCM notifications + fallback.
- [x] Issue-only logging on Baby Unit.

### Phase 2 (Post-MVP)
- [ ] Android app on Google Play Store.
- [ ] Multi-parent approval UI refinement.
- [ ] Battery drain benchmarking & optimization.
- [ ] TURN relay server (if STUN insufficient).
- [ ] Analytics dashboard.

### Phase 3 (Future)
- [ ] iOS support.
- [ ] Cloud backup option.
- [ ] Multiple cameras per Baby Unit.
- [ ] Advanced sleep analytics.

---

## Decisions Summary Table

| Question | Decision | Rationale |
|----------|----------|-----------|
| P2P Failover | Auto-switch, optimal latency TBD | Maximize efficiency & speed |
| Viewer Limit | 2 simultaneous (design choice) | Battery on old devices |
| Master Authority | Baby Unit full control | Prevent misuse |
| Min SDK | TBD during development | Target Android 8+ likely |
| Notifications | FCM preferred, fallback allowed | Flexibility + privacy |
| State Management | Redux-like state machine | Robust sync & recovery |
| QR Expiration | 5 minutes | Prevent old code reuse |
| Encryption | DTLS-SRTP (WebRTC standard) | Industry best practice |
| Signaling Trust | Signaling ≠ Storage | Media flows P2P only |
| Local Discovery | QR code only | Simplicity + security |
| Wake-up | FCM for Sleep mode | Battery efficient |
| Bitrate | Up to 5Mbps, adaptive | Performance on old devices |
| Motion Alert | Every detection (no suppression) | Frequent parent check-ins |
| Cry Alert | Every detection (no suppression) | Parental awareness |
| Cooldown | 60s (non-configurable) | Prevent fatigue, ensure awareness |
| Heartbeat | 30s timeout (revisit if needed) | Reasonable ghost timeout |
| Battery Alert | 15% informational | Parent decides action |
| History Retention | 1 month (configurable future) | Balance storage & history |
| Baby Logging | Issue-only + logs | Debug without privacy loss |
| Signaling SLA | No formal SLA (MVP) | Personal project initially |
| Self-Host | YES - code provided | User privacy option |
| Distribution | Play Store + GitHub APK | Official + community |
| License | Source Available (trust-based) | Non-commercial enforced socially |
| Multi-Baby | Planned post-MVP | Scope management |
| Multi-Camera | Planned post-MVP | Scope management |
| Cloud Storage | No MVP; encrypted option future | Privacy-first approach |

---

## Next Steps

1. **Finalize Android minimum SDK version** (recommend API 26+ for broad compatibility).
2. **Design React Native components** for Parent/Baby role selection & QR flow.
3. **Set up Spring Boot signaling server** scaffold (STOMP/WebSocket handling).
4. **Implement WebRTC P2P** connection with DTLS-SRTP.
5. **Build state machine** for Active/Sentry/Sleep management.
6. **Integrate FCM** for notifications.
7. **Develop motion & cry detection** algorithms (with benchmarking).
8. **Create SQLite schema** for event history.
9. **Test on target hardware** (Poco X4 Pro 5G + older test devices).
10. **Document deployment steps** for signaling server self-hosting.

---

**Document Version:** v2.0
**Last Updated:** 2026-05-12
**Created By:** @himanshuarora111
