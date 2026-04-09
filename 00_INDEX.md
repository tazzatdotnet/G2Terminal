# Even Hub G2 Developer Knowledge Base — Master Index

> **Last verified**: April 2026
> **SDK Version**: `@evenrealities/even_hub_sdk` v0.0.9 (published Mar 25, 2026)
> **CLI Version**: `@evenrealities/evenhub-cli` v0.1.11 (published Mar 25, 2026)
> **Simulator Version**: `@evenrealities/evenhub-simulator` v0.6.2 (published Mar 25, 2026)
> **Even Hub Launch Date**: April 3, 2026 (public launch)

---

## Purpose of This Knowledge Base

This knowledge base is a comprehensive, AI-optimized reference for building applications on the **Even Hub** platform for the **Even Realities G2 smart glasses**. It is structured so that an AI assistant (Claude, GPT, etc.) can always reference the most accurate, up-to-date, and clean information when helping a developer build, debug, or deploy an Even Hub app.

Every file in this knowledge base is sourced directly from official documentation, official GitHub repositories, official npm packages, and verified community resources. Nothing is fabricated.

---

## File Index

| File | Title | Description |
|------|-------|-------------|
| `00_INDEX.md` | **Master Index** | This file. Navigation, critical notes, live resource URLs |
| `01_PLATFORM_OVERVIEW.md` | **Platform Overview** | G2 hardware, Even Hub ecosystem, development paths |
| `02_SDK_QUICKSTART.md` | **SDK Quickstart** | Installation, project setup, minimal working app |
| `03_SDK_API_REFERENCE.md` | **SDK API Reference** | Full `EvenAppBridge` API, all methods, all types |
| `04_UI_CONTAINERS.md` | **UI Containers** | Canvas system, container types, layout rules |
| `05_EVENTS_AND_INPUT.md` | **Events & Input** | All event types, IMU, audio, TouchBar, R1 ring |
| `06_BLE_PROTOCOL.md` | **BLE Protocol** | Low-level Bluetooth protocol (native companion apps) |
| `07_EXAMPLES_AND_PATTERNS.md` | **Examples & Patterns** | CLI, Simulator, even-dev, packaging, deployment |
| `08_TROUBLESHOOTING.md` | **Troubleshooting** | Real-world app examples, patterns, open-source repos |
| `09_ADVANCED_TOPICS.md` | **Advanced Topics** | Audio, IMU, image pipeline, multi-page, AI integration |

---

## Live Authoritative Resource URLs

Always check these sources for the latest information. These are the canonical references.

### Official Even Realities

| Resource | URL |
|----------|-----|
| Even Hub Portal | https://hub.evenrealities.com |
| Even Hub Developer Docs | https://hub.evenrealities.com/docs |
| Even Realities GitHub Org | https://github.com/even-realities |
| Even Realities Support Center | https://support.evenrealities.com |

### Official npm Packages

| Package | URL | Current Version |
|---------|-----|-----------------|
| `@evenrealities/even_hub_sdk` | https://www.npmjs.com/package/@evenrealities/even_hub_sdk | 0.0.9 |
| `@evenrealities/evenhub-cli` | https://www.npmjs.com/package/@evenrealities/evenhub-cli | 0.1.11 |
| `@evenrealities/evenhub-simulator` | https://www.npmjs.com/package/@evenrealities/evenhub-simulator | 0.6.2 |

### Official GitHub Repositories

| Repo | URL | Description |
|------|-----|-------------|
| EvenDemoApp | https://github.com/even-realities/EvenDemoApp | Official Flutter/Dart BLE demo app |
| EH-InNovel | https://github.com/even-realities/EH-InNovel | Official Kotlin Multiplatform SDK demo |
| lvgl-sys-v9 | https://github.com/even-realities/lvgl-sys-v9 | LVGL Rust bindings (firmware-level) |

### Community Resources

