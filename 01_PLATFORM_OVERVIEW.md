---

# 01 — Platform Overview

> **Source**: Official npm packages, official GitHub repos, community research (`nickustinov/even-g2-notes`). All facts verified April 2026.

---

## What Is Even Hub?

**Even Hub** is the official third-party app platform for the **Even Realities G2 smart glasses**, launched publicly on **April 3, 2026**. It allows developers to build and publish web-based apps that run on the G2 glasses via the Even App on the user's iPhone.

Even Hub apps are **standard web applications** — HTML, CSS, TypeScript/JavaScript — hosted on any server. The Even App on the user's iPhone loads your web app in a WebView and acts as a relay between your app and the glasses over Bluetooth. You do not write firmware, you do not write native mobile code (for Path A), and you do not touch Bluetooth directly.

At launch, Even Hub shipped with ~50 apps and a developer community of 2,000+ members.

---

## G2 Hardware Specifications

| Component | Specification |
|-----------|---------------|
| **Display** | Dual micro-LED (one per lens), green monochrome |
| **Canvas resolution** | 576 × 288 pixels per eye |
| **Color depth** | 4-bit greyscale — 16 shades of green. White = bright green; black = off |
| **Sync** | Both lenses synced via physical FPC cable (not wireless) |
| **Connectivity** | BLE 5.x — ~28m real-world range (~9dB more power than G1) |
| **Microphone** | Yes — accessible via SDK |
| **Camera** | ❌ None |
| **Speaker** | ❌ None |
| **Touch input** | Temple tips (tap, double-tap, swipe gestures) |
| **Ring input** | R1 Smart Ring — separate BLE device for scroll/click |
| **IMU / Gyroscope** | Yes — accessible via SDK (`imuControl`) |
| **Wearing detection** | Yes — reported via `DeviceStatus.isWearing` |
| **Charging** | Charging case — reported via `DeviceStatus.isInCase` / `isCharging` |
| **Battery** | Reported as 0–100 via `DeviceStatus.batteryLevel` |
| **Paired to** | iPhone only (via Even App, Flutter-based) |

### Display Detail: 4-Bit Greyscale
The G2 display renders **16 shades of green** (4-bit). All images and UI elements are converted to this color space by the host app before transmission. There is no RGB, no true black-and-white — everything is a shade of green. "Black" pixels are simply off (no light emitted), which is why black backgrounds look clean on the micro-LED display.

### R1 Smart Ring
The R1 ring is a **separate BLE device** that pairs independently with the iPhone. It provides:
- **Scroll up / scroll down** — moves selection in lists, scrolls text
- **Single tap (click)** — confirms selection
- **Double tap** — secondary action

The ring's input events arrive in your app via the same `onEvenHubEvent` callback as temple touch events. You do not need to handle the ring separately — the SDK unifies all input sources.

---

## The Even Hub App Architecture

### Connection Model

```
[Your Server]  <──HTTPS──>  [iPhone WebView]  <──BLE──>  [G2 Glasses]
     ↑                            ↑
  Your backend              Even App (Flutter)
  API keys, DB,             flutter_inappwebview
  heavy compute             SDK bridge injected
```

**Step by step:**
1. Your web app is hosted on any server (Vercel, Cloudflare, your own VPS, etc.)
2. The user opens your app from the Even Hub menu on their iPhone or from the glasses menu
3. The Even App (Flutter) opens your app's URL in a `flutter_inappwebview` WebView
4. The SDK injects an `EvenAppBridge` object into the WebView's `window` object
5. Your JavaScript calls the bridge to send UI commands to the glasses
6. The Flutter app relays those commands over BLE to the glasses
7. The glasses render the UI and send back input events (taps, scrolls)
8. The Flutter app pushes those events into your WebView via `window._listenEvenAppMessage(...)`

### Two-Way Communication

