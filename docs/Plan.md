# Master Build Specification: View Baby Monitor (Viyu)

## 1. Project Overview
* **Goal:** A zero-cost, zero-server-storage, Peer-to-Peer (P2P) baby monitor.
* **Core Tech:** React Native (Android), Java Spring Boot (Signaling), WebRTC (Media).
* **Naming:** "View Baby Monitor" (Internal/Developer ref: **Viyu**).

---

## 2. The "Zero-Dollar" Infrastructure (User Guide)
To build and run this for free, the AI must implement support for the following:

* **Signaling Server:** Java Spring Boot JAR running on **Oracle Cloud Always Free** (Ampere A1 Compute).
* **STUN Server:** Use Google’s Public STUN: `stun:stun.l.google.com:19302`.
* **Notifications:** Firebase Cloud Messaging (FCM) Free Tier.
* **Local Discovery:** UDP Broadcast for zero-data home monitoring.

---

## 3. Backend: Java Spring Boot Signaling Server
* **Protocol:** WebSockets with STOMP sub-protocol.
* **Logic:**
    * Stateless session management using `ConcurrentHashMap`.
    * RoomID based matchmaking.
    * **Payloads:** Exchange of WebRTC SDP Offer, SDP Answer, and ICE Candidates.
* **Security:** Simple Handshake Interceptor to validate `RoomID` and `SecretKey`.

---

## 4. Frontend: React Native (Android Only)

### Dual-Role Architecture

#### A. Baby Unit (Host)
* **Camera:** Single camera lock (Front/Back) chosen at setup.
* **Stealth Mode:** Full-screen `#000000` overlay to turn off OLED pixels.
* **Analysis:**
    * **Motion:** Pixel-comparison on a 144p hidden stream.
    * **Cry:** FFT-based frequency analysis (2kHz - 4kHz).
* **Authority:** Host-side manifest of active PeerIDs. Capacity for 2 concurrent viewers.

#### B. Parent Unit (Viewer)
* **Live Feed:** WebRTC Video/Audio (Full Duplex).
* **Quality Selector:** YouTube-style toggle (Auto, 1080p, 720p, 480p).
* **Local History:** SQLite/AsyncStorage for event logs received from the Baby Unit.

---

## 5. Critical Logic Workflows

### A. Power States
| State | Description |
| :--- | :--- |
| **Active** | Video + Audio transmission (HD). |
| **Sentry** | Parent app minimized. Video track STOPPED on Baby Unit (save battery). Audio remains user-toggled. |
| **Sleep** | Manually triggered. Camera process KILLED. Baby Unit listens via a low-power WebSocket/UDP port for a "Wake Up" command. |

### B. Adaptive Bitrate (ABR)
* Monitor `availableOutgoingBitrate` via WebRTC `getStats()`.
* Automatically adjust `scaleResolutionDownBy` and `maxBitrate` (Cap at 5Mbps).

### C. The "Ghost" Protocol
* Baby Unit expects a heartbeat every 5s from Parent Units.
* If heartbeat is missing for 30 seconds, the Baby Unit closes that peer connection to free a slot.

### D. Hybrid Notifications
* **App Active:** Send alerts via WebRTC Data Channel.
* **App Killed:** Baby Unit sends a POST request to the Signaling Server -> Signaling Server hits Firebase FCM -> Parent phone receives Push Notification.

---

## 6. Implementation Instructions for the AI
* **Phase 1:** Setup Java Spring Boot STOMP server with `/app` and `/topic` endpoints.
* **Phase 2:** Create React Native project. Implement `react-native-webrtc` and basic P2P connection between two devices on the same Wi-Fi.
* **Phase 3:** Build the "Stealth Mode" UI and the logic for stopping video when the app backgrounds.
* **Phase 4:** Implement the Java-based Motion and Audio frequency analysis modules.
* **Phase 5:** Integrate Firebase for background notifications.

---

## How to use this with an AI:
If you are using an AI like Cursor or Claude, start with this prompt:

> "I want to build an app called **View Baby Monitor**. I have a Master Specification Document ready. I want to start with the Backend (Java Spring Boot). Please analyze the 'Signaling Server' section of my spec and generate the initial project structure and the WebSocket configuration class."
