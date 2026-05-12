# View Baby Monitor (Viyu) - Plan v3 📋

**Last Updated:** 2026-05-12  
**Based on:** Answers from core team discussion + implementation suggestions

---

## 📑 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Server Responsibility (Minimal)](#server-responsibility-minimal)
3. [Baby Unit (Master Logic)](#baby-unit-master-logic)
4. [Parent Unit (Viewer)](#parent-unit-viewer)
5. [Security & QR Code Flow](#security--qr-code-flow)
6. [Notifications & Fallback Strategy](#notifications--fallback-strategy)
7. [State Management & Persistence](#state-management--persistence)
8. [Sensors & Detection](#sensors--detection)
9. [Data Storage](#data-storage)
10. [Deployment & Tech Stack](#deployment--tech-stack)
11. [Implementation Roadmap](#implementation-roadmap)

---

## Architecture Overview

### Core Principle: **Baby Unit is the Source of Truth**

The signaling server is **ONLY** responsible for:
- Exchanging WebRTC connection metadata (SDP offers/answers, ICE candidates)
- Facilitating initial peer discovery
- **NOT** storing media, state, or user data

All logic, state, and decision-making resides on the **Baby Unit**.

```
┌─────────────────┐         ┌──────────────────┐         ┌──────────────────┐
│  Parent Unit    │◄───────►│ Signaling Server │◄───────►│   Baby Unit      │
│  (Viewer)       │         │ (STOMP Broker)   │         │  (Host/Master)   │
└─────────────────┘         └──────────────────┘         └──────────────────┘
       │                                                           │
       │                    P2P WebRTC                             │
       └───────────────────────────────────────────────────────────┘
              (Video/Audio/Data Channel - Direct)
```

---

## Server Responsibility (Minimal)

### Spring Boot STOMP Server Functions

**ONLY these operations:**

1. **Room Registration:**
   - Baby Unit registers room on startup (RoomID + JWT token).
   - Server stores: `{RoomID → Baby Unit connection handle}` in memory.
   - TTL: Auto-cleanup after 5 minutes of inactivity.

2. **SDP Exchange:**
   - Parent sends SDP offer → Server forwards to Baby Unit.
   - Baby Unit sends SDP answer → Server forwards to Parent.
   - **ICE Candidates:** Exchanged through server (STOMP message relay).

3. **Connection Capacity Check:**
   - Parent requests connection → Server asks Baby Unit: "Do you have capacity?"
   - Baby Unit responds YES/NO.
   - If NO: Server relays error to Parent ("Max 2 viewers").

4. **FCM Fallback Trigger:**
   - If parent app is killed and unreachable via WebRTC Data Channel:
     - Baby Unit notifies signaling server: "Send FCM to parent X."
     - Server retrieves Parent's FCM token and triggers Firebase.

5. **Heartbeat Relay (Optional Optimization):**
   - If both devices prefer not to do heartbeat directly:
     - Parent sends heartbeat via server → Server forwards to Baby Unit.
   - **OR** Parent sends heartbeat directly via Data Channel (preferred).

---

### STOMP Topic Structure

```
/topic/baby-{roomId}          ← Parent listens for updates from Baby Unit
/topic/parent-{roomId}        ← Baby Unit listens for requests from Parent
/app/register                 ← Baby Unit registers room
/app/heartbeat                ← Heartbeat relay (optional)
/app/fcm-trigger              ← Baby Unit requests FCM notification
```

**Example Flow:**
1. Baby Unit: `SEND /app/register {RoomID, JWT, FCMtoken}`
2. Server: `SUBSCRIBE /topic/baby-{roomId}` on behalf of Baby Unit
3. Parent: `SEND /app/request-connect {RoomID, JWT}`
4. Server: `SEND /topic/baby-{roomId} {parent-request}`
5. Baby Unit: Evaluates capacity, sends response
6. Server: `SEND /topic/parent-{roomId} {answer: accept/reject}`

---

## Baby Unit (Master Logic)

### Responsibilities

1. **QR Code Generation:**
   - Generate JWT token (5-min expiration).
   - Embed: `{Baby Unit ID, Local IP, JWT, Firebase Token}` in QR.
   - Display on screen.
   - No server involvement.

2. **Viewer Management:**
   - Maintain in-memory list of connected viewers (max 2).
   - Approve/Reject connection requests from parents.
   - Display active viewer count + names.
   - "Kick" button to disconnect any viewer instantly.

3. **State Machine (Active/Sentry/Sleep):**
   - Manage video/audio encoding based on state.
   - Persist state to disk (resume on crash within 30s TTL).

4. **Sensor Logic:**
   - Motion detection (pixel-comparison on 240p hidden stream).
   - Cry detection (FFT on 2kHz-4kHz band).
   - **Only active when parent is connected** (not in Sleep mode).

5. **Data Storage:**
   - All event history (motion, cry, connections) stored locally in SQLite.
   - Summarized events (no raw frame data).
   - Keep ~5MB of data (approximately 1 month).

6. **FCM Token Management:**
   - Store own FCM token locally.
   - Share with server only when registering the room.
   - On token refresh → Update locally and re-register with server.

7. **Heartbeat & Ghost Timeout:**
   - Expect heartbeat from each connected parent every 5-15 seconds.
   - If no heartbeat for >30s: Assume parent disconnected, free slot.
   - Enforced on Baby Unit (not server).

---

## Parent Unit (Viewer)

### Responsibilities

1. **QR Code Scanning:**
   - Scan QR from Baby Unit.
   - Extract: `{Baby ID, Local IP, JWT, Baby FCM Token}`.
   - Establish P2P connection via signaling server.

2. **Dual-Path Connectivity:**
   - **Path A (Preferred):** Connect via local IP on same WiFi (direct UDP).
   - **Path B (Fallback):** Use signaling server + STUN/TURN if not on same WiFi.

3. **WebRTC Connection:**
   - Send SDP offer to signaling server.
   - Receive SDP answer from Baby Unit.
   - Exchange ICE candidates.

4. **State Management:**
   - Track connection status (Active/Sentry/Sleep).
   - Resume state if reconnected within 30 seconds.
   - Drop state if >30 seconds (start fresh session).

5. **Local Event History:**
   - Receive summarized event data from Baby Unit.
   - Store in local SQLite database.
   - Display in "History" tab.

6. **Quality Control:**
   - YouTube-style quality selector (Auto, 1080p, 720p, 480p).
   - Manual override or auto-adaptive bitrate.

---

## Security & QR Code Flow

### QR Code Generation & Expiration

**On Baby Unit:**

1. Generate JWT token:
   ```
   {
     "baby_id": "unique-device-id",
     "iat": current_timestamp,
     "exp": current_timestamp + 300,  // 5 minutes
     "local_ip": "192.168.1.100",
     "port": 5000
   }
   ```

2. Encode into QR:
   ```
   Baby Unit ID | Local IP | JWT | FCM Token
   ```

3. Display on screen (refresh every 5 minutes or on-demand).

**On Parent Unit:**

1. Scan QR.
2. Parse JWT (verify signature locally or via signaling server).
3. Extract Baby Unit ID + Local IP + JWT.
4. Attempt local connection first; if fails, use signaling server.

**JWT Validation:**
- Parent validates JWT signature using public key stored locally (or fetches from Baby Unit).
- If invalid: Show error "Invalid QR Code."
- If expired: Show error "QR Code expired. Scan a new one."

---

### Initial Connection Workflow

```
1. Parent scans QR → Gets Baby ID, Local IP, JWT
2. Parent enters name (e.g., "Dad", "Mom")
3. Parent sends connection request:
   {
     "parent_id": "...",
     "parent_name": "Dad",
     "jwt": "...",
     "baby_id": "..."
   }
4. Signaling server relays to Baby Unit
5. Baby Unit displays approval dialog:
   "Parent Unit requesting access: **Dad** [APPROVE] [REJECT]"
6. User approves → Baby Unit sends acceptance
7. Server relays acceptance to Parent
8. P2P WebRTC connection established
```

---

## Notifications & Fallback Strategy

### Hybrid Notification Engine

**Decision:** Firebase first, Data Channel fallback.

---

### Scenario 1: Parent App is OPEN (Foreground or Backgrounded)

1. Baby Unit detects motion/cry.
2. **Attempt 1:** Send alert via WebRTC Data Channel.
3. Latency: <100ms (instant).
4. If Data Channel fails (network issue):
   - Fall back to Firebase FCM.

---

### Scenario 2: Parent App is KILLED by OS

1. Baby Unit detects motion/cry.
2. **Attempt 1:** Send via Data Channel → Fails (app is killed).
3. **Attempt 2:** Send FCM request to signaling server:
   ```
   POST /fcm-trigger
   {
     "parent_id": "...",
     "alert_type": "motion|cry",
     "timestamp": "..."
   }
   ```
4. Signaling server looks up Parent's FCM token.
5. Sends Firebase Cloud Message.
6. Firebase delivers to Parent's device.
7. Parent receives notification (even if app is killed).
8. Parent can tap notification → App re-launches.

---

### Scenario 3: Both WiFi and 4G Down (Offline)

**Suggestion:** Show "Offline" message to parent. Here's what I recommend:

1. **Baby Unit Perspective:**
   - No connectivity to signaling server.
   - If parent is connected locally (same WiFi), continue streaming.
   - If parent is not local, streaming stops.

2. **Parent Unit Perspective:**
   - Show "Baby Unit Offline" message.
   - Offer option to "Retry" every 5 seconds.
   - Store any sensor events locally and sync when reconnected.

3. **No Reconnect Logic (Keep It Simple):**
   - Don't auto-reconnect; make user explicitly retry.
   - This prevents accidental battery drain on endless reconnect loops.
   - Future improvement: Exponential backoff strategy.

---

### Scenario 4: Network Switch (WiFi ↔ 4G)

**Suggestion:** Use ICE auto-restart + Network change detection.

1. Parent's device switches from WiFi to 4G.
2. Android OS notifies app of network change.
3. App triggers ICE restart:
   ```javascript
   peerConnection.restartIce();
   ```
4. WebRTC automatically:
   - Gathers new ICE candidates.
   - Tests new connection path (via STUN).
   - Switches seamlessly.
5. Latency: 1-3 seconds (imperceptible).

**Implementation:**
- Listen to `ConnectivityManager` on Android.
- On network change, call `peerConnection.restartIce()`.
- Show brief "Reconnecting..." spinner.

---

## State Management & Persistence

### Multi-State Power Engine

| State | Video | Audio | Data Channel | Sensors | CPU/Battery | When Used |
|-------|-------|-------|--------------|---------|-------------|-----------|
| **Active (Live)** | ON (1080p) | ON | ON | Running | High | Parent actively watching |
| **Sentry (Bg)** | OFF | User Toggle | ON | **Running** | Medium | Parent app minimized |
| **Sleep (Idle)** | OFF | OFF | Listener | OFF | Minimal | Monitor deactivated |

---

### State Persistence & Recovery

**On Baby Unit:**

1. Store current state to disk (SQLite or file):
   ```sql
   CREATE TABLE state_cache (
     key TEXT PRIMARY KEY,
     value TEXT,
     timestamp INTEGER
   );
   ```

2. On app crash/restart:
   ```
   IF (time_since_last_state < 30 seconds)
     Resume state (Active/Sentry/Sleep)
   ELSE
     Default to Sleep state
   ```

**Parent Reconnection:**

1. Parent disconnects.
2. If reconnection within 30 seconds:
   - Resume with same state (don't drop video).
   - Show "Reconnected" message.
3. If reconnection after 30 seconds:
   - Treat as new session.
   - Start from Sleep state.
   - Parent must re-activate.

---

### Heartbeat & Ghost Timeout

**Implementation on Baby Unit:**

```pseudocode
FOR EACH connected_parent:
  last_heartbeat = current_time
  
  EVERY 5 seconds:
    IF (current_time - last_heartbeat > 30 seconds):
      DISCONNECT parent (free slot)
      LOG "Parent ghosted, slot freed"

WHEN parent sends keep-alive:
  last_heartbeat = current_time  // Reset timer
```

**Transport:**
- Heartbeat can be via Data Channel (preferred) OR signaling server relay.
- Simple PING/PONG message (1 byte).

---

## Sensors & Detection

### Motion Detection

**Activation:** Only when parent is connected (not in Sleep).

**Algorithm:**
1. Capture camera frame at 240p resolution (hidden, not displayed).
2. Convert to grayscale.
3. Compare with previous frame (pixel-by-pixel difference).
4. If >X% of pixels changed: Motion detected.
5. Send summarized event: `{timestamp, confidence: low|medium|high}`.

**CPU Impact:** ~10% per thread (estimated; benchmark needed).

---

### Cry Detection

**Activation:** Only when parent is connected (not in Sleep).

**Algorithm:**
1. Capture audio stream at 16kHz sample rate.
2. Apply FFT on 1024-sample window (~64ms).
3. Extract frequency bins in 2kHz - 4kHz range.
4. If energy in this band exceeds threshold: Cry detected.
5. Send summarized event: `{timestamp, frequency_peak: Hz, confidence}`.

**CPU Impact:** ~15% per thread (estimated; benchmark needed).

---

### Alert Deduplication

**60-Second Cooldown (Non-configurable):**

```pseudocode
WHEN motion or cry detected:
  IF (time_since_last_alert < 60 seconds):
    SKIP alert (still in cooldown)
  ELSE:
    SEND alert to parent
    last_alert_timestamp = current_time
```

**Behavior:**
- First motion/cry → Alert sent immediately.
- Next 60 seconds: Additional detections ignored.
- After 60 seconds: Next detection triggers new alert.

**Example:**
```
00:00:00 - Cry detected → Alert sent
00:00:30 - Another cry → Skipped (cooldown active)
00:00:50 - Motion detected → Skipped (cooldown active)
00:01:05 - Another cry → Alert sent (cooldown expired)
```

---

## Data Storage

### SQLite Schema (Baby Unit + Parent Unit)

**Table 1: Events (Summarized)**

```sql
CREATE TABLE events (
  id INTEGER PRIMARY KEY,
  event_type TEXT,          -- 'motion', 'cry', 'connection'
  timestamp INTEGER,         -- Unix timestamp
  details TEXT,              -- JSON: {confidence, frequency, etc}
  created_at INTEGER
);

CREATE INDEX idx_timestamp ON events(timestamp DESC);
```

**Table 2: Connections (Baby Unit)**

```sql
CREATE TABLE connections (
  id INTEGER PRIMARY KEY,
  parent_id TEXT,
  parent_name TEXT,
  connected_at INTEGER,
  disconnected_at INTEGER,
  created_at INTEGER
);
```

**Table 3: State Cache (Baby Unit)**

```sql
CREATE TABLE state_cache (
  key TEXT PRIMARY KEY,
  value TEXT,
  updated_at INTEGER
);
```

**Storage Limit:** ~5MB per database.
- Estimated capacity: 1 month of hourly events (~720 events).
- Auto-cleanup: Delete events older than 30 days.

---

## Deployment & Tech Stack

### Backend: Spring Boot + Maven

**Project Structure:**
```
signaling-server/
├── pom.xml
├── src/
│   ├── main/java/com/viyu/
│   │   ├── SignalingServerApplication.java
│   │   ├── config/
│   │   │   ├── WebSocketConfig.java
│   │   │   └── StompConfig.java
│   │   ├── controller/
│   │   │   └── SignalingController.java
│   │   ├── service/
│   │   │   ├── RoomService.java
│   │   │   └── FCMService.java
│   │   ├── model/
│   │   │   ├── Room.java
│   │   │   ├── Viewer.java
│   │   │   └── SignalingMessage.java
│   │   └── util/
│   │       ├── JWTValidator.java
│   │       └── Logger.java
│   └── resources/
│       └── application.properties
└── Dockerfile
```

**Key Dependencies:**
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
<dependency>
  <groupId>com.google.firebase</groupId>
  <artifactId>firebase-admin</artifactId>
</dependency>
```

---

### Frontend: React Native (Android)

**Project Structure:**
```
baby-monitor-app/
├── package.json
├── app.json
├── android/
├── ios/
├── src/
│   ├── components/
│   │   ├── QRCodeScreen.js
│   │   ├── VideoFeed.js
│   │   ├── HistoryTab.js
│   │   └── ViewerManagement.js
│   ├── services/
│   │   ├── WebRTCService.js
│   │   ├── SignalingService.js
│   │   ├── SensorService.js
│   │   └── DatabaseService.js
│   ├── state/
│   │   └── BabyUnitState.js  (Redux or Zustand)
│   ├── utils/
│   │   ├── JWTUtil.js
│   │   ├── SQLiteUtil.js
│   │   └── Logger.js
│   └── App.js
└── README.md
```

**Key Dependencies:**
```json
{
  "react-native-webrtc": "^1.96.0",
  "react-native-sqlite-storage": "^6.0.0",
  "@react-native-firebase/messaging": "^18.0.0",
  "zustand": "^4.3.0",
  "axios": "^1.4.0"
}
```

**Build & Release:**
- Development: React Native CLI on Android emulator.
- Release: Android Studio build → APK/AAB for Play Store & GitHub.
- Version: Read from `package.json` → `versionCode` + `versionName`.

---

### Logging Strategy

**File-Based Error Logs Only:**

**Baby Unit:**
```
/data/data/com.viyu.monitor/logs/errors.log
├── [2026-05-12 10:30:45] ERROR: WebRTC connection failed - STUN timeout
├── [2026-05-12 10:35:12] ERROR: Motion detection FFT computation timeout
└── [2026-05-12 11:00:00] ERROR: SQLite disk full - cleanup triggered
```

**Format:**
```
[TIMESTAMP] LEVEL: MESSAGE
```

**Max Log Size:** 2-5MB (auto-rotate old logs).

**Content:**
- ❌ No media/frame data.
- ❌ No user names or PII.
- ✅ Error messages + stack traces.
- ✅ Connection failures + reasons.
- ✅ Sensor failures.
- ✅ State machine violations.

---

## Implementation Roadmap

### Phase 1: Core Signaling Server (Sprint 1)

**Milestones:**
1. Spring Boot project scaffold with Maven.
2. WebSocket + STOMP configuration.
3. Room registration endpoint (`/app/register`).
4. SDP offer/answer relay (`/topic/baby-*`, `/topic/parent-*`).
5. JWT validation utility.
6. FCM integration (receive tokens, send notifications).
7. Error logging to file.
8. Docker configuration for Oracle Cloud deployment.

**Definition of Done:**
- Two Android devices can exchange SDP via signaling server.
- Server logs all errors to file.
- Server cleans up rooms after 5-min inactivity.

---

### Phase 2: Baby Unit (React Native) - Signaling (Sprint 2)

**Milestones:**
1. React Native project scaffold with Firebase setup.
2. QR code generation (JWT + 5-min expiration).
3. Baby Unit registration with signaling server.
4. Receive connection requests from parent.
5. Approval/Rejection UI.
6. FCM token management.
7. State persistence (SQLite).
8. Error logging to file.

**Definition of Done:**
- Baby Unit generates QR code.
- Parent can connect to Baby Unit via QR.
- Baby Unit can kick viewer.
- State persists across crashes.

---

### Phase 3: WebRTC P2P + Local Connection (Sprint 3)

**Milestones:**
1. React Native WebRTC setup (`react-native-webrtc`).
2. Local network detection (same WiFi check via IP subnet).
3. Direct P2P connection (local IP preferred).
4. Fallback to signaling server if local fails.
5. Audio + video streaming.
6. ICE auto-restart on network switch.

**Definition of Done:**
- Parent and Baby Unit can stream video/audio locally.
- Fallback to server if local fails.
- Network switch triggers auto-reconnect.

---

### Phase 4: Sensor Logic + State Machine (Sprint 4)

**Milestones:**
1. Motion detection (240p pixel-comparison).
2. Cry detection (FFT on 2kHz-4kHz band).
3. Active/Sentry/Sleep state machine.
4. Heartbeat + 30-second ghost timeout.
5. 60-second alert cooldown.
6. Store summarized events in SQLite.

**Definition of Done:**
- Sensors detect motion and cry.
- Alerts sent with proper cooldown.
- State transitions work correctly.

---

### Phase 5: UI & Quality Control (Sprint 5)

**Milestones:**
1. Live video feed display.
2. YouTube-style quality selector.
3. Full-duplex audio (two-way).
4. Stealth mode (`#000000` overlay, blocks input).
5. History tab (display events from SQLite).
6. Viewer management UI.

**Definition of Done:**
- Parent can watch live feed with quality control.
- Two-way audio works.
- History tab displays past events.

---

### Phase 6: Testing + Optimization (Sprint 6)

**Milestones:**
1. Battery drain benchmarking (all states).
2. CPU usage profiling (sensors).
3. Network failover testing.
4. Crash recovery testing (state resume).
5. Stress test (2 simultaneous viewers).

**Definition of Done:**
- MVP ready for beta testing.
- Known issues documented.

---

## Key Design Decisions Summary

| Decision | Rationale |
|----------|-----------|
| **All logic on Baby Unit** | Reduce server load; easier self-hosting. |
| **Server = Signaling Only** | Simplicity; privacy; easy to replace. |
| **JWT + QR Code** | Secure, ephemeral, no database needed. |
| **Firebase First** | Reliable; handles app-killed scenario. |
| **Heartbeat on Baby Unit** | Enforces capacity limits locally. |
| **30-Second Resume TTL** | Balance between state preservation + fresh start. |
| **SQLite Only** | Simpler than dual AsyncStorage + SQLite. |
| **Summarized Events** | Save storage; privacy (no raw frame data). |
| **No Encryption at Rest** | Trust Android sandbox; reduces complexity. |
| **File-Based Logs** | Simple; no external service dependency. |
| **Local Network Priority** | Zero-latency, zero-data when at home. |
| **ICE Auto-Restart** | Seamless WiFi ↔ 4G switching. |

---

## Next Steps

1. **Create Spring Boot project** with Maven + WebSocket config.
2. **Set up STOMP endpoints** for room registration + SDP relay.
3. **Implement JWT validation** (5-min expiration).
4. **Create React Native scaffold** with Firebase setup.
5. **Build QR code generation** (Baby Unit).
6. **Implement WebRTC P2P** with local network fallback.
7. **Add sensor logic** (motion + cry detection).
8. **Store events in SQLite** (summarized).
9. **Test end-to-end** on two Android devices.
10. **Deploy signaling server** to Oracle Cloud.

---

**Document Version:** v3.0  
**Last Updated:** 2026-05-12  
**Status:** Ready for Implementation