| Direction | Path |
|-----------|------|
| **Web → Glasses** | `bridge.callEvenApp(method, params)` → Flutter WebView handler → BLE → Glasses |
| **Glasses → Web** | Glasses → BLE → Flutter → `window._listenEvenAppMessage(...)` → your callback |

### What the iPhone Does (and Doesn't Do)
- ✅ Loads your web app URL in a WebView
- ✅ Relays messages between your WebView and the glasses over BLE
- ✅ Injects the SDK bridge into the WebView
- ✅ Handles BLE pairing, authentication, and connection management
- ❌ Does **not** run your app logic — it just loads your page
- ❌ Does **not** process your data — your backend does that

### Implications for App Design
- Your **backend** can hold API keys, call third-party APIs, do heavy computation — the glasses UI is a thin frontend
- Standard web security applies: HTTPS, session tokens, server-side secrets
- The WebView has **full browser capabilities**: `fetch`, `WebSocket`, `Web Audio API`, etc.
- **Do not use browser `localStorage`** — it does not survive app or glasses restarts inside the `.ehpk` WebView. Use `bridge.setLocalStorage` / `bridge.getLocalStorage` instead
- Auto-connect on page load — the user's phone is not always accessible when wearing glasses, so never require a manual "Connect" button tap as the only way to connect

---

## Two Development Paths

### Path A — Even Hub WebView App (Recommended)

Build a standard web app using the `@evenrealities/even_hub_sdk`. The Even App handles all BLE communication. You never touch Bluetooth directly.

**Best for:** Most developers. Any web developer can build this.

**Stack:**
- Any web framework (Vite, React, Vue, Svelte, vanilla JS, etc.)
- `@evenrealities/even_hub_sdk` for glasses communication
- Any backend (Node, Python, Cloudflare Workers, etc.)
- Any hosting (Vercel, Cloudflare Pages, your own server)

**Development tools:**
- `@evenrealities/evenhub-simulator` — desktop simulator for rapid iteration
- `@evenrealities/evenhub-cli` — QR code sideloading + packaging
- `even-dev` (community) — unified multi-app dev environment

**Publishing:**
- Package with `evenhub pack app.json ./dist` → `.ehpk` file
- Submit to Even Hub portal

**See:** `02_SDK_QUICKSTART.md`, `03_SDK_API_REFERENCE.md`, `04_UI_CONTAINERS.md`

---

### Path B — Native Companion App with BLE Protocol (Advanced)

Build a native iOS or Android app that communicates directly with the glasses over Bluetooth using the raw BLE protocol. This is what the official `EvenDemoApp` (Flutter/Dart) does.

**Best for:** Advanced developers who need capabilities not exposed by the SDK, or who want to build a fully native companion app.

**Stack:**
- Flutter/Dart, Swift, Kotlin, or any native mobile framework
- Raw BLE 5.x communication
- Custom packet construction per the Even Realities BLE protocol

**Complexity:** Significantly higher. You must handle:
- BLE device discovery and pairing
- Authentication handshake
- Packet framing and sequencing
- Image encoding (1-bit BMP, 576×136px for the BLE path)
- All error handling at the protocol level

**Reference:**
- `github.com/even-realities/EvenDemoApp` — official Flutter demo
- `github.com/i-soxi/even-g2-protocol` — community BLE reverse engineering

**See:** `06_BLE_PROTOCOL.md`

---

### ⚠️ Do Not Mix Paths
If you are using the SDK (Path A), you do not need to know BLE commands. If you are building a native companion app (Path B), you do not use the SDK. These are completely separate approaches.

---

## Even Hub Ecosystem Overview

### Official Packages

| Package | Purpose | Version |
|---------|---------|---------|
| `@evenrealities/even_hub_sdk` | TypeScript SDK for WebView apps | 0.0.9 |
| `@evenrealities/evenhub-cli` | CLI: QR sideload, init, pack, login | 0.1.11 |
| `@evenrealities/evenhub-simulator` | Desktop simulator for development | 0.6.2 |

### Community Packages