| Resource | URL | Description |
|----------|-----|-------------|
| even-dev | https://github.com/BxNxM/even-dev | Unified multi-app dev environment + simulator launcher |
| even-g2-protocol | https://github.com/i-soxi/even-g2-protocol | Community BLE reverse engineering |
| even-g2-notes | https://github.com/nickustinov/even-g2-notes | Community SDK reference docs (independent research) |
| even-better-sdk | https://www.npmjs.com/package/@jappyjan/even-better-sdk | Community wrapper SDK (higher-level API) |
| even-realities-ui | https://www.npmjs.com/package/@jappyjan/even-realities-ui | Community UI component library |

---

## Critical Notes for AI Assistants

Read these before generating any code or advice for Even Hub development.

### 1. Even Hub Apps Are WebView Apps
Even Hub apps are **standard web applications** (HTML + TypeScript/JavaScript) that run inside a WebView embedded in the Even App on the user's iPhone. They are **not** native mobile apps. The SDK (`@evenrealities/even_hub_sdk`) provides a bridge between the WebView and the native Even App, which in turn communicates with the glasses over Bluetooth.

### 2. Two Completely Different Development Paths
There are **two distinct ways** to build for the G2:

- **Path A — Even Hub WebView App** (recommended for most developers): Build a web app using the Even Hub SDK. The Even App handles all BLE communication. You never touch Bluetooth directly.
- **Path B — Native Companion App with BLE Protocol**: Build a native mobile app (iOS/Android) that communicates directly with the glasses over Bluetooth using the raw BLE protocol. This is what `EvenDemoApp` (the official Flutter demo) does. Significantly more complex.

**Do not mix these paths.** If a developer is using the SDK, they do not need to know BLE commands. If they are building a native companion app, they do not use the SDK.

### 3. SDK Initialization Is Async — Always Await
The bridge is not available synchronously. Always use:
```typescript
const bridge = await waitForEvenAppBridge();
```
Never call `EvenAppBridge.getInstance()` without first ensuring the bridge is ready via `waitForEvenAppBridge()`.

### 4. `createStartUpPageContainer` Is a One-Time Call
`createStartUpPageContainer` can only be called **once** per app session. After the first call, all subsequent UI updates must use `rebuildPageContainer`. Calling `createStartUpPageContainer` again after the first call will have no effect.

### 5. Canvas Dimensions Are 576×288 Pixels
The G2 display canvas is exactly **576 pixels wide × 288 pixels tall**. Origin (0,0) is at the **top-left**. X increases to the right, Y increases downward. The display is **4-bit greyscale** (16 shades). Colors are specified as values 0–15.

### 6. Image Transmission Must Be Sequential (No Concurrency)
Never send multiple images concurrently. Always wait for `updateImageRawData` to return success before sending the next image. The glasses have limited memory and will fail on concurrent image sends. Also: prefer images with simple, single-tone colors; avoid frequent image updates.

### 7. Container Limits
- `containerTotalNum` must be **1–12**
- Maximum **8 text containers** per page (`textObject` max 8 items)
- Exactly **one** container must have `isEventCapture: 1` — all others must be `0`
- Image containers: width range **20–288**, height range **20–144**
- Image containers cannot receive image data during `createStartUpPageContainer` — call `updateImageRawData` after creation
- List items: max **20 items** per list, each item name max **64 characters**
- Container names: max **16 characters**
- Text content (startup): max **1000 characters**; text content (upgrade): max **2000 characters**

### 8. `onLaunchSource` Fires Once
The `onLaunchSource` event fires exactly **once** after the WebView finishes loading. Register the listener early. Values are `'appMenu'` (opened from phone app menu) or `'glassesMenu'` (opened from glasses menu). Initialize glasses UI only when `source === 'glassesMenu'`.

### 9. Audio and IMU Require `createStartUpPageContainer` First
Both `audioControl(true)` and `imuControl(true, pace)` will fail if called before `createStartUpPageContainer` has been successfully called.

### 10. IMU Report Frequency Uses Enum Values, Not Hz
`imuControl` takes an `ImuReportPace` enum: `P100`, `P200`, ..., `P1000` (step 100). These are **protocol pace codes**, not physical Hz values. Default is `ImuReportPace.P100`.

### 11. Audio PCM Spec (Simulator)
The simulator emits audio events at: **16000 Hz sample rate**, **signed 16-bit little-endian PCM**, **100ms per event** (3200 bytes / 1600 samples).

