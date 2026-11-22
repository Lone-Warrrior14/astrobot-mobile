# 🌌 Astra Talk Mobile (Android)

**Astra Talk Mobile** is a standalone, privacy-first offline AI companion application deeply optimized for **Android devices powered by Snapdragon 8 Gen 2 (OnePlus 11, etc.)**.

Built with **Capacitor 7**, **React 18**, **Qualcomm QNN (Hexagon HTP v73 NPU)**, **Google ML Kit OCR**, and **Client-Side Vector RAG**.

---

## ⚡ Key Highlights & Capabilities

### 1. 🎨 On-Device NPU Stable Diffusion (Qualcomm Hexagon HTP v73)
- Fully offline image synthesis directly utilizing Snapdragon 8 Gen 2's Hexagon NPU.
- **Lightning Speed**: Generates 512×512 photorealistic images in **~4.1 seconds** with zero cloud latency.
- **Multi-Word Prompt Parsing**: Automatically routes natural queries (e.g. `create a black cat and white cat`, `draw sunset over mountains`) directly to the on-device NPU engine.
- **Persistent Image Gallery**: Generated PNGs are saved to internal storage and restored across app restarts via `Capacitor.convertFileSrc`.

### 2. 📄 Scoped In-Chat Vector RAG & Google ML Kit OCR
- **Google ML Kit Text Recognition**: Native OCR extracts text from images and multi-page PDFs (rendered via Android `PdfRenderer`) in **20–40ms**.
- **On-Device Vector Database**: Uses IndexedDB (`localVectorDb.js`) with 384-dimensional dense semantic vectors and cosine similarity search.
- **Strict Per-Chat Isolation**: Embeddings and document chunks are strictly bound to their respective `conversationId`, guaranteeing zero cross-chat contamination.

### 3. 🐱 Interactive Astra Mascot Companion (`/astra` & `/astraonly`)
- **Reactive Expressions**: Frame-by-frame state-driven animations (`hi`, `think`, `speaking`, `idle`, `celebration`, `sleep_loop`, `wake_up`).
- **Tactile Interactions**: Head poking triggers playful boop sounds; rubbing/petting triggers real acoustic cat purrs (`purr.wav`).
- **Speech Synthesis**: Real-time voice generation with natural cadence and emoji filtering.

### 4. 🎯 Android Hardware & Gesture Back Button Navigation
- Intercepts native Android back button/gesture:
  1. Closes navigation drawer if open.
  2. Exits fullscreen companion mode (`/astraonly`) back to main chat.
  3. Switches secondary tabs (*Documents & OCR*, *Snapdragon Models*, *Cloud Endpoints*) back to chat.
  4. Exits app cleanly only when pressed from root chat.

---

## 📁 Repository Structure
```
astrobot-mobile/
├── android/               # Complete Android Studio / Gradle native project
│   ├── app/               # Main application module & AndroidManifest.xml
│   ├── build.gradle       # Root gradle file
│   └── gradlew            # Gradle wrapper executable
├── web-src/               # Standalone Vite + React mobile frontend
│   ├── src/               # React UI, Astra mascot, and localVectorDb
│   └── package.json       # Web frontend dependencies
├── build_apk.sh           # One-command build script
├── package.json           # Capacitor CLI & plugin dependencies
├── capacitor.config.json  # Mobile app ID (com.astratalk.app) configuration
├── FLOW_AND_UML.md        # Comprehensive Mermaid architecture & UML diagrams
└── .gitignore             # Git ignore file
```

---

## 📐 Architecture & UML Flow Diagrams
Detailed architecture diagrams and interactive Mermaid sequence charts are available in:
👉 **[FLOW_AND_UML.md](FLOW_AND_UML.md)**

---

## 🚀 Building & Installing

### 1. Build Web Assets & Sync Capacitor
```bash
# Build React frontend
cd web-src
npm install
npm run build

# Sync assets to Android
cd ..
npm install
npx cap sync android
```

### 2. Assemble Debug APK
```bash
cd android
./gradlew assembleDebug
```

### 3. Deploy via ADB
```bash
adb install -r -d app/build/outputs/apk/debug/app-debug.apk
```

---

## 🔒 Privacy & Security Guarantee
- **100% Local Inference**: Zero data leaves your device.
- **Zero Third-Party Telemetry**: Fully functional offline.