| Package | Purpose |
|---------|---------|
| `@jappyjan/even-better-sdk` | Higher-level wrapper around the official SDK |
| `@jappyjan/even-realities-ui` | UI component library for Even Hub apps |

### Official GitHub Repos

| Repo | Description |
|------|-------------|
| `even-realities/EvenDemoApp` | Flutter/Dart BLE demo (Path B reference) |
| `even-realities/EH-InNovel` | Kotlin Multiplatform SDK demo (novel reader) |
| `even-realities/lvgl-sys-v9` | LVGL Rust bindings (firmware-level, not for app devs) |

### Community GitHub Repos

| Repo | Description |
|------|-------------|
| `BxNxM/even-dev` | Unified multi-app dev environment + simulator launcher |
| `i-soxi/even-g2-protocol` | BLE protocol reverse engineering |
| `nickustinov/even-g2-notes` | Community SDK reference documentation |

---

## Platform Constraints and Limitations

### Display Constraints
- **No CSS, no DOM, no flexbox** — the glasses UI is not a browser. You define containers with pixel coordinates; the glasses firmware renders them
- **No font selection** — single LVGL font baked into firmware
- **No font size control** — one size only
- **No text alignment** — left-aligned, top-aligned only. To "center" text, manually pad with spaces
- **No background color or fill** — containers have borders only; no fill color
- **No animations or transitions** — static rendering only
- **No z-index control** — containers draw in declaration order; later containers draw on top
- **No per-item styling in lists** — all items are plain text, single line, same style
- **No programmatic scroll control** — firmware handles internal scrolling; no API to get/set scroll offset

### Input Constraints
- **No touchscreen** — input is temple taps and R1 ring only
- **No keyboard** — text input must come from your phone UI or voice (microphone)
- **No mouse/pointer** — no hover states, no cursor

### Audio Constraints
- **No speaker** — audio output is impossible on the hardware
- **Microphone only** — PCM input at 16kHz, 10ms frames, 40 bytes/frame, S16LE mono
- **Requires `createStartUpPageContainer` first** — `audioControl(true)` fails without it

### Memory and Performance Constraints
- **Limited glasses memory** — avoid frequent image updates; avoid large images
- **Sequential image transmission only** — never send images concurrently; always await success before sending the next
- **Image size limits** — width: 20–200px, height: 20–100px (hardware enforced; simulator is more lenient)
- **Container limits** — max 4 containers per page, max 8 text containers, exactly 1 event capture container

### Storage Constraints
- **No browser `localStorage`** — wiped on app/glasses restart inside `.ehpk` WebView
- **SDK storage only** — `bridge.setLocalStorage` / `bridge.getLocalStorage` (key-value, string only)
- **No `removeLocalStorage`** — write empty string and treat as absent

### SDK Constraints
- **No direct BLE access** from SDK
- **No arbitrary pixel drawing** — limited to list/text/image container model
- **No `imgEvent`** — defined in protocol but not exposed in SDK types
- **No audio output**
- **No image color** — 4-bit greyscale only (16 levels of green)

---

## What the SDK Does NOT Expose

This is a critical list for AI assistants — do not suggest these capabilities:

| Capability | Status |
|-----------|--------|
| Direct BLE access | ❌ Not available in SDK |
| Arbitrary pixel drawing | ❌ Not available — container model only |
| `imgEvent` | ❌ Defined in protocol, not in SDK types |
| Audio output / speaker | ❌ No speaker on hardware |
| Text alignment (center, right) | ❌ Left-aligned only |
| Font size / weight / family | ❌ Single firmware font |
| Background color / fill | ❌ Borders only |
| Per-item list styling | ❌ Plain text only |
| Programmatic scroll position | ❌ Firmware-controlled |
| Animations / transitions | ❌ Static only |
| Camera | ❌ No camera on hardware |
| GPS / location | ❌ Not exposed |
| Haptics | ❌ Not exposed |

---