### 12. The Simulator Is Not 100% Accurate
Known differences from hardware:
- Font rendering may differ
- List scroll behavior / focused-item positioning can vary
- Image size constraints are **not enforced** in simulator (hardware is stricter)
- Status events are hardcoded (not emitted dynamically)
- `imuData` is always `null` in simulator
- `eventSource` is hardcoded to `1` (`TOUCH_EVENT_FROM_GLASSES_R`)
- Error-response handling may differ

Always test on physical G2 glasses before publishing.

### 13. Use SDK Storage, Not Browser `localStorage`
Even Hub apps run in a WebView. Do **not** use browser `localStorage` — use `bridge.setLocalStorage()` / `bridge.getLocalStorage()` which persist data to the native app side.

### 14. `app.json` Is Required for Publishing
Every app published to Even Hub must have an `app.json` metadata file. The `package_id` must be a valid lowercase dot-separated identifier (e.g., `com.yourname.appname`). Use `evenhub pack app.json ./dist` to package into an `.ehpk` file.

### 15. Even Hub Launched April 3, 2026
Even Hub officially launched on April 3, 2026. The platform is actively evolving — always check npm for the latest SDK version before starting a project.

### 16. `rebuildPageContainer` vs `createStartUpPageContainer`
These two methods have **identical parameter structures** (`CreateStartUpPageContainer` / `RebuildPageContainer` are the same shape). The only difference is their role: `createStartUpPageContainer` is called once to initialize; `rebuildPageContainer` is used for all subsequent page updates and new page creation.

### 17. `imageData` Accepted Formats
`updateImageRawData.imageData` accepts: `number[]` (recommended), `Uint8Array`, `ArrayBuffer`, or base64 `string`. The SDK auto-converts `Uint8Array`/`ArrayBuffer` to `number[]` during serialization.

---

## Quick Decision Tree

```
I want to build for the G2...

├── As a web app (recommended)
│   ├── Install: npm install @evenrealities/even_hub_sdk
│   ├── Dev tool: npx @evenrealities/evenhub-simulator@latest <url>
│   ├── QR sideload: npx @evenrealities/evenhub-cli qr
│   ├── Multi-app env: github.com/BxNxM/even-dev
│   └── See: 02_SDK_QUICKSTART.md, 03_SDK_API_REFERENCE.md
│
├── As a native companion app (advanced, BLE)
│   ├── Reference: github.com/even-realities/EvenDemoApp
│   ├── Protocol: github.com/i-soxi/even-g2-protocol
│   └── See: 06_BLE_PROTOCOL.md
│
└── I want to use Kotlin/Multiplatform
    ├── Reference: github.com/even-realities/EH-InNovel
    └── See: 02_SDK_QUICKSTART.md (Kotlin section)
```

---

## Key API Surface at a Glance

```typescript
// Init
const bridge = await waitForEvenAppBridge();

// One-time UI setup
await bridge.createStartUpPageContainer({ containerTotalNum, listObject, textObject, imageObject });

// Subsequent UI updates
await bridge.rebuildPageContainer({ containerTotalNum, listObject, textObject, imageObject });

// Update image (sequential only, never concurrent)
await bridge.updateImageRawData({ containerID, containerName, imageData });

// Update text
await bridge.textContainerUpgrade({ containerID, containerName, content, contentOffset, contentLength });

// Shutdown
await bridge.shutDownPageContainer(0); // 0=immediate, 1=prompt user

// Events
bridge.onLaunchSource((source) => { /* 'appMenu' | 'glassesMenu' */ });
bridge.onDeviceStatusChanged((status) => { /* DeviceStatus */ });
bridge.onEvenHubEvent((event) => { /* listEvent | textEvent | sysEvent | audioEvent */ });

// Sensors
await bridge.audioControl(true);                        // mic on
await bridge.imuControl(true, ImuReportPace.P500);      // IMU on at P500 pace

// Storage (use these, NOT browser localStorage)
await bridge.setLocalStorage('key', 'value');
const val = await bridge.getLocalStorage('key');

// User & device info
const user = await bridge.getUserInfo();    // { uid, name, avatar, country }
const device = await bridge.getDeviceInfo(); // { model, sn, status }
```