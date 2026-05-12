# View Baby Monitor (Viyu) 👶📱

**View Baby Monitor** is a high-performance, privacy-first, and strictly non-commercial monitoring solution designed for parents who refuse to compromise on data security. Named after the creator's son, **Viyu**, this project aims to provide a professional-grade experience using spare Android hardware without ever storing a single byte of media on a third-party server.

---

## 🌟 Core Philosophy
*   **Zero-Server Storage:** Media (Audio/Video) is transmitted strictly Peer-to-Peer (P2P). No "cloud recording."
*   **Free for Life:** Designed to run on "Always Free" infrastructure (Oracle Cloud, Firebase Free Tier, Google STUN).
*   **Absolute Privacy:** Data is encrypted end-to-end via WebRTC.
*   **Parental Authority:** The device physically located with the baby is the "Master Node" with the power to kick any viewer.

---

## 🛠 Technical Stack

### Frontend (Mobile App)
*   **Framework:** React Native (JavaScript/TypeScript)
*   **Rationale:** Leverages Java (Native Modules) and Angular/React experience for rapid UI development and robust hardware access.
*   **Platform:** Optimized for Android (specifically tested on Poco X4 Pro 5G with OLED).

### Backend (Signaling Server)
*   **Language:** Java Spring Boot
*   **Communication:** STOMP over WebSockets for structured, low-latency signaling.
*   **Infrastructure:** Deployed on Oracle Cloud Always Free (ARM Ampere).
*   **Discovery:** Google Public STUN servers for NAT traversal.

### Connectivity
*   **Primary:** Local Network (UDP Broadcast) for zero-latency, zero-data usage when at home.
*   **Secondary:** WebRTC via Signaling Server (Fallback for 4G/5G/Remote monitoring).
*   **Protocol:** Adaptive Bitrate WebRTC with ICE Auto-Restart on network switching.

---

## 📑 Detailed Functional Specifications

### 1. The Multi-State Power Engine
To maximize the battery life of the Baby Unit, the app operates in three distinct logical states:

| State | Video | Audio | Data Channel | Description |
| :--- | :--- | :--- | :--- | :--- |
| **Active (Live)** | **ON (HD)** | **ON** | **ON** | Parent is actively watching. Full resolution stream. |
| **Sentry (Bg)** | **OFF** | **User Toggle** | **ON** | Parent app is minimized. Video encoding stops to save CPU. |
| **Sleep (Idle)** | **OFF** | **OFF** | **Listener** | Monitor is "Deactivated." Camera process is killed. Waits for wake-up ping. |

### 2. Video & Audio Engineering
*   **Adaptive High-Definition:** Defaults to the highest hardware-supported resolution (1080p). Dynamically scales bitrate (up to 5Mbps cap) based on network conditions using WebRTC `getStats`.
*   **Manual Quality Control:** YouTube-style gear icon on Parent Unit to force 1080p, 720p, 480p, or Auto.
*   **Full Duplex Audio:** High-quality, two-way communication (Phone-call style) for comforting the baby.
*   **Stealth Mode (OLED):** A `#000000` true-black overlay widget that shuts off pixels on the baby’s phone while keeping the camera and sensors fully operational.

### 3. Intelligent Sentry Sensors
*   **Visual Motion Detection:** Runs a frame-comparison algorithm on a low-resolution hidden track to detect movement without overheating the device.
*   **Cry Detection:** Frequency-based analysis (FFT) targeting the 2kHz - 4kHz resonance typical of infant distress.
*   **Alert Logic:** Fires a High-Priority notification to Parent Units. Includes a **60-second cooldown** to prevent notification fatigue.

### 4. Security & Mesh Networking
*   **The "Viyu" Mesh:** Supports up to **2 simultaneous viewers**.
*   **Viewer Visibility:** Every connected parent can see a list of who else is currently viewing the feed.
*   **Authority Dashboard:** The Baby Unit displays an active viewer count and allows the physical user to "Kill" any session instantly.
*   **Heartbeat Protocol:** A 30-second "Ghost" timeout. If a parent loses signal without disconnecting, the Baby Unit clears the slot after 30s of inactivity.

### 5. Notifications & Alerts
*   **Hybrid Engine:** 
    *   Uses **Firebase Cloud Messaging (FCM)** to wake up the Parent Unit if the app has been killed by the OS.
    *   Uses **WebRTC Data Channels** for instant alerts if the app is currently open/backgrounded.
*   **Critical System Alerts:** 
    *   **Low Battery:** Notification fired when Baby Unit hits 15%.
    *   **Connection Lost:** The Parent Unit triggers an alarm if the heartbeat from the Baby Unit fails for >30s.
*   **Standard Volume:** Alerts respect the system notification volume and Do Not Disturb settings.

### 6. Logging & Data Retention
*   **Zero Logs on Baby Unit:** For privacy, the Baby Unit is stateless.
*   **Local History:** All events (Motion, Cry, Connections, Battery) are sent to the Parent Unit and stored in a local SQLite/AsyncStorage database for the "History" tab.

---

## 🚀 Deployment & Installation

### Signaling Server (Spring Boot)
1. Build the JAR using Maven/Gradle.
2. Deploy to a Linux VM (Oracle Cloud recommended).
3. Ensure ports for WebSockets are open.

### Mobile App (React Native)
1. **Baby Setup:** Select "Baby Unit" role -> Choose Camera -> Generate QR Code.
2. **Parent Setup:** Select "Parent Unit" role -> Scan QR Code -> Authenticate with Baby Unit.
3. **FCM Setup:** Add `google-services.json` for Play Store-like push notification support.

---

## ⚖️ License
**Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0)**

*This means you are free to use, modify, and build upon this code for personal use. However, you **CANNOT** use this software for commercial purposes, including but not limited to, selling it as a paid app on the Google Play Store or Apple App Store. Any derivative works must be shared under this same license.*
