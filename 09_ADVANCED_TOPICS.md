# 09 — Advanced Topics

> **Purpose:** Deep dives into advanced Even Hub development — IMU/sensor data,
> `callEvenApp`, Kotlin Multiplatform, the `even-better-sdk` community library,
> `evenhub-cli` internals, advanced BLE patterns, multi-page architecture,
> canvas/image rendering, voice AI pipelines, backend architecture, security,
> testing, CI/CD, and performance profiling.
>
> **Prerequisites:** Read `02_SDK_QUICKSTART.md`, `03_SDK_API_REFERENCE.md`,
> `04_UI_CONTAINERS.md`, `05_EVENTS_AND_INPUT.md` first.

---

## Table of Contents

1. [IMU & Sensor Data Deep Dive](#1-imu--sensor-data-deep-dive)
2. [callEvenApp — Native Bridge](#2-callevenapp--native-bridge)
3. [Kotlin Multiplatform Apps](#3-kotlin-multiplatform-apps)
4. [even-better-sdk Community Library](#4-even-better-sdk-community-library)
5. [evenhub-cli Deep Dive](#5-evenhub-cli-deep-dive)
6. [Advanced State Management](#6-advanced-state-management)
7. [Advanced Multi-Page Architecture](#7-advanced-multi-page-architecture)
8. [R1 Ring Advanced Patterns](#8-r1-ring-advanced-patterns)
9. [Advanced BLE Patterns](#9-advanced-ble-patterns)
10. [Canvas API & Image Rendering Pipeline](#10-canvas-api--image-rendering-pipeline)
11. [Animation & Motion on the Glasses](#11-animation--motion-on-the-glasses)
12. [Voice AI Pipeline (End-to-End)](#12-voice-ai-pipeline-end-to-end)
13. [Backend Architecture for Even Hub Apps](#13-backend-architecture-for-even-hub-apps)
14. [Security Considerations](#14-security-considerations)
15. [Offline & Resilience Patterns](#15-offline--resilience-patterns)
16. [Testing Strategy](#16-testing-strategy)
17. [Packaging & Distribution](#17-packaging--distribution)
18. [CI/CD for Even Hub Apps](#18-cicd-for-even-hub-apps)
19. [Performance Profiling](#19-performance-profiling)
20. [Quick-Reference Summary](#20-quick-reference-summary)

---

## 1. IMU & Sensor Data Deep Dive

### What the G2 Exposes

The G2 glasses contain a 6-axis IMU (3-axis accelerometer + 3-axis gyroscope).
Data is streamed via BLE to the phone and surfaced through the SDK.

```typescript
interface IMUData {
  ax: number;  // Accelerometer X (m/s²) — left/right tilt
  ay: number;  // Accelerometer Y (m/s²) — forward/back tilt
  az: number;  // Accelerometer Z (m/s²) — up/down (gravity ~9.8)
  gx: number;  // Gyroscope X (°/s) — pitch rate
  gy: number;  // Gyroscope Y (°/s) — yaw rate
  gz: number;  // Gyroscope Z (°/s) — roll rate
  timestamp: number;  // ms since epoch
}

bridge.addIMUDataListener((data: IMUData) => {
  console.log('IMU:', data);
});
```

### Wearing Detection

```typescript
bridge.addWearingStatusListener((status: { wearing: boolean }) => {
  if (status.wearing) {
    console.log('Glasses put on — resume display');
    animator.start(renderFrame);
  } else {
    console.log('Glasses removed — pause to save battery');
    animator.stop();
  }
});
```

### Head Gesture Detector

Classify nod, shake, and tilt gestures from a rolling window of IMU samples:

```typescript
class HeadGestureDetector {
  private history: IMUData[] = [];
  private readonly windowMs = 500;

  feed(data: IMUData): 'nod' | 'shake' | 'tilt_left' | 'tilt_right' | null {
    this.history.push(data);
    const cutoff = Date.now() - this.windowMs;
    this.history = this.history.filter(d => d.timestamp > cutoff);
    return this.classify();
  }

  private classify(): 'nod' | 'shake' | 'tilt_left' | 'tilt_right' | null {
    if (this.history.length < 5) return null;

    const gyY = this.history.map(d => d.gy);
    const gyZ = this.history.map(d => d.gz);

    const peakY = Math.max(...gyY.map(Math.abs));
    const peakZ = Math.max(...gyZ.map(Math.abs));

    const crossingsY = this.zeroCrossings(gyY);
    const crossingsZ = this.zeroCrossings(gyZ);

    if (peakY > 80 && crossingsY >= 2) return 'nod';
    if (peakZ > 80 && crossingsZ >= 2) return 'shake';

    const lastAx = this.history[this.history.length - 1].ax;
    if (lastAx >  4) return 'tilt_right';
    if (lastAx < -4) return 'tilt_left';

    return null;
  }

  private zeroCrossings(values: number[]): number {
    let count = 0;
    for (let i = 1; i < values.length; i++) {
      if (values[i - 1] * values[i] < 0) count++;
    }
    return count;
  }
}

const gestureDetector = new HeadGestureDetector();

bridge.addIMUDataListener((data) => {
  const gesture = gestureDetector.feed(data);
  if (gesture === 'nod')        handleNod();
  else if (gesture === 'shake') handleShake();
  else if (gesture)             handleTilt(gesture);
});
```

### Complementary Filter (Orientation Estimation)

Raw gyroscope data drifts over time. A complementary filter fuses accel + gyro:

```typescript
class ComplementaryFilter {
  private pitch = 0;
  private roll  = 0;
  private lastTs: number | null = null;
  private readonly alpha = 0.98;   // gyro weight

  update(data: IMUData): { pitch: number; roll: number } {
    const now = data.timestamp;
    const dt  = this.lastTs ? (now - this.lastTs) / 1000 : 0;
    this.lastTs = now;

    // Accel-based angles (degrees)
    const accelPitch = Math.atan2(data.ay, data.az) * (180 / Math.PI);
    const accelRoll  = Math.atan2(data.ax, data.az) * (180 / Math.PI);

    // Fuse
    this.pitch = this.alpha * (this.pitch + data.gx * dt) +
                 (1 - this.alpha) * accelPitch;
    this.roll  = this.alpha * (this.roll  + data.gy * dt) +
                 (1 - this.alpha) * accelRoll;

    return { pitch: this.pitch, roll: this.roll };
  }
}
```

### IMU Sampling Rate & Battery

- IMU data arrives at approximately **50 Hz** over BLE
- Continuous IMU listening increases battery drain on both glasses and phone
- **Best practice:** Only enable IMU when the feature actively needs it

```typescript
// Enable only when needed
await bridge.startIMU();
// ... use IMU ...
await bridge.stopIMU();
```

---

## 2. callEvenApp — Native Bridge

### What It Does

`callEvenApp` is a low-level escape hatch that sends arbitrary commands to the
native Even Hub iOS app. It is used for features not yet exposed as first-class
SDK methods.

```typescript
const result = await bridge.callEvenApp({
  command: 'COMMAND_NAME',
  params: { key: 'value' },
});
```

### Known Commands (Community-Documented)

| Command | Params | Returns | Notes |
|---|---|---|---|
| `GET_DEVICE_INFO` | `{}` | Device info object | Same as `getDeviceInfo()` |
| `SET_BRIGHTNESS` | `{ level: 0–100 }` | `{ success: boolean }` | Glasses display brightness |
| `GET_BATTERY` | `{}` | `{ glasses: number, ring: number }` | Battery % for each device |
| `TRIGGER_HAPTIC` | `{ pattern: 'short'\|'long'\|'double' }` | `{ success: boolean }` | R1 ring haptic feedback |
| `GET_FIRMWARE_VERSION` | `{}` | `{ version: string }` | Glasses firmware string |

> ⚠️ **Warning:** `callEvenApp` commands are undocumented and may change between
> SDK versions. Prefer official SDK methods where available. Always handle
> errors gracefully.

### Safe Wrapper Pattern

```typescript
async function callEvenAppSafe<T>(
  bridge: EvenAppBridge,
  command: string,
  params: Record<string, unknown> = {}
): Promise<T | null> {
  try {
    const result = await bridge.callEvenApp({ command, params });
    return result as T;
  } catch (err) {
    console.warn(`callEvenApp(${command}) failed:`, err);
    return null;
  }
}

// Usage
const battery = await callEvenAppSafe<{ glasses: number; ring: number }>(
  bridge, 'GET_BATTERY'
);
if (battery) {
  await bridge.updateTextData({
    containerId: batteryTextId,
    text: `🔋 Glasses ${battery.glasses}%  Ring ${battery.ring}%`,
  });
}
```

### Haptic Feedback via callEvenApp

```typescript
async function haptic(
  bridge: EvenAppBridge,
  pattern: 'short' | 'long' | 'double' = 'short'
): Promise<void> {
  await callEvenAppSafe(bridge, 'TRIGGER_HAPTIC', { pattern });
}

// Confirm action with haptic
bridge.addContainerEventListener(listContainerId, async (event) => {
  await haptic(bridge, 'short');
  await handleSelection(event.subId);
});
```

---

## 3. Kotlin Multiplatform Apps

### When to Use KMP

Use Kotlin Multiplatform (KMP) when:
- Your team is primarily Kotlin/Android developers
- You want to share business logic between Android and the Even Hub WebView
- You need Compose Multiplatform UI on both platforms

### Project Structure (from EH-InNovel reference app)

```
composeApp/
├── src/
│   ├── webMain/kotlin/
│   │   └── .../
│   │       ├── sdk/
│   │       │   ├── EvenHubBridge.kt        ← Shared expect API
│   │       │   ├── EvenHubTypes.kt         ← Data classes (shared)
│   │       │   ├── EvenHubJsParsers.kt     ← JSON serialization
│   │       │   └── EvenHubJsInterop.kt     ← JS interop utilities
│   │       └── ui/                         ← Compose Multiplatform UI
│   ├── jsMain/kotlin/
│   │   └── .../sdk/
│   │       └── EvenHubBridge.js.kt         ← JS actual implementation
│   └── wasmJsMain/kotlin/
│       └── .../
│           └── EvenHubBridge.wasmJs.kt     ← Wasm actual implementation
└── build.gradle.kts
```

### Bridge Initialization (KMP)

```kotlin
// webMain — shared expect declaration
expect class EvenHubBridge {
    suspend fun ensureReady(): Unit
    suspend fun updateText(containerId: Int, text: String): Unit
    suspend fun getUser(): UserInfo
}

// jsMain — JS actual
actual class EvenHubBridge {
    private var bridge: dynamic = null

    actual suspend fun ensureReady() {
        bridge = waitForEvenAppBridge()   // calls JS SDK
    }

    actual suspend fun updateText(containerId: Int, text: String) {
        bridge.updateTextData(js("({ containerId, text })"))
    }

    actual suspend fun getUser(): UserInfo {
        val raw = bridge.getUserInfo().await()
        return UserInfo(name = raw.name as String)
    }
}
```

### Compose UI Example

```kotlin
@Composable
fun GlassesStatusCard(bridge: EvenHubBridge) {
    var status by remember { mutableStateOf("Connecting...") }

    LaunchedEffect(Unit) {
        bridge.ensureReady()
        val user = bridge.getUser()
        status = "Hello, ${user.name}"
    }

    Card(modifier = Modifier.fillMaxWidth().padding(16.dp)) {
        Text(text = status, style = MaterialTheme.typography.h6)
    }
}
```

### Gradle Dependencies

```kotlin
// build.gradle.kts
kotlin {
    js(IR) { browser() }
    wasmJs { browser() }

    sourceSets {
        val webMain by getting {
            dependencies {
                implementation(npm("@evenrealities/even_hub_sdk", "0.0.9"))
                implementation("org.jetbrains.compose.runtime:runtime:1.6.0")
            }
        }
    }
}
```

---

## 4. even-better-sdk Community Library

### Overview

`@jappyjan/even-better-sdk` (v0.0.8) is a community wrapper around the
official SDK that adds:

- Opinionated page composition API
- Partial text updates (update one line without rebuilding the whole container)
- Shared SDK initialization (singleton pattern)
- Configurable logging

```bash
npm install @jappyjan/even-better-sdk
```

### Initialization

```typescript
import { EvenBetterSDK } from '@jappyjan/even-better-sdk';

const sdk = await EvenBetterSDK.getInstance({
  logLevel: 'info',   // 'debug' | 'info' | 'warn' | 'error' | 'none'
});
```

### Page Composition API

```typescript
import { Page, TextElement, ListElement } from '@jappyjan/even-better-sdk';

const homePage = new Page('home');

homePage.add(new TextElement({
  id: 'title',
  x: 0, y: 0, width: 576, height: 40,
  text: 'My App',
  fontSize: 24,
}));

homePage.add(new ListElement({
  id: 'menu',
  x: 0, y: 50, width: 576, height: 238,
  isEventCapture: true,
  items: [
    { id: 0, label: 'Option A' },
    { id: 1, label: 'Option B' },
  ],
}));

await sdk.showPage(homePage);
```

### Partial Text Updates

```typescript
// Update a single line without rebuilding the container
await sdk.updateText('title', 'Updated Title');

// Update multiple lines atomically
await sdk.updateTexts({
  title:    'New Title',
  subtitle: 'New Subtitle',
});
```

### When to Use even-better-sdk vs Official SDK

| Scenario | Recommendation |
|---|---|
| New project, want faster iteration | `even-better-sdk` |
| Need maximum control / low-level access | Official SDK |
| Production app, stability critical | Official SDK (fewer unknowns) |
| Rapid prototyping | `even-better-sdk` |
| Kotlin Multiplatform | Official SDK (JS interop easier) |

> ⚠️ `even-better-sdk` is community-maintained. Pin to a specific version and
> test after SDK updates.

---

## 5. evenhub-cli Deep Dive

### Installation

```bash
npm install -g @evenrealities/evenhub-cli
# Verify
evenhub --version
```

### All Commands

```bash
# Development
evenhub qr --http --port 5173        # Generate QR code for sideloading
evenhub qr --https --port 5173       # HTTPS variant (required for microphone)
evenhub serve --port 5173            # Start dev server with QR

# Packaging
evenhub pack <manifest> <dist> -o <output.ehpk>
# Example:
evenhub pack app.json dist -o myapp.ehpk

# Validation
evenhub validate myapp.ehpk          # Check package integrity

# Info
evenhub info myapp.ehpk              # Show package metadata
```

### `app.json` Manifest — Full Reference

```json
{
  "packageId":     "com.yourcompany.appname",
  "name":          "My App",
  "version":       "1.0.0",
  "description":   "Short description shown in Even Hub store",
  "author":        "Your Name <email@example.com>",
  "entryPoint":    "index.html",
  "icon":          "icon.png",
  "permissions":   ["microphone", "imu"],
  "minSdkVersion": "0.0.9",
  "category":      "productivity",
  "tags":          ["weather", "dashboard"],
  "homepage":      "https://yourapp.com",
  "repository":    "https://github.com/you/myapp"
}
```

**`packageId` rules:**
- Reverse-domain format: `com.company.app`
- Lowercase letters, digits, dots only
- No consecutive dots, no leading/trailing dots
- Regex: `/^[a-z][a-z0-9]*(\.[a-z][a-z0-9]*)+$/`

**`permissions` values:**
- `"microphone"` — access to G2 microphone
- `"imu"` — access to IMU/accelerometer data

### Icon Requirements

| Property | Value |
|---|---|
| Format | PNG |
| Size | 512 × 512 px |
| Background | Transparent or solid |
| Safe zone | Keep content within inner 400 × 400 px |

### QR Sideloading Flow

```
1. Run: evenhub qr --http --port 5173
2. Even Hub app on iPhone → Developer → Scan QR
3. App loads in Even Hub WebView
4. Glasses connect automatically via BLE
```

> **HTTPS required for microphone:** Use `--https` flag and accept the
> self-signed certificate on the iPhone when prompted.

### Custom Vite Plugin (even-dev)

The `even-dev` simulator project ships a custom Vite plugin that injects the
simulator bridge. To use it in your own project:

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import { evenHubSimulatorPlugin } from 'even-dev/vite-plugin';  // if installed

export default defineConfig({
  plugins: [
    evenHubSimulatorPlugin({ port: 3001 }),
  ],
  server: {
    host: '0.0.0.0',
    port: 5173,
  },
});
```

---

## 6. Advanced State Management

### Why Standard Approaches Break

Even Hub apps live in a WebView that can be suspended, resumed, or restarted
without warning. `window.localStorage` is **not** bridged to the glasses — only
`EvenAppBridge` storage persists across sessions. React/Vue/Svelte component
state is wiped on every WebView reload.

### Recommended State Architecture

```
┌─────────────────────────────────────────────────────┐
│                   App State Layer                   │
│                                                     │
│  ┌──────────────┐   ┌──────────────────────────┐   │
│  │  Ephemeral   │   │     Persistent Store     │   │
│  │  (in-memory) │   │  (EvenAppBridge storage) │   │
│  │              │   │                          │   │
│  │ • UI state   │   │ • User preferences       │   │
│  │ • Timers     │   │ • Session tokens         │   │
│  │ • Animations │   │ • Last-known values      │   │
│  └──────────────┘   └──────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

### Minimal State Manager (TypeScript)

```typescript
import { EvenAppBridge } from '@evenrealities/even_hub_sdk';

type Listener<T> = (state: T) => void;

class EvenStore<T extends Record<string, unknown>> {
  private state: T;
  private listeners = new Set<Listener<T>>();
  private bridge: EvenAppBridge;
  private persistKeys: (keyof T)[];

  constructor(
    bridge: EvenAppBridge,
    initial: T,
    persistKeys: (keyof T)[] = []
  ) {
    this.bridge = bridge;
    this.state = { ...initial };
    this.persistKeys = persistKeys;
  }

  async hydrate(): Promise<void> {
    for (const key of this.persistKeys) {
      const raw = await this.bridge.getLocalStorage(String(key));
      if (raw !== null && raw !== undefined) {
        try {
          (this.state as Record<string, unknown>)[key as string] =
            JSON.parse(raw);
        } catch {
          (this.state as Record<string, unknown>)[key as string] = raw;
        }
      }
    }
    this.notify();
  }

  get<K extends keyof T>(key: K): T[K] {
    return this.state[key];
  }

  async set<K extends keyof T>(key: K, value: T[K]): Promise<void> {
    (this.state as Record<string, unknown>)[key as string] = value;
    if (this.persistKeys.includes(key)) {
      await this.bridge.setLocalStorage(
        String(key),
        JSON.stringify(value)
      );
    }
    this.notify();
  }

  subscribe(listener: Listener<T>): () => void {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }

  private notify(): void {
    const snapshot = { ...this.state };
    this.listeners.forEach(l => l(snapshot));
  }
}

// Usage
interface AppState {
  username: string;
  theme: 'light' | 'dark';
  lastPage: string;
  counter: number;
}

const store = new EvenStore<AppState>(
  bridge,
  { username: '', theme: 'light', lastPage: 'home', counter: 0 },
  ['username', 'theme', 'lastPage']
);

await store.hydrate();
store.subscribe(state => renderGlassesUI(state));
```

### Handling WebView Restarts

```typescript
async function onBridgeReady(bridge: EvenAppBridge): Promise<void> {
  await store.hydrate();
  const lastPage = store.get('lastPage');
  await navigateTo(lastPage);
}
```

---

## 7. Advanced Multi-Page Architecture

### Page Model Rules

- Max **4 containers per page** (across all types)
- Max **8 text containers** total across all pages
- Exactly **one** `isEventCapture: 1` container per page
- `createStartUpPageContainer` is called **once** at app start

### Router Implementation

```typescript
type PageName = 'home' | 'detail' | 'settings' | 'loading';

interface PageDef {
  name: PageName;
  build: (bridge: EvenAppBridge) => Promise<number[]>;
  destroy?: (bridge: EvenAppBridge, ids: number[]) => Promise<void>;
}

class EvenRouter {
  private bridge: EvenAppBridge;
  private pages = new Map<PageName, PageDef>();
  private currentPage: PageName | null = null;
  private currentIds: number[] = [];

  constructor(bridge: EvenAppBridge) {
    this.bridge = bridge;
  }

  register(page: PageDef): void {
    this.pages.set(page.name, page);
  }

  async navigate(to: PageName): Promise<void> {
    if (to === this.currentPage) return;

    if (this.currentPage) {
      const def = this.pages.get(this.currentPage)!;
      if (def.destroy) {
        await def.destroy(this.bridge, this.currentIds);
      } else {
        for (const id of this.currentIds) {
          await this.bridge.deleteContainer({ containerId: id });
        }
      }
    }

    const def = this.pages.get(to);
    if (!def) throw new Error(`Unknown page: ${to}`);

    this.currentIds = await def.build(this.bridge);
    this.currentPage = to;
  }

  current(): PageName | null {
    return this.currentPage;
  }
}
```

### Wiring Pages to Events

```typescript
const router = new EvenRouter(bridge);

router.register({
  name: 'home',
  build: async (b) => {
    await b.createPage({
      containers: [{
        type: 'list',
        property: {
          containerId: 10,
          containerType: 1,
          x: 0, y: 0, width: 576, height: 288,
          isEventCapture: 1,
          itemList: [
            { subId: 0, text: '▶ Detail' },
            { subId: 1, text: '⚙ Settings' },
          ],
        },
      }],
    });
    return [10];
  },
});

bridge.addContainerEventListener(10, async (event) => {
  if (event.subId === 0) await router.navigate('detail');
  if (event.subId === 1) await router.navigate('settings');
});

await router.navigate('home');
```

---

## 8. R1 Ring Advanced Patterns

### All Ring Events

```typescript
type RingAction =
  | 'tap'
  | 'double_tap'
  | 'long_press'
  | 'swipe_forward'
  | 'swipe_back'
  | 'swipe_up'
  | 'swipe_down';

bridge.addRingEventListener((event: { action: RingAction }) => {
  switch (event.action) {
    case 'tap':           handleTap();        break;
    case 'double_tap':    handleDoubleTap();  break;
    case 'long_press':    handleLongPress();  break;
    case 'swipe_forward': handleNext();       break;
    case 'swipe_back':    handlePrev();       break;
    case 'swipe_up':      handleScrollUp();   break;
    case 'swipe_down':    handleScrollDown(); break;
  }
});
```

### Context-Sensitive Ring Bindings

```typescript
class RingContextManager {
  private contexts = new Map<string, Partial<Record<RingAction, () => void>>>();
  private active = 'default';

  register(
    context: string,
    bindings: Partial<Record<RingAction, () => void>>
  ): void {
    this.contexts.set(context, bindings);
  }

  activate(context: string): void {
    this.active = context;
  }

  handle(action: RingAction): void {
    const ctx      = this.contexts.get(this.active) ?? {};
    const fallback = this.contexts.get('default') ?? {};
    const handler  = ctx[action] ?? fallback[action];
    handler?.();
  }
}

const ring = new RingContextManager();

ring.register('default', {
  tap:           () => router.navigate('home'),
  swipe_forward: () => router.navigate('detail'),
  swipe_back:    () => router.navigate('home'),
});

ring.register('media', {
  tap:           () => togglePlayPause(),
  swipe_forward: () => nextTrack(),
  swipe_back:    () => prevTrack(),
  swipe_up:      () => volumeUp(),
  swipe_down:    () => volumeDown(),
});

bridge.addRingEventListener((e) => ring.handle(e.action as RingAction));
```

---

## 9. Advanced BLE Patterns

### Understanding the Dual BLE Architecture

```
iPhone
  ├── BLE Connection A (Left glass)  ← Commands, text, images
  └── BLE Connection B (Right glass) ← Mirrored display sync
```

Both connections must be active for the display to work. The SDK handles this
transparently, but it has implications:

- **Effective bandwidth is halved** — data must be sent to both glasses
- **Packet loss on either connection** causes display glitches
- **Reconnection** is automatic but takes 1–3 seconds

### BLE Throughput Limits

| Operation | Typical Latency | Max Safe Rate |
|---|---|---|
| Text update | 20–60 ms | ~15 updates/sec |
| Image push (200×100) | 100–300 ms | ~3 frames/sec |
| Container create | 50–150 ms | On-demand only |
| Container delete | 30–80 ms | On-demand only |

### Update Queue (Prevent BLE Saturation)

```typescript
class UpdateQueue {
  private queue: Array<() => Promise<void>> = [];
  private running = false;

  enqueue(task: () => Promise<void>): void {
    this.queue.push(task);
    if (!this.running) this.drain();
  }

  private async drain(): Promise<void> {
    this.running = true;
    while (this.queue.length > 0) {
      const task = this.queue.shift()!;
      try { await task(); }
      catch (e) { console.error('Queue task failed', e); }
    }
    this.running = false;
  }
}

const uiQueue = new UpdateQueue();

// Always enqueue glasses updates
uiQueue.enqueue(() =>
  bridge.updateTextData({ containerId: 1, text: newText })
);
```

### Reconnection Handling

```typescript
bridge.addDeviceStatusListener((status) => {
  if (!status.connected) {
    animator.stop();
    poller.stop();
  }
  if (status.connected) {
    // Re-render current page on reconnect
    renderCurrentPage();
    animator.start(renderFrame);
    poller.start(fetchData, POLL_INTERVAL);
  }
});
```

### Low-Level BLE Protocol (Native Apps Only)

> This section applies to **Path B** (native companion apps using raw BLE),
> not Even Hub WebView apps. See `06_BLE_PROTOCOL.md` for full details.

Key packet structure:
```
[LR][CMD][SEQ][LEN_H][LEN_L][DATA...][CRC]
```
- `LR`: `0x01` = left glass, `0x02` = right glass
- `CMD`: command byte (e.g. `0x4E` = send text, `0x15` = send image)
- `SEQ`: sequence number (0–255, wraps)
- `CRC`: XOR of all preceding bytes

---

## 10. Canvas API & Image Rendering Pipeline

### Display Specifications

| Property | Value |
|---|---|
| Resolution | 576 × 288 px |
| Colour depth | 4-bit greyscale (16 shades) |
| Image container max size | 200 × 100 px |
| Image format for SDK | Base64-encoded PNG |
| Image format for BLE path | 1-bit BMP (576 × 136 px) |

### Basic Canvas → Base64 Pipeline

```typescript
function renderToBase64(
  width: number,
  height: number,
  draw: (ctx: CanvasRenderingContext2D) => void
): string {
  const canvas = document.createElement('canvas');
  canvas.width  = width;
  canvas.height = height;
  const ctx = canvas.getContext('2d')!;

  // Even Hub glasses are greyscale — convert palette
  ctx.filter = 'grayscale(100%)';
  draw(ctx);

  return canvas.toDataURL('image/png').split(',')[1];
}

// Usage
const base64 = renderToBase64(200, 100, (ctx) => {
  ctx.fillStyle = '#000';
  ctx.fillRect(0, 0, 200, 100);
  ctx.fillStyle = '#fff';
  ctx.font = 'bold 24px monospace';
  ctx.fillText('Hello!', 20, 60);
});

await bridge.updateImageData({ containerId: imageContainerId, imageData: base64 });
```

### Floyd-Steinberg Dithering (Better Greyscale Quality)

```typescript
function floydSteinbergDither(
  ctx: CanvasRenderingContext2D,
  width: number,
  height: number
): void {
  const imageData = ctx.getImageData(0, 0, width, height);
  const data = imageData.data;

  for (let y = 0; y < height; y++) {
    for (let x = 0; x < width; x++) {
      const i = (y * width + x) * 4;
      const grey = 0.299 * data[i] + 0.587 * data[i+1] + 0.114 * data[i+2];
      const quantised = Math.round(grey / 17) * 17;
      const error = grey - quantised;

      data[i] = data[i+1] = data[i+2] = quantised;

      const distribute = (dx: number, dy: number, factor: number) => {
        const ni = ((y + dy) * width + (x + dx)) * 4;
        if (ni >= 0 && ni < data.length) {
          data[ni]   = Math.min(255, Math.max(0, data[ni]   + error * factor));
          data[ni+1] = Math.min(255, Math.max(0, data[ni+1] + error * factor));
          data[ni+2] = Math.min(255, Math.max(0, data[ni+2] + error * factor));
        }
      };

      distribute( 1,  0, 7/16);
      distribute(-1,  1, 3/16);
      distribute( 0,  1, 5/16);
      distribute( 1,  1, 1/16);
    }
  }

  ctx.putImageData(imageData, 0, 0);
}
```

### Sprite Sheet Animation

Pre-render all frames to avoid per-frame canvas operations at runtime:

```typescript
class SpriteAnimator {
  private frames: string[] = [];
  private frameIndex = 0;
  private intervalId: ReturnType<typeof setInterval> | null = null;

  prerender(frameCount: number, renderFn: (frame: number) => string): void {
    this.frames = Array.from({ length: frameCount }, (_, i) => renderFn(i));
  }

  async play(bridge: EvenAppBridge, containerId: number, fps = 10): Promise<void> {
    if (this.frames.length === 0) return;
    this.intervalId = setInterval(async () => {
      await bridge.updateImageData({
        containerId,
        imageData: this.frames[this.frameIndex],
      });
      this.frameIndex = (this.frameIndex + 1) % this.frames.length;
    }, 1000 / fps);
  }

  stop(): void {
    if (this.intervalId !== null) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
    this.frameIndex = 0;
  }
}
```

---

## 11. Animation & Motion on the Glasses

### Constraints

| Constraint | Value | Notes |
|---|---|---|
| Practical max FPS | ~12–15 fps | BLE throughput limited |
| Image push | Sequential only | `await` each `updateImageData` |
| Text animation | ~15 updates/sec | Much cheaper than images |
| Canvas size | 576 × 288 px | Fixed |

### Frame Loop Pattern

```typescript
class GlassesAnimator {
  private bridge: EvenAppBridge;
  private containerId: number;
  private running = false;
  private lastFrameTime = 0;
  private targetFps: number;

  constructor(bridge: EvenAppBridge, containerId: number, fps = 12) {
    this.bridge = bridge;
    this.containerId = containerId;
    this.targetFps = fps;
  }

  start(renderFrame: (t: number) => string): void {
    this.running = true;
    const interval = 1000 / this.targetFps;

    const tick = async (now: number) => {
      if (!this.running) return;
      if (now - this.lastFrameTime >= interval) {
        this.lastFrameTime = now;
        const base64 = renderFrame(now);
        await this.bridge.updateImageData({
          containerId: this.containerId,
          imageData: base64,
        });
      }
      requestAnimationFrame(tick);
    };

    requestAnimationFrame(tick);
  }

  stop(): void { this.running = false; }
}
```

### Text-Based Animation (Lower BLE Cost)

For counters, progress bars, and status text — updating a text container is
far cheaper than pushing image frames:

```typescript
async function animateProgressBar(
  bridge: EvenAppBridge,
  containerId: number,
  percent: number
): Promise<void> {
  const filled = Math.round(percent / 5);
  const empty  = 20 - filled;
  const bar    = '█'.repeat(filled) + '░'.repeat(empty);
  await bridge.updateTextData({ containerId, text: `${bar} ${percent}%` });
}
```

---

## 12. Voice AI Pipeline (End-to-End)

### PCM Audio Format

| Parameter | Value |
|---|---|
| Sample rate | 16 000 Hz |
| Bit depth | 16-bit signed PCM |
| Channels | Mono |
| Byte order | Little-endian |
| Delivery | `onMicrophoneData` callback chunks |

### Full Transcription Pipeline

```typescript
class AudioTranscriber {
  private bridge: EvenAppBridge;
  private chunks: ArrayBuffer[] = [];
  private recording = false;

  constructor(bridge: EvenAppBridge) {
    this.bridge = bridge;
  }

  async startRecording(): Promise<void> {
    this.chunks = [];
    this.recording = true;
    await this.bridge.startMicrophone();
    this.bridge.addMicrophoneDataListener((chunk: ArrayBuffer) => {
      if (this.recording) this.chunks.push(chunk);
    });
  }

  async stopAndTranscribe(): Promise<string> {
    this.recording = false;
    await this.bridge.stopMicrophone();

    const pcm = this.mergePCM(this.chunks);
    const wav = this.pcmToWav(pcm, 16000, 1, 16);

    const form = new FormData();
    form.append('file', new Blob([wav], { type: 'audio/wav' }), 'audio.wav');
    form.append('model', 'whisper-1');

    const res = await fetch('https://your-backend.com/transcribe', {
      method: 'POST',
      body: form,
    });
    const { text } = await res.json();
    return text;
  }

  private mergePCM(chunks: ArrayBuffer[]): ArrayBuffer {
    const total  = chunks.reduce((n, c) => n + c.byteLength, 0);
    const merged = new Uint8Array(total);
    let offset   = 0;
    for (const chunk of chunks) {
      merged.set(new Uint8Array(chunk), offset);
      offset += chunk.byteLength;
    }
    return merged.buffer;
  }

  private pcmToWav(
    pcm: ArrayBuffer,
    sampleRate: number,
    channels: number,
    bitDepth: number
  ): ArrayBuffer {
    const dataLen  = pcm.byteLength;
    const buffer   = new ArrayBuffer(44 + dataLen);
    const view     = new DataView(buffer);
    const writeStr = (off: number, s: string) =>
      [...s].forEach((c, i) => view.setUint8(off + i, c.charCodeAt(0)));

    writeStr(0, 'RIFF');
    view.setUint32(4,  36 + dataLen, true);
    writeStr(8, 'WAVE');
    writeStr(12, 'fmt ');
    view.setUint32(16, 16, true);
    view.setUint16(20, 1, true);
    view.setUint16(22, channels, true);
    view.setUint32(24, sampleRate, true);
    view.setUint32(28, sampleRate * channels * bitDepth / 8, true);
    view.setUint16(32, channels * bitDepth / 8, true);
    view.setUint16(34, bitDepth, true);
    writeStr(36, 'data');
    view.setUint32(40, dataLen, true);
    new Uint8Array(buffer, 44).set(new Uint8Array(pcm));
    return buffer;
  }
}
```

### End-to-End AI Voice Assistant

```typescript
async function runVoiceAssistant(bridge: EvenAppBridge): Promise<void> {
  const transcriber = new AudioTranscriber(bridge);

  // Show listening indicator
  await bridge.updateTextData({ containerId: statusId, text: '🎤 Listening...' });
  await transcriber.startRecording();

  // Auto-stop after 5 seconds
  await new Promise(r => setTimeout(r, 5000));

  await bridge.updateTextData({ containerId: statusId, text: '⏳ Processing...' });
  const transcript = await transcriber.stopAndTranscribe();

  // Send to AI backend
  const res = await fetch('https://your-backend.com/ai', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ prompt: transcript }),
  });
  const { answer } = await res.json();

  // Display answer on glasses
  await bridge.updateTextData({ containerId: answerTextId, text: answer });
  await bridge.updateTextData({ containerId: statusId, text: '✅ Done' });
}

// Trigger on R1 ring double-tap
bridge.addRingEventListener((event) => {
  if (event.action === 'double_tap') runVoiceAssistant(bridge);
});
```

### Voice Command Pattern

```typescript
class VoiceCommander {
  private transcriber: AudioTranscriber;
  private commands = new Map<RegExp, () => Promise<void>>();

  constructor(bridge: EvenAppBridge) {
    this.transcriber = new AudioTranscriber(bridge);
  }

  on(pattern: RegExp, handler: () => Promise<void>): void {
    this.commands.set(pattern, handler);
  }

  async listen(): Promise<void> {
    await this.transcriber.startRecording();
    await new Promise(r => setTimeout(r, 5000));
    const text  = await this.transcriber.stopAndTranscribe();
    const lower = text.toLowerCase();

    for (const [pattern, handler] of this.commands) {
      if (pattern.test(lower)) { await handler(); return; }
    }
    console.log('No command matched:', text);
  }
}

const vc = new VoiceCommander(bridge);
vc.on(/next|forward/, () => router.navigate('detail'));
vc.on(/back|home/,    () => router.navigate('home'));
vc.on(/settings/,     () => router.navigate('settings'));
```

---

## 13. Backend Architecture for Even Hub Apps

### Architecture Reminder

```
Phone WebView  ──HTTPS──▶  Your Backend  ──▶  External APIs
     │                    (secrets live here)
    BLE
     │
  G2 Glasses
```

The WebView is the **only** network client. The glasses never talk to the
internet directly.

### Polling Pattern

```typescript
class DataPoller {
  private intervalId: ReturnType<typeof setInterval> | null = null;

  start(fetchFn: () => Promise<void>, intervalMs: number): void {
    fetchFn();
    this.intervalId = setInterval(fetchFn, intervalMs);
  }

  stop(): void {
    if (this.intervalId !== null) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }
}

const poller = new DataPoller();
poller.start(async () => {
  const res  = await fetch('https://api.example.com/weather');
  const data = await res.json();
  await bridge.updateTextData({
    containerId: weatherTextId,
    text: `${data.temp}°C  ${data.condition}`,
  });
}, 5 * 60 * 1000);
```

### WebSocket Pattern (Real-Time)

```typescript
class EvenWebSocket {
  private ws: WebSocket | null = null;
  private reconnectDelay = 1000;
  private maxDelay = 30_000;

  connect(url: string, onMessage: (data: unknown) => void): void {
    const attempt = () => {
      this.ws = new WebSocket(url);
      this.ws.onopen    = () => { this.reconnectDelay = 1000; };
      this.ws.onmessage = (e) => {
        try { onMessage(JSON.parse(e.data)); } catch { onMessage(e.data); }
      };
      this.ws.onclose   = () => {
        setTimeout(attempt, this.reconnectDelay);
        this.reconnectDelay = Math.min(this.reconnectDelay * 2, this.maxDelay);
      };
    };
    attempt();
  }

  send(data: unknown): void {
    if (this.ws?.readyState === WebSocket.OPEN)
      this.ws.send(JSON.stringify(data));
  }
}
```

### Recommended Backend Stack

| Layer | Recommendation | Notes |
|---|---|---|
| Runtime | Node.js / Bun / Deno | Familiar JS ecosystem |
| Framework | Express / Hono / Fastify | Lightweight |
| AI proxy | OpenAI, Anthropic, Gemini | Keys stay server-side |
| STT proxy | Whisper API / Deepgram | Forward PCM/WAV from app |
| Hosting | Railway, Fly.io, Vercel | Low-latency for BLE round-trips |
| Auth | JWT short-lived tokens | See §14 |

---

## 14. Security Considerations

### Never Embed API Keys in the Bundle

```typescript
// ✅ CORRECT — key lives on server
async function askAI(prompt: string): Promise<string> {
  const res = await fetch('https://your-backend.com/api/ask', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ prompt }),
  });
  const { answer } = await res.json();
  return answer;
}

// ❌ WRONG — key exposed in bundle
async function askAIDirect(prompt: string): Promise<string> {
  const res = await fetch('https://api.openai.com/v1/chat/completions', {
    headers: { Authorization: `Bearer sk-EXPOSED_KEY` },
    // ...
  });
  return res.json();
}
```

### Session Token Pattern

```typescript
async function authenticate(bridge: EvenAppBridge): Promise<string> {
  const deviceInfo = await bridge.getDeviceInfo();
  const res = await fetch('https://your-backend.com/auth', {
    method: 'POST',
    body: JSON.stringify({ deviceId: deviceInfo.deviceId }),
  });
  const { token, expiresAt } = await res.json();
  await bridge.setLocalStorage('sessionToken', token);
  await bridge.setLocalStorage('sessionExpiry', String(expiresAt));
  return token;
}

async function getValidToken(bridge: EvenAppBridge): Promise<string> {
  const token  = await bridge.getLocalStorage('sessionToken');
  const expiry = Number(await bridge.getLocalStorage('sessionExpiry'));
  if (token && Date.now() < expiry - 60_000) return token;
  return authenticate(bridge);
}
```

### Security Checklist

- [ ] No API keys in JS bundle
- [ ] HTTPS for all backend calls
- [ ] Short-lived session tokens (< 1 hour)
- [ ] Rate-limit AI endpoints on backend
- [ ] Validate `packageId` on backend to reject unknown apps
- [ ] Sanitise all text before displaying on glasses (no XSS via text containers)
- [ ] Use `evenhub validate` before every release

---

## 15. Offline & Resilience Patterns

### Detecting Connectivity

```typescript
let isOnline = navigator.onLine;
window.addEventListener('online',  () => { isOnline = true;  onReconnect(); });
window.addEventListener('offline', () => { isOnline = false; onDisconnect(); });
```

### Cache-First Data Strategy

```typescript
async function fetchWithCache<T>(
  bridge: EvenAppBridge,
  cacheKey: string,
  fetchFn: () => Promise<T>,
  maxAgeMs = 5 * 60 * 1000
): Promise<T> {
  const cached   = await bridge.getLocalStorage(cacheKey);
  const cachedAt = Number(await bridge.getLocalStorage(`${cacheKey}_ts`));

  if (cached && Date.now() - cachedAt < maxAgeMs) {
    return JSON.parse(cached) as T;
  }

  try {
    const fresh = await fetchFn();
    await bridge.setLocalStorage(cacheKey, JSON.stringify(fresh));
    await bridge.setLocalStorage(`${cacheKey}_ts`, String(Date.now()));
    return fresh;
  } catch {
    if (cached) return JSON.parse(cached) as T;
    throw new Error(`No data available for ${cacheKey}`);
  }
}
```

---

## 16. Testing Strategy

### Unit Testing (Vitest)

```typescript
// bridge.mock.ts
export const mockBridge = {
  updateTextData:  vi.fn().mockResolvedValue(0),
  updateImageData: vi.fn().mockResolvedValue(0),
  getLocalStorage: vi.fn().mockResolvedValue(null),
  setLocalStorage: vi.fn().mockResolvedValue(0),
  addContainerEventListener: vi.fn(),
  addRingEventListener:      vi.fn(),
  addIMUDataListener:        vi.fn(),
  addDeviceStatusListener:   vi.fn(),
};

// store.test.ts
import { describe, it, expect, vi } from 'vitest';
import { EvenStore } from './store';
import { mockBridge } from './bridge.mock';

describe('EvenStore', () => {
  it('persists keys to bridge storage', async () => {
    const store = new EvenStore(
      mockBridge as any,
      { username: '', counter: 0 },
      ['username']
    );

    await store.set('username', 'Alice');

    expect(mockBridge.setLocalStorage).toHaveBeenCalledWith(
      'username',
      '"Alice"'
    );
  });

  it('does not persist non-persist keys', async () => {
    const store = new EvenStore(
      mockBridge as any,
      { username: '', counter: 0 },
      ['username']
    );

    await store.set('counter', 42);

    expect(mockBridge.setLocalStorage).not.toHaveBeenCalledWith(
      'counter',
      expect.anything()
    );
  });
});
```

### Integration Testing with the Simulator

```typescript
// Use even-dev simulator for integration tests
// Start simulator: npx even-dev
// Then run tests against http://localhost:3001

async function integrationTest(): Promise<void> {
  const bridge = await waitForEvenAppBridge();

  // Test container creation
  const result = await bridge.createStartUpPageContainer({
    containers: [{
      type: 'text',
      property: {
        containerId: 999,
        containerType: 2,
        x: 0, y: 0, width: 576, height: 50,
        isEventCapture: 0,
        text: 'Integration Test',
      },
    }],
  });

  console.assert(result === 0, 'Container creation should return 0');
  console.log('✅ Integration test passed');
}
```

### E2E Testing Checklist

```
□ App loads in Even Hub simulator without errors
□ createStartUpPageContainer returns 0
□ Text containers display correct content
□ List container scroll events fire
□ R1 ring events trigger correct handlers
□ IMU data listener receives data
□ Microphone start/stop works
□ Local storage persists across page reload
□ App handles glasses disconnect gracefully
□ App recovers after glasses reconnect
□ Package validates with evenhub validate
□ QR sideload works on physical device
```

### Mocking IMU Data for Tests

```typescript
function simulateIMU(
  bridge: typeof mockBridge,
  sequence: Partial<IMUData>[]
): void {
  const listeners: Array<(data: IMUData) => void> = [];

  bridge.addIMUDataListener.mockImplementation(
    (cb: (data: IMUData) => void) => listeners.push(cb)
  );

  sequence.forEach((partial, i) => {
    setTimeout(() => {
      const data: IMUData = {
        ax: 0, ay: 0, az: 9.8,
        gx: 0, gy: 0, gz: 0,
        timestamp: Date.now(),
        ...partial,
      };
      listeners.forEach(l => l(data));
    }, i * 20);
  });
}

// Simulate a nod gesture
simulateIMU(mockBridge, [
  { gy: 100 }, { gy: -100 }, { gy: 100 }, { gy: -100 },
]);
```

---

## 17. Packaging & Distribution

### Build & Pack Commands

```bash
# 1. Install CLI
npm install -g @evenrealities/evenhub-cli

# 2. Build production bundle
npm run build

# 3. Generate QR for sideloading (dev)
evenhub qr --http --port 5173

# 4. Package for distribution
evenhub pack app.json dist -o myapp.ehpk

# 5. Validate package
evenhub validate myapp.ehpk
```

### Bundle Size Optimisation

```typescript
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
  build: {
    target: 'es2020',
    minify: 'terser',
    terserOptions: {
      compress: { drop_console: true, drop_debugger: true },
    },
    rollupOptions: {
      output: {
        manualChunks: { sdk: ['@evenrealities/even_hub_sdk'] },
      },
    },
    chunkSizeWarningLimit: 200,
  },
});
```

---

## 18. CI/CD for Even Hub Apps

### GitHub Actions Workflow

```yaml
# .github/workflows/build.yml
name: Build & Package

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci
      - run: npx tsc --noEmit
      - run: npx eslint src --ext .ts,.tsx
      - run: npm run build

      - name: Install Even Hub CLI
        run: npm install -g @evenrealities/evenhub-cli

      - name: Package
        run: evenhub pack app.json dist -o app-${{ github.sha }}.ehpk

      - name: Validate
        run: evenhub validate app-${{ github.sha }}.ehpk

      - uses: actions/upload-artifact@v4
        with:
          name: even-hub-package
          path: '*.ehpk'
          retention-days: 30
```

### Versioning Strategy

```bash
# Bump version in package.json and sync to app.json
npm version patch
node -e "
  const pkg = require('./package.json');
  const app = require('./app.json');
  app.version = pkg.version;
  require('fs').writeFileSync('./app.json', JSON.stringify(app, null, 2));
"
```

---

## 19. Performance Profiling

### Measuring Render Time

```typescript
async function measureRenderTime(
  bridge: EvenAppBridge,
  containerId: number,
  content: string
): Promise<number> {
  const start = performance.now();
  await bridge.updateTextData({ containerId, text: content });
  const elapsed = performance.now() - start;
  console.log(`Render took ${elapsed.toFixed(1)}ms`);
  return elapsed;
}
```

### BLE Latency Benchmark

```typescript
async function benchmarkBle(
  bridge: EvenAppBridge,
  containerId: number,
  iterations = 20
): Promise<{ avgMs: number; minMs: number; maxMs: number }> {
  const times: number[] = [];

  for (let i = 0; i < iterations; i++) {
    const start = performance.now();
    await bridge.updateTextData({ containerId, text: `Ping ${i}` });
    times.push(performance.now() - start);
    await new Promise(r => setTimeout(r, 50));
  }

  return {
    avgMs: times.reduce((a, b) => a + b, 0) / times.length,
    minMs: Math.min(...times),
    maxMs: Math.max(...times),
  };
}

// Run in dev mode only
if (import.meta.env.DEV) {
  const stats = await benchmarkBle(bridge, debugContainerId);
  console.log('BLE latency:', stats);
  // Typical: avg ~40ms, min ~20ms, max ~120ms
}
```

### Memory Profiling

```typescript
function logMemory(label: string): void {
  if ('memory' in performance) {
    const m = (performance as any).memory;
    console.log(
      `[${label}] Heap: ${(m.usedJSHeapSize / 1024 / 1024).toFixed(1)} MB` +
      ` / ${(m.totalJSHeapSize / 1024 / 1024).toFixed(1)} MB`
    );
  }
}
```

### FPS Meter

```typescript
class FPSMeter {
  private frames = 0;
  private lastTime = performance.now();
  private fps = 0;

  tick(): number {
    this.frames++;
    const now     = performance.now();
    const elapsed = now - this.lastTime;
    if (elapsed >= 1000) {
      this.fps      = Math.round((this.frames * 1000) / elapsed);
      this.frames   = 0;
      this.lastTime = now;
    }
    return this.fps;
  }
}

const fpsMeter = new FPSMeter();
requestAnimationFrame(function loop() {
  const fps = fpsMeter.tick();
  if (import.meta.env.DEV) document.title = `FPS: ${fps}`;
  requestAnimationFrame(loop);
});
```

### Performance Targets

| Area | Target | How to Measure |
|---|---|---|
| BLE text update | < 60 ms avg | `benchmarkBle()` |
| BLE image push | < 300 ms per frame | `performance.now()` around `updateImageData` |
| JS heap | < 50 MB | `logMemory()` |
| Bundle size (gzip) | < 150 KB | `vite build --report` |
| Frame generation | < 16 ms | `FPSMeter` |
| Bridge init time | < 3 s | Time from page load to `waitForEvenAppBridge` resolve |


---

## 20. Quick-Reference Summary

### Pattern Lookup Table

| Goal | Method / Pattern |
|---|---|
| Initialize SDK | `waitForEvenAppBridge()` → `EvenAppBridge.getInstance()` |
| Send text to glasses | `bridge.sendTextToGlasses(text)` |
| Send image to glasses | `bridge.pushImageData(base64)` — sequential only |
| Create UI container | `bridge.createContainer(pageId, props)` |
| Capture list events | `bridge.setOnListEvent(cb)` |
| Capture text events | `bridge.setOnTextEvent(cb)` |
| Read IMU data | `bridge.setOnIMUData(cb)` |
| Detect head gestures | Accumulate IMU samples, threshold pitch/roll/yaw delta |
| Trigger microphone | `bridge.startMicrophone()` / `bridge.stopMicrophone()` |
| Get audio PCM | `bridge.setOnAudioData(cb)` → convert to WAV |
| Persist data across restarts | `bridge.setLocalStorage(key, value)` |
| Read persisted data | `bridge.getLocalStorage(key)` |
| Detect glasses disconnect | `bridge.setOnDeviceStatus(cb)` → check `status === 0` |
| Detect wearing state | `bridge.setOnWearingStatus(cb)` |
| R1 ring single click | `bridge.setOnRingEvent(cb)` → `action === 'single'` |
| R1 ring double click | `bridge.setOnRingEvent(cb)` → `action === 'double'` |
| R1 ring long press | `bridge.setOnRingEvent(cb)` → `action === 'long'` |
| Navigate pages | Destroy old containers, build new containers |
| Startup page (one-time) | `bridge.createStartUpPageContainer(props)` — call once ever |
| Call native Even AI | `bridge.callEvenApp({ type: 'ai', ... })` |
| Package app | `evenhub-cli build` → `evenhub-cli publish` |
| Test in simulator | `even-dev` simulator or `evenhub-cli dev` |
| Dither image for glasses | Floyd-Steinberg → 4-bit greyscale → 1-bit BMP (BLE path) |
| Prevent BLE saturation | Queue updates, min 100ms between image pushes |
| Backend proxy for API keys | Never embed keys in bundle — proxy all external API calls |

---

### Critical Limits Reminder

```
Canvas:          576 × 288 px  (top-left origin, X→right, Y→down)
Greyscale:       4-bit (16 shades) — SDK path
BLE image:       1-bit BMP, 576 × 136 px — native BLE path
Containers/page: Max 4 total (1 must be isEventCapture: 1)
Text containers: Max 8 per page
Image push:      Sequential only — never concurrent
Startup page:    createStartUpPageContainer() — ONE TIME, ever
BLE interval:    Min ~100ms between transmissions
Storage:         Use bridge.setLocalStorage() NOT window.localStorage
SDK version:     0.0.9 (check npm for updates)
WebView:         iPhone WKWebView — no Node.js, no native fs
Audio format:    PCM 16-bit, 16kHz mono → convert to WAV for APIs
```