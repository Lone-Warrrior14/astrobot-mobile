# 📱 AstraTalk Mobile (Android) - Comprehensive Code Details

This document provides an exhaustive, code-level explanation of every major component, native bridge, hardware accelerator integration, and client-side system in the **AstraTalk Mobile Application** (`astrobot-mobile`).

---

## 📑 Table of Contents
1. [Mobile System Architecture](#1-mobile-system-architecture)
2. [Native Bridge Layer: `OfflineEnginePlugin.java`](#2-native-bridge-layer-offlineenginepluginjava)
3. [Android Lifecycle Entrypoint: `MainActivity.java`](#3-android-lifecycle-entrypoint-mainactivityjava)
4. [Standalone React Frontend: `web-src/src/App.jsx`](#4-standalone-react-frontend-web-srcsrcappjsx)
5. [Client-Side Vector Database & Scoped RAG: `localVectorDb.js`](#5-client-side-vector-database--scoped-rag-localvectordbjs)
6. [Interactive Mascot Engine: `web-src/src/AstraCompanion.jsx`](#6-interactive-mascot-engine-web-srcsrcastracompanionjsx)
7. [Hardware Acceleration Pipelines (Snapdragon 8 Gen 2)](#7-hardware-acceleration-pipelines-snapdragon-8-gen-2)
8. [Google ML Kit Native OCR & PDF Processing](#8-google-ml-kit-native-ocr--pdf-processing)
9. [Android Hardware / Gesture Back Button Handling](#9-android-hardware--gesture-back-button-handling)

---

## 1. Mobile System Architecture

AstraTalk Mobile runs completely offline on Android without requiring an external server:
- **Presentation Container**: Capacitor 7 Android WebView loading an optimized Single Page Application built with React 18 and Vite (`web-src/`).
- **Bridge Engine**: `OfflineEnginePlugin.java` exposes hardware features and native C++ routines to JavaScript via `@PluginMethod`.
- **Hardware Targets**:
  - **NPU**: Qualcomm Hexagon HTP v73 for on-device Stable Diffusion Turbo / AnythingV5 (~4.1s generation).
  - **GPU**: Adreno 740 via OpenCL CLBlast kernels.
  - **CPU**: Multi-threaded ARM Cortex-X3 / Cortex-A715 cores for GGUF text generation.

---

## 2. Native Bridge Layer: `OfflineEnginePlugin.java`

Located at `android/app/src/main/java/com/astratalk/app/OfflineEnginePlugin.java`.

### Plugin Methods (`@PluginMethod`)

#### A. `generateImageNative(PluginCall call)`
- Receives `{ prompt: "..." }` from JavaScript.
- First verifies whether the Qualcomm QNN Stable Diffusion daemon is active on port `8085` (`http://127.0.0.1:8085/health`).
- If running, dispatches generation via HTTP POST with parameters: `{ steps: 15, cfg: 7, width: 512, height: 512 }`.
- Streams progress via Server-Sent Events. Upon the `complete` event:
  - Decodes base64 RGB raw bytes into Android `Bitmap`.
  - Saves the image to internal app storage (`/sdcard/Android/data/com.astratalk.app/files/generated_images/sd_npu_<time>.png`).
  - Resolves plugin call with `{ status: "success", path: outFile.getAbsolutePath(), timeMs: genTime, backend: "Qualcomm Hexagon HTP v73 NPU" }`.
- If QNN is unavailable, falls back to standalone OpenCL / CPU GGUF inference (`stable-diffusion-v1-5-Q4_0.gguf`).

#### B. `performOcrOnPdf(PluginCall call)` & `performOcrOnImage(PluginCall call)`
- Receives base64-encoded PDF or image bytes.
- Utilizes Android's native `android.graphics.pdf.PdfRenderer` to render PDF pages into bitmaps.
- Passes rendered bitmaps directly to Google ML Kit (`TextRecognition.getClient()`).
- Returns extracted textual content within **20–40ms** to JavaScript for indexing into the local vector database.

#### C. Model Loading & Inference (`loadModel`, `startInference`, `getNextToken`)
- Binds to native JNI C++ libraries (`native-lib.cpp`) compiled with the Android NDK.
- Loads quantized `.gguf` weights into memory.
- Tokenizes incoming ChatML prompts and streams back next-token predictions token-by-token.

---

## 3. Android Lifecycle Entrypoint: `MainActivity.java`

Located at `android/app/src/main/java/com/astratalk/app/MainActivity.java`.
- Extends Capacitor's `BridgeActivity`.
- Registers `OfflineEnginePlugin.class` during `onCreate`:
  ```java
  registerPlugin(OfflineEnginePlugin.class);
  super.onCreate(savedInstanceState);
  ```

---

## 4. Standalone React Frontend: `web-src/src/App.jsx`

`web-src/src/App.jsx` drives the mobile user interface.

### Features
- **Local Persistence**: Conversations are saved to `localStorage` under `astratalk_mobile_conversations`.
- **Image Turn Persistence**: When images are generated, the markdown string referencing `Capacitor.convertFileSrc(nativePath)` is committed into the active conversation so images remain visible across app restarts.
- **Top Navigation Bar**: Selects between local GGUF models, Snapdragon hardware presets, and optional Cloud Endpoints.
- **Tabs**:
  - `chat`: Main messaging feed with collapsible thinking boxes and mascot companion.
  - `documents`: On-device document library showing OCR-indexed files and token counts.
  - `mobile_models`: Download manager for on-device GGUF models.
  - `cloud_endpoints`: Custom OpenAI-compatible cloud endpoint manager.

---

## 5. Client-Side Vector Database & Scoped RAG: `localVectorDb.js`

`web-src/src/localVectorDb.js` implements a vector database running directly inside the mobile browser engine:
- **Storage**: Browser `IndexedDB` (`AstraTalkVectorDB`, store: `documents`).
- **Embeddings**: Computes 384-dimensional dense semantic vectors using an on-device TF-IDF / Subword semantic projection matrix.
- **Scoped Querying (`queryVectorDb`)**:
  - Mandatory filter: `doc.conversationId === activeConvId`.
  - Performs cosine similarity between the question vector and chunk vectors.
  - Returns top-3 context chunks to augment the local prompt, preventing cross-chat leakage.

---

## 6. Interactive Mascot Engine: `web-src/src/AstraCompanion.jsx`

`web-src/src/AstraCompanion.jsx` renders the companion mascot on mobile screens:
- **Asset Optimization**: High-performance local GIF rendering with touch event listeners (`onTouchStart`, `onTouchMove`, `onTouchEnd`).
- **Tactile Responses**:
  - Tap / Boop: Plays `boop.mp3` or spoken vocal greeting.
  - Head Rub: Detects finger scrubbing across head coordinates and loops `purr.wav`.
- **Fullscreen Companion Mode (`/astraonly`)**: Hides standard UI and focuses entirely on voice interaction, large animations, and speech lip-sync.

---

## 7. Hardware Acceleration Pipelines (Snapdragon 8 Gen 2)

- **Qualcomm QNN Daemon**: Interacts directly with the Snapdragon Hexagon HTP v73 tensor processor, achieving ~4.1-second Stable Diffusion inference.
- **Adreno 740 GPU**: Executes matrix multiplication via OpenCL kernels when running text generation.
- **Memory Management**: Keeps memory footprint under 3.5GB to ensure smooth multitasking with Android OS services.

---

## 8. Google ML Kit Native OCR & PDF Processing

- Embedded locally in `android/app/build.gradle` via `com.google.mlkit:text-recognition:16.0.0`.
- Zero network dependencies; works in airplane mode.
- Extracts clean text from photographed receipts, books, technical manuals, and multi-page research PDFs.

---

## 9. Android Hardware / Gesture Back Button Handling

Implemented using `@capacitor/app` (`CapApp.addListener('backButton')`):
1. **Sidebar Drawer Open**: Closes the drawer.
2. **Fullscreen Companion Open**: Returns to standard chat.
3. **Sub-Tab Active** (*Documents*, *Models*, *Endpoints*): Returns to `chat` tab.
4. **Main Chat Root**: Calls `CapApp.exitApp()` to safely suspend or exit the application.
