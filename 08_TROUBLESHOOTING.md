# 08 — Troubleshooting, Debugging & Known Issues

> **Purpose:** This file is the definitive reference for diagnosing and fixing problems when building Even Hub apps for the G2 smart glasses. Every issue listed here has been observed in real development or reported by the community.
> **Cross-reference:** `02_SDK_QUICKSTART.md` for initialization, `03_SDK_API_REFERENCE.md` for method signatures, `04_UI_CONTAINERS.md` for container rules, `05_EVENTS_AND_INPUT.md` for event handling, `07_EXAMPLES_AND_PATTERNS.md` for correct patterns.

---

## Quick Diagnosis Index

| Symptom | Jump to |
|---------|---------|
| App never loads on glasses | [App Won't Load](#app-wont-load) |
| `waitForEvenAppBridge` never resolves | [Bridge Never Ready](#bridge-never-ready) |
| Nothing appears on glasses display | [Blank Display](#blank-display) |
| `createStartUpPageContainer` returns non-zero | [Page Creation Errors](#page-creation-errors) |
| Events not firing | [Events Not Firing](#events-not-firing) |
| Scroll events fire but click doesn't | [Click Events Missing](#click-events-missing) |
| Works in simulator but not on glasses | [Simulator vs Hardware Mismatch](#simulator-vs-hardware-mismatch) |
| Images not appearing | [Image Display Issues](#image-display-issues) |
| Images appear then disappear | [Image Flicker / Disappear](#image-flicker--disappear) |
| Audio / microphone not working | [Audio Issues](#audio-issues) |
| App crashes after a few minutes | [Memory & Stability Issues](#memory--stability-issues) |
| Text layout broken / misaligned | [Text Rendering Issues](#text-rendering-issues) |
| App works once then breaks on reopen | [Session / Lifecycle Issues](#session--lifecycle-issues) |
| Local storage not persisting | [Storage Issues](#storage-issues) |
| Network requests failing | [Network Issues](#network-issues) |
| Packaging / `.ehpk` errors | [Packaging Errors](#packaging-errors) |
| `package_id` rejected | [Package ID Errors](#package-id-errors) |

---

## App Won't Load

### Symptom
Scanning the QR code opens the Even App but the app never appears on the glasses, or the Even App shows a loading spinner indefinitely.

### Causes & Fixes

**1. Dev server not reachable from phone**

The phone must reach your dev server over the local network. `localhost` only works on the machine running the server.

```bash
# Wrong — phone can't reach localhost
vite --port 5173

# Correct — bind to all interfaces
vite --host 0.0.0.0 --port 5173
```

Check that your firewall allows inbound connections on port 5173. On macOS:
```bash
# Check if port is open
lsof -i :5173

# Allow through macOS firewall via System Settings → Network → Firewall
```

**2. QR code points to wrong IP**

```bash
# Regenerate QR — evenhub-cli auto-detects your local IP
npx evenhub qr --http --port 5173

# If it picks the wrong interface, specify IP manually
npx evenhub qr --http --host 192.168.1.42 --port 5173
```

**3. Phone and computer on different networks**

Both must be on the same Wi-Fi network. Corporate networks with client isolation will block this. Use a personal hotspot or home router.

**4. HTTPS required for some features**

Some SDK features (microphone, sensors) require HTTPS even in development. Use:
```bash
npx evenhub qr --port 5173  # omit --http to use HTTPS tunnel
```

**5. App crashes immediately on load**

Open the Even App's developer console (if available) or check browser devtools on the phone. A JavaScript error on startup will prevent the app from initialising.

---

## Bridge Never Ready

### Symptom
`await waitForEvenAppBridge()` never resolves. The app hangs at startup.

### Causes & Fixes

**1. Called outside Even Hub WebView**

`waitForEvenAppBridge()` only resolves when running inside the Even Hub WebView on the phone. It will never resolve in a regular browser tab.

```typescript
// Add a timeout for development/testing outside Even Hub
async function waitForBridgeWithTimeout(timeoutMs = 5000): Promise<EvenAppBridge | null> {
  return Promise.race([
    waitForEvenAppBridge(),
    new Promise<null>(resolve => setTimeout(() => resolve(null), timeoutMs)),
  ])
}

const bridge = await waitForBridgeWithTimeout()
if (!bridge) {
  console.warn('Not running in Even Hub — showing fallback UI')
  showFallbackUI()
  return
}
```

**2. Script loaded before DOM is ready**

Ensure your script is loaded as a module or deferred:
```html
<!-- Correct -->
<script type="module" src="/src/main.ts"></script>

<!-- Also correct -->
<script defer src="/src/main.ts"></script>

<!-- Wrong — may run before Even Hub injects the bridge -->
<script src="/src/main.ts"></script>
```

**3. Multiple calls to `waitForEvenAppBridge`**

Only call it once. Multiple concurrent calls can cause race conditions.

```typescript
// Wrong
const bridge1 = await waitForEvenAppBridge()
const bridge2 = await waitForEvenAppBridge()  // ← don't do this

// Correct — call once, store reference
let bridge: EvenAppBridge | null = null

async function getBridge(): Promise<EvenAppBridge> {
  if (!bridge) bridge = await waitForEvenAppBridge()
  return bridge
}
```

**4. SDK version mismatch**

Ensure you're using the correct import path for your SDK version:
```typescript
// SDK 0.0.9 — correct import
import { waitForEvenAppBridge } from '@evenrealities/even_hub_sdk'

// Check installed version
// package.json → dependencies → @evenrealities/even_hub_sdk
```

---

## Blank Display

### Symptom
`createStartUpPageContainer` returns 0 (success) but nothing appears on the glasses display.

### Causes & Fixes

**1. `onLaunchSource` not registered before bridge resolves**

The `onLaunchSource` callback fires immediately after registration if the app was already launched. If you register it too late, you miss the event.

```typescript
// Wrong — async gap between bridge ready and onLaunchSource registration
const bridge = await waitForEvenAppBridge()
await someOtherAsyncOperation()  // ← launch source fires here, you miss it
bridge.onLaunchSource(async (source) => { ... })

// Correct — register onLaunchSource immediately
const bridge = await waitForEvenAppBridge()
bridge.onLaunchSource(async (source) => {  // ← register synchronously
  if (source === 'glassesMenu') {
    await initGlassesUI(bridge)
  }
})
```

**2. `createStartUpPageContainer` called outside `onLaunchSource`**

The glasses display is only active when the app is launched from the glasses menu. Calling `createStartUpPageContainer` when `source === 'phoneMenu'` has no effect.

```typescript
bridge.onLaunchSource(async (source) => {
  if (source !== 'glassesMenu') {
    // Phone UI only — do NOT call createStartUpPageContainer here
    showPhoneUI()
    return
  }
  // Only here is it valid to call createStartUpPageContainer
  await bridge.createStartUpPageContainer(...)
})
```

**3. Content is empty string or whitespace only**

The firmware may not render a container with empty content. Always provide at least a single space:

```typescript
// Risky — may not render
content: ''

// Safe
content: ' '

// Better — always have real content
content: 'Loading...'
```

**4. Container dimensions are zero**

```typescript
// Wrong — zero width/height
new TextContainerProperty({
  width: 0,
  height: 0,
  ...
})

// Correct
new TextContainerProperty({
  width: 576,
  height: 288,
  ...
})
```

**5. All containers have `isEventCapture: 0`**

At least one container must have `isEventCapture: 1`. Without it, the firmware may not activate the display layer.

---

## Page Creation Errors

### `createStartUpPageContainer` Return Codes

| Code | Meaning | Fix |
|------|---------|-----|
| `0` | Success | — |
| `1` | Invalid config | Check all required fields are present and valid |
| `2` | Oversize | Reduce container dimensions or content length |
| `3` | Out of memory | Reduce number of containers, content size, or image data |

### Code 1: Invalid Config — Common Causes

```typescript
// Missing required field
new TextContainerProperty({
  containerID: 1,
  // containerName missing ← causes code 1
  content: 'Hello',
  xPosition: 0,
  yPosition: 0,
  width: 576,
  height: 288,
  borderWidth: 0,
  borderColor: 0,
  paddingLength: 8,
  isEventCapture: 1,
})

// containerTotalNum doesn't match actual container count
new CreateStartUpPageContainer({
  containerTotalNum: 2,  // ← says 2
  textObject: [          // ← but only 1 text container
    new TextContainerProperty({ ... })
  ],
  // no imageObject ← total is 1, not 2 → code 1
})

// Correct: containerTotalNum = sum of all containers across all types
new CreateStartUpPageContainer({
  containerTotalNum: 3,  // 2 text + 1 image = 3
  textObject: [
    new TextContainerProperty({ containerID: 1, ... }),
    new TextContainerProperty({ containerID: 2, ... }),
  ],
  imageObject: [
    new ImageContainerProperty({ containerID: 3, ... }),
  ],
})
```

### Code 2: Oversize — Common Causes

```typescript
// Container extends beyond canvas
new TextContainerProperty({
  xPosition: 400,
  width: 300,  // 400 + 300 = 700 > 576 ← oversize
  ...
})

// Fix: ensure xPosition + width ≤ 576, yPosition + height ≤ 288
new TextContainerProperty({
  xPosition: 400,
  width: 176,  // 400 + 176 = 576 ✓
  ...
})

// Image container too large
new ImageContainerProperty({
  width: 300,   // ← max is 200
  height: 150,  // ← max is 100
  ...
})
```

### Code 3: Out of Memory — Common Causes

- Too many containers (max 4 per page)
- Content string too long (keep under ~2000 chars per container)
- Image data too large (ensure image is ≤ 200×100px before base64 encoding)
- Multiple rapid calls to `rebuildPageContainer` without waiting for completion

```typescript
// Guard against oversized content
function safeContent(content: string, maxChars = 1500): string {
  if (content.length <= maxChars) return content
  return content.slice(0, maxChars - 3) + '...'
}
```

---

## Events Not Firing

### Symptom
`bridge.onEvenHubEvent` callback is never called, or only fires for some event types.

### Causes & Fixes

**1. No container with `isEventCapture: 1`**

Events are only delivered if at least one container has `isEventCapture: 1`. This is the most common cause.

```typescript
// Wrong — no event capture
new TextContainerProperty({
  ...
  isEventCapture: 0,  // ← events will never fire
})

// Correct
new TextContainerProperty({
  ...
  isEventCapture: 1,  // ← exactly one container must have this
})
```

**2. `onEvenHubEvent` registered before `createStartUpPageContainer`**

Register the event handler AFTER the page is created:

```typescript
// Correct order
const result = await bridge.createStartUpPageContainer(config)
if (result !== 0) return

// Register AFTER page creation
bridge.onEvenHubEvent((event) => {
  // handle events
})
```

**3. Multiple `onEvenHubEvent` registrations**

Each call to `onEvenHubEvent` adds a new listener. If you call it multiple times (e.g. on re-render), you'll get duplicate events. Use a flag:

```typescript
let eventHandlerRegistered = false

async function initPage(bridge: EvenAppBridge) {
  await bridge.createStartUpPageContainer(config)

  if (!eventHandlerRegistered) {
    eventHandlerRegistered = true
    bridge.onEvenHubEvent(handleEvent)
  }
}
```

**4. Wrong container type for expected events**

- **List containers** → generate `listEvent`
- **Text containers** → generate `textEvent`
- **System events** (lifecycle, IMU) → always `sysEvent`

If you have a list container with `isEventCapture: 1` and check `event.textEvent`, you'll never see events. Check `event.listEvent` instead.

**5. Glasses not in foreground**

Events are only delivered when the app is in the foreground (glasses display is active). If the user navigates away, events stop until `FOREGROUND_ENTER_EVENT` fires.

---

## Click Events Missing

### Symptom
Scroll events fire correctly but single-tap (click) events never arrive.

### Cause

For text containers, the click event arrives with `eventType === undefined` (not `OsEventTypeList.CLICK_EVENT`). This is a known SDK quirk.

```typescript
// Wrong — misses real hardware click events
if (event.textEvent?.eventType === OsEventTypeList.CLICK_EVENT) {
  handleClick()
}

// Correct — handles both undefined (real hardware) and CLICK_EVENT (simulator)
const t = event.textEvent?.eventType ?? event.sysEvent?.eventType
const isClick = t === OsEventTypeList.CLICK_EVENT || t === undefined

if (isClick) {
  handleClick()
}
```

> ⚠️ **This is the single most common bug in Even Hub apps.** Always use `t === undefined` as the click condition for text container events on real hardware.

### List Container Click Events

For list containers, click events DO arrive with `eventType === OsEventTypeList.CLICK_EVENT` (not undefined). The undefined-click quirk only applies to text containers.

```typescript
bridge.onEvenHubEvent((event) => {
  if (event.listEvent) {
    // List: click arrives as CLICK_EVENT (not undefined)
    const isClick = event.listEvent.eventType === OsEventTypeList.CLICK_EVENT
                 || event.listEvent.eventType === undefined  // safe to include both
    if (isClick) {
      const index = event.listEvent.currentSelectItemIndex ?? 0
      handleListSelect(index)
    }
  }

  if (event.textEvent) {
    // Text: click arrives as undefined on real hardware
    const t = event.textEvent.eventType
    const isClick = t === OsEventTypeList.CLICK_EVENT || t === undefined
    if (isClick) handleClick()
  }
})
```

---

## Simulator vs Hardware Mismatch

### Symptom
App works perfectly in the BxNxM simulator but behaves differently on real glasses.

### Known Differences

| Behaviour | Simulator | Real Hardware | Fix |
|-----------|-----------|---------------|-----|
| Click event type | `CLICK_EVENT` | `undefined` (text containers) | Check both |
| Event object | `sysEvent` | `textEvent` or `listEvent` | Check all three |
| `createStartUpPageContainer` | Can call multiple times | One-time only | Use flag |
| Image render speed | Instant | 200–500ms | Add queue |
| Font metrics | Browser font | LVGL firmware font | Test on device |
| Column alignment | Accurate | Slightly off | Use device for final check |
| Audio | Simulated | Real LC3 pipeline | Test on device |
| Battery info | Mock (100%) | Real value | Handle both |
| Scroll sensitivity | 1 event per gesture | May fire multiple | Add cooldown |

### Universal Event Handler (Works on Both)

```typescript
function getEventType(event: EvenHubEvent): OsEventTypeList | undefined {
  return event.textEvent?.eventType
      ?? event.listEvent?.eventType
      ?? event.sysEvent?.eventType
}

function isClickEvent(event: EvenHubEvent): boolean {
  // Skip lifecycle events
  const t = getEventType(event)
  if (t === OsEventTypeList.FOREGROUND_ENTER_EVENT) return false
  if (t === OsEventTypeList.FOREGROUND_EXIT_EVENT) return false

  // Click = CLICK_EVENT (simulator/list) OR undefined (real hardware text containers)
  return t === OsEventTypeList.CLICK_EVENT || t === undefined
}

function isScrollDown(event: EvenHubEvent): boolean {
  return getEventType(event) === OsEventTypeList.SCROLL_BOTTOM_EVENT
}

function isScrollUp(event: EvenHubEvent): boolean {
  return getEventType(event) === OsEventTypeList.SCROLL_TOP_EVENT
}

function isDoubleClick(event: EvenHubEvent): boolean {
  return getEventType(event) === OsEventTypeList.DOUBLE_CLICK_EVENT
}
```

---

## Image Display Issues

### Symptom
Image container is created but no image appears, or image appears corrupted.

### Causes & Fixes

**1. `updateImageRawData` called before page creation completes**

Always await `createStartUpPageContainer` before sending image data:

```typescript
const result = await bridge.createStartUpPageContainer(config)
if (result !== 0) return

// Only now send image data
await bridge.updateImageRawData(new ImageRawDataUpdate({
  containerID: 2,
  containerName: 'canvas',
  imageData: base64,
}))
```

**2. Base64 string includes data URL prefix**

```typescript
// Wrong — includes prefix
const base64 = canvas.toDataURL('image/png')
// → "data:image/png;base64,iVBORw0KGgo..."

// Correct — strip prefix
const base64 = canvas.toDataURL('image/png').replace(/^data:image\/png;base64,/, '')
// → "iVBORw0KGgo..."
```

**3. Image dimensions exceed 200×100px**

The image container has a hard maximum of 200×100 pixels. Sending a larger image will either fail silently or corrupt the display.

```typescript
// Always enforce max dimensions
const MAX_W = 200
const MAX_H = 100

const canvas = document.createElement('canvas')
canvas.width = Math.min(desiredWidth, MAX_W)
canvas.height = Math.min(desiredHeight, MAX_H)
```

**4. Concurrent image updates**

Sending two `updateImageRawData` calls simultaneously corrupts the BLE transfer. Always queue:

```typescript
// Wrong — concurrent
bridge.updateImageRawData(update1)  // ← not awaited
bridge.updateImageRawData(update2)  // ← fires before update1 completes

// Correct — sequential
await bridge.updateImageRawData(update1)
await bridge.updateImageRawData(update2)

// Best — use ImageQueue class (see 07_EXAMPLES_AND_PATTERNS.md)
```

**5. `containerID` / `containerName` mismatch**

The `containerID` and `containerName` in `updateImageRawData` must exactly match those used in `createStartUpPageContainer`:

```typescript
// In createStartUpPageContainer:
new ImageContainerProperty({
  containerID: 3,
  containerName: 'myImage',
  ...
})

// In updateImageRawData — must match exactly:
new ImageRawDataUpdate({
  containerID: 3,       // ← must match
  containerName: 'myImage',  // ← must match
  imageData: base64,
})
```

---

## Image Flicker / Disappear

### Symptom
Image appears briefly then disappears, or flickers when updated.

### Causes & Fixes

**1. `rebuildPageContainer` called while image update is in flight**

Rebuilding the page cancels any pending image transfer. Wait for image updates to complete before rebuilding:

```typescript
// Wrong
bridge.updateImageRawData(imageUpdate)  // ← not awaited
bridge.rebuildPageContainer(newConfig)  // ← cancels image transfer

// Correct
await bridge.updateImageRawData(imageUpdate)
await bridge.rebuildPageContainer(newConfig)
```

**2. Game loop sending frames faster than BLE can handle**

At 10 FPS, each frame takes ~100ms. BLE image transfer takes ~200–500ms. If you send frames faster than they can be transmitted, the queue grows unboundedly and frames appear to flicker or lag.

```typescript
// Adaptive frame rate — only send next frame after previous completes
class AdaptiveGameLoop {
  private running = false
  private lastFrameTime = 0

  async start(bridge: EvenAppBridge, state: GameState) {
    this.running = true
    while (this.running) {
      const frameStart = Date.now()
      updatePhysics(state)
      const frame = renderFrame(state)
      await bridge.updateImageRawData(new ImageRawDataUpdate({
        containerID: 2,
        containerName: 'canvas',
        imageData: frame,
      }))
      // Enforce minimum frame time (don't go faster than hardware can handle)
      const elapsed = Date.now() - frameStart
      const minFrameTime = 250  // ~4 FPS max for image apps
      if (elapsed < minFrameTime) {
        await new Promise(r => setTimeout(r, minFrameTime - elapsed))
      }
    }
  }

  stop() { this.running = false }
}
```

---

## Audio Issues

### Symptom
Microphone doesn't activate, PCM data never arrives, or audio quality is poor.

### Causes & Fixes

**1. `audioControl(true)` called before page creation**

The audio pipeline requires an active display session. Call `audioControl` only after `createStartUpPageContainer` succeeds.

**2. PCM data arrives in `audioEvent` not `textEvent`**

```typescript
bridge.onEvenHubEvent((event) => {
  // Wrong — checking wrong event type
  if (event.textEvent?.audioPcm) { ... }

  // Correct
  if (event.audioEvent) {
    const pcmFrame = event.audioEvent.audioPcm  // Uint8Array, 40 bytes
    accumulatePcm(pcmFrame)
  }
})
```

**3. Not stopping microphone on background**

Leaving the microphone active when the app goes to background causes battery drain and may prevent other apps from using audio.

```typescript
bridge.onEvenHubEvent((event) => {
  if (event.sysEvent?.eventType === OsEventTypeList.FOREGROUND_EXIT_EVENT) {
    bridge.audioControl(false).catch(console.error)
    isListening = false
  }
})
```

**4. PCM format mismatch for STT APIs**

The G2 delivers PCM in a specific format. Ensure your WAV header matches:

| Property | Value |
|----------|-------|
| Format | PCM (uncompressed) |
| Sample rate | 16,000 Hz |
| Bit depth | 16-bit signed |
| Byte order | Little-endian |
| Channels | 1 (mono) |
| Frame size | 40 bytes (10ms per frame) |

```typescript
// Correct WAV header for G2 PCM
function pcmToWav(pcm: Uint8Array): ArrayBuffer {
  const sampleRate = 16000
  const numChannels = 1
  const bitsPerSample = 16
  // ... (see Recipe 5 in 07_EXAMPLES_AND_PATTERNS.md for full implementation)
}
```

**5. Audio cuts out after ~30 seconds**

This is expected behaviour — implement auto-stop at 30 seconds:

```typescript
const MAX_RECORDING_FRAMES = 3000  // 30s × 100 frames/s

if (pcmChunks.length >= MAX_RECORDING_FRAMES) {
  await bridge.audioControl(false)
  isListening = false
  // process audio...
}
```

---

## Memory & Stability Issues

### Symptom
App works initially but slows down, crashes, or stops responding after a few minutes.

### Causes & Fixes

**1. Event listener accumulation**

Every call to `bridge.onEvenHubEvent` adds a new listener. If called on every render, listeners accumulate:

```typescript
// Wrong — adds new listener on every render
async function render(bridge: EvenAppBridge) {
  await bridge.textContainerUpgrade({ ... })
  bridge.onEvenHubEvent(handleEvent)  // ← adds ANOTHER listener each time
}

// Correct — register once
let listenerRegistered = false

async function initOnce(bridge: EvenAppBridge) {
  await bridge.createStartUpPageContainer(config)
  if (!listenerRegistered) {
    listenerRegistered = true
    bridge.onEvenHubEvent(handleEvent)
  }
}
```

**2. `setInterval` not cleared on background**

```typescript
let refreshInterval: ReturnType<typeof setInterval> | null = null

bridge.onEvenHubEvent((event) => {
  if (event.sysEvent?.eventType === OsEventTypeList.FOREGROUND_ENTER_EVENT) {
    // Start interval
    refreshInterval = setInterval(() => refresh(bridge), 30_000)
  }
  if (event.sysEvent?.eventType === OsEventTypeList.FOREGROUND_EXIT_EVENT) {
    // ALWAYS clear interval on background
    if (refreshInterval) {
      clearInterval(refreshInterval)
      refreshInterval = null
    }
  }
})
```

**3. PCM buffer growing unboundedly**

If you accumulate PCM frames without a size limit, the buffer will grow until the WebView runs out of memory:

```typescript
const MAX_PCM_FRAMES = 3000  // 30 seconds max
const pcmChunks: Uint8Array[] = []

bridge.onEvenHubEvent((event) => {
  if (event.audioEvent && isListening) {
    if (pcmChunks.length < MAX_PCM_FRAMES) {
      pcmChunks.push(event.audioEvent.audioPcm)
    } else {
      // Auto-stop at limit
      bridge.audioControl(false)
      isListening = false
      processAudio(pcmChunks)
    }
  }
})
```

**4. Image queue growing unboundedly**

If your game loop produces frames faster than BLE can transmit them, the queue grows forever. Cap the queue:

```typescript
class BoundedImageQueue {
  private queue: Array<() => Promise<void>> = []
  private running = false
  private readonly MAX_QUEUE = 3  // drop old frames if queue is full

  enqueue(task: () => Promise<void>) {
    if (this.queue.length >= this.MAX_QUEUE) {
      this.queue.shift()  // drop oldest frame
    }
    this.queue.push(task)
    this.drain()
  }

  private async drain() {
    if (this.running || this.queue.length === 0) return
    this.running = true
    while (this.queue.length > 0) {
      const task = this.queue.shift()!
      try { await task() } catch (e) { console.error(e) }
    }
    this.running = false
  }
}
```

---

## Text Rendering Issues

### Symptom
Text appears cut off, columns don't align, or characters render as boxes.

### Causes & Fixes

**1. Font is not monospaced**

The G2 firmware uses an LVGL proportional font. Characters have different widths. `padEnd()` and `padStart()` will not produce accurate column alignment.

```typescript
// This looks aligned in your editor but NOT on the glasses
const row1 = 'Temperature'.padEnd(16) + '18°C'
const row2 = 'Humidity'.padEnd(16) + '65%'

// Better: use tab-like separators and test on real hardware
const row1 = 'Temperature    18°C'
const row2 = 'Humidity       65%'

// Best: use separate text containers for each column
// (see Pattern 5 in 07_EXAMPLES_AND_PATTERNS.md)
```

**2. Content too long — text overflows container**

The firmware clips text at the container boundary. There is no automatic scrolling within a container.

```typescript
// Estimate max chars based on container size
// Rule of thumb: ~40 chars per line at default font size, ~8 lines per 288px height
// Actual values depend on font size and content

function estimateMaxChars(containerWidth: number, containerHeight: number): number {
  const charsPerLine = Math.floor(containerWidth / 14)  // ~14px per char (approximate)
  const linesPerContainer = Math.floor(containerHeight / 36)  // ~36px per line (approximate)
  return charsPerLine * linesPerContainer
}
```

**3. Unsupported Unicode characters render as boxes**

Test all special characters in the simulator before shipping. If a character renders as `□` or `?`, it's not in the firmware font.

Safe ranges:
- Basic Latin (U+0020–U+007E) — always safe
- Latin-1 Supplement (U+00A0–U+00FF) — generally safe
- Box Drawing (U+2500–U+257F) — safe
- Block Elements (U+2580–U+259F) — safe
- Geometric Shapes (U+25A0–U+25FF) — mostly safe
- Arrows (U+2190–U+21FF) — mostly safe
- Mathematical Operators (U+2200–U+22FF) — mostly safe
- Emoji (U+1F000+) — **unreliable, avoid**

**4. Newlines not rendering**

Use `\n` for line breaks. `<br>` and HTML tags are not supported — this is not a browser renderer.

```typescript
// Wrong
content: 'Line 1<br>Line 2'

// Correct
content: 'Line 1\nLine 2'
```

**5. `paddingLength` eating into usable area**

`paddingLength` applies to all four sides. A `paddingLength: 8` on a 576×288 container gives you 560×272 of usable space.

```typescript
// Account for padding in content width calculations
const PADDING = 8
const usableWidth = containerWidth - (PADDING * 2)   // 576 - 16 = 560
const usableHeight = containerHeight - (PADDING * 2) // 288 - 16 = 272
```

---

## Session / Lifecycle Issues

### Symptom
App works on first launch but breaks when reopened, or state is lost between sessions.

### Causes & Fixes

**1. `createStartUpPageContainer` called again on reopen**

This is the most critical lifecycle bug. `createStartUpPageContainer` can only be called ONCE per session. Calling it again after the app is backgrounded and foregrounded will fail or produce undefined behaviour.

```typescript
// Wrong — calls createStartUpPageContainer every time app comes to foreground
bridge.onEvenHubEvent(async (event) => {
  if (event.sysEvent?.eventType === OsEventTypeList.FOREGROUND_ENTER_EVENT) {
    await bridge.createStartUpPageContainer(config)  // ← WRONG on second call
  }
})

// Correct — createStartUpPageContainer only on first launch
let pageCreated = false

bridge.onLaunchSource(async (source) => {
  if (source !== 'glassesMenu') return
  if (!pageCreated) {
    const result = await bridge.createStartUpPageContainer(config)
    if (result === 0) pageCreated = true
  }
})

// On foreground re-enter: refresh content only
bridge.onEvenHubEvent(async (event) => {
  if (event.sysEvent?.eventType === OsEventTypeList.FOREGROUND_ENTER_EVENT) {
    // Use textContainerUpgrade or rebuildPageContainer — NOT createStartUpPageContainer
    await bridge.textContainerUpgrade({ containerID: 1, containerName: 'main', content: getContent() })
  }
})
```

**2. State not persisted across sessions**

JavaScript variables are reset when the WebView is reloaded. Use `bridge.setLocalStorage` to persist state:

```typescript
// On app start — restore state
const savedState = await bridge.getLocalStorage({ key: 'appState' })
if (savedState?.value) {
  Object.assign(state, JSON.parse(savedState.value))
}

// On state change — save state
async function saveState() {
  await bridge.setLocalStorage({
    key: 'appState',
    value: JSON.stringify(state),
  })
}
```

**3. `onLaunchSource` fires multiple times**

`onLaunchSource` can fire more than once in a session (e.g. if the user switches apps and returns). Guard against duplicate initialisation:

```typescript
let initialized = false

bridge.onLaunchSource(async (source) => {
  if (source !== 'glassesMenu') return
  if (initialized) {
    // Already initialized — just refresh content
    await refreshContent(bridge)
    return
  }
  initialized = true
  await initGlassesUI(bridge)
})
```

---

## Storage Issues

### Symptom
Data saved with `setLocalStorage` is not available on next launch, or `getLocalStorage` always returns null.

### Causes & Fixes

**1. Using `window.localStorage` instead of `bridge.setLocalStorage`**

```typescript
// Wrong — not persisted across sessions on G2
window.localStorage.setItem('key', 'value')
const value = window.localStorage.getItem('key')

// Correct
await bridge.setLocalStorage({ key: 'key', value: 'value' })
const result = await bridge.getLocalStorage({ key: 'key' })
const value = result?.value
```

**2. Not awaiting storage operations**

```typescript
// Wrong — fire and forget
bridge.setLocalStorage({ key: 'prefs', value: JSON.stringify(prefs) })

// Correct
await bridge.setLocalStorage({ key: 'prefs', value: JSON.stringify(prefs) })
```

**3. Storing non-string values**

`setLocalStorage` only accepts strings. Serialize objects:

```typescript
// Wrong
await bridge.setLocalStorage({ key: 'count', value: 42 as any })

// Correct
await bridge.setLocalStorage({ key: 'count', value: String(42) })
await bridge.setLocalStorage({ key: 'prefs', value: JSON.stringify({ theme: 'dark' }) })
```

**4. Key not found returns null/undefined**

Always handle the missing-key case:

```typescript
const result = await bridge.getLocalStorage({ key: 'prefs' })
// result may be null if key doesn't exist
const prefs = result?.value ? JSON.parse(result.value) : DEFAULT_PREFS
```

---

## Network Issues

### Symptom
`fetch` calls fail with CORS errors, network errors, or are blocked silently.

### Causes & Fixes

**1. Hostname not in `app.json` permissions**

The Even Hub WebView sandbox blocks requests to hosts not listed in `permissions.network`:

```json
{
  "permissions": {
    "network": [
      "api.openai.com",
      "api.open-meteo.com"
    ]
  }
}
```

> ⚠️ This only applies to packaged `.ehpk` apps. During development (QR code / dev server), network permissions are not enforced.

**2. CORS errors from third-party APIs**

Some APIs don't allow browser-side requests. Solutions:
- Use a backend proxy (your own server)
- Use APIs that explicitly support CORS (Open-Meteo, HackerNews Firebase API, etc.)
- Use `callEvenApp` to delegate to the native Even App for certain operations

**3. HTTP (not HTTPS) blocked**

The WebView enforces HTTPS for all external requests in production. Use HTTPS endpoints only.

```typescript
// Wrong
fetch('http://api.example.com/data')

// Correct
fetch('https://api.example.com/data')
```

**4. API keys exposed in client bundle**

Never put API keys in your Even Hub app bundle — it's a WebView and the source is readable.

```typescript
// Wrong — key visible in bundle
const API_KEY = 'sk-abc123...'
fetch('https://api.openai.com/v1/chat/completions', {
  headers: { 'Authorization': `Bearer ${API_KEY}` }
})

// Correct — proxy through your own backend
fetch('https://your-backend.com/api/chat', {
  method: 'POST',
  body: JSON.stringify({ message: userInput }),
})
// Your backend holds the API key and forwards to OpenAI
```

---

## Packaging Errors

### Symptom
`evenhub pack` fails, or the `.ehpk` file is rejected by the Even App.

### Causes & Fixes

**1. `dist/` folder doesn't exist**

Run `npm run build` before packaging:

```bash
npm run build   # creates dist/
npx evenhub pack app.json dist -o myapp.ehpk
```

**2. `entrypoint` file not found in dist**

```json
{
  "entrypoint": "index.html"  // must exist at dist/index.html after build
}
```

Check your Vite config — the default output is `dist/index.html`. If you've changed `build.outDir` or `build.rollupOptions`, update `app.json` accordingly.

**3. `app.json` not valid JSON**

```bash
# Validate JSON
node -e "JSON.parse(require('fs').readFileSync('app.json', 'utf8'))"
# No output = valid. Error = invalid JSON.
```

**4. `.ehpk` file too large**

Even Hub has a size limit for app packages (exact limit not officially documented — keep under 5MB). Optimise your bundle:

```typescript
// vite.config.ts — enable minification and tree-shaking
export default defineConfig({
  build: {
    minify: 'esbuild',
    target: 'es2020',
    rollupOptions: {
      output: {
        manualChunks: undefined,  // single bundle for Even Hub
      }
    }
  }
})
```

---

## Package ID Errors

### Symptom
Even Hub rejects the app with a package ID error.

### Valid Package ID Rules

```
com.yourname.myapp     ✅ Valid
com.yourname.myapp2    ✅ Valid (numbers allowed after first char)
io.github.yourname.app ✅ Valid (4 segments OK)

com.YourName.MyApp     ❌ Uppercase not allowed
com.your-name.my-app   ❌ Hyphens not allowed
com.your_name.my_app   ❌ Underscores not allowed
com.1yourname.app      ❌ Segment starts with number
com.myapp              ❌ Only 2 segments (minimum 3)
myapp                  ❌ No dots (minimum 3 segments)
```

### Regex for Validation

```typescript
function isValidPackageId(id: string): boolean {
  // Each segment: starts with lowercase letter, followed by lowercase letters or digits
  const segmentPattern = /^[a-z][a-z0-9]*$/
  const segments = id.split('.')
  return segments.length >= 3 && segments.every(s => segmentPattern.test(s))
}

// Test
isValidPackageId('com.yourname.myapp')   // true
isValidPackageId('com.your-name.myapp')  // false
isValidPackageId('com.YourName.myapp')   // false
```

---

## Debugging Techniques

### 1. Console Logging

`console.log` output is visible in:
- The Even App's developer console (if enabled)
- Your browser devtools when testing via QR code on a phone connected to your computer

```typescript
// Structured logging helper
function log(category: string, message: string, data?: any) {
  const timestamp = new Date().toISOString().slice(11, 23)
  if (data !== undefined) {
    console.log(`[${timestamp}] [${category}] ${message}`, data)
  } else {
    console.log(`[${timestamp}] [${category}] ${message}`)
  }
}

// Usage
log('BRIDGE', 'Initialized')
log('EVENT', 'Received', event)
log('RENDER', 'Page created', result)
```

### 2. On-Glasses Debug Display

Display debug info directly on the glasses:

```typescript
async function debugDisplay(bridge: EvenAppBridge, lines: string[]) {
  const content = lines.slice(0, 8).join('\n')  // max ~8 lines visible
  await bridge.textContainerUpgrade({
    containerID: 1,
    containerName: 'main',
    content,
  })
}

// Usage
await debugDisplay(bridge, [
  `State: ${JSON.stringify(state).slice(0, 50)}`,
  `Page: ${pageIndex}`,
  `Events: ${eventCount}`,
  `Last event: ${lastEventType}`,
  `Memory: ${(performance as any).memory?.usedJSHeapSize ?? 'N/A'}`,
])
```

### 3. Event Logger

Log all events to understand what the firmware is sending:

```typescript
bridge.onEvenHubEvent((event) => {
  const summary = {
    hasText: !!event.textEvent,
    textType: event.textEvent?.eventType,
    hasList: !!event.listEvent,
    listType: event.listEvent?.eventType,
    listIndex: event.listEvent?.currentSelectItemIndex,
    hasSys: !!event.sysEvent,
    sysType: event.sysEvent?.eventType,
    hasAudio: !!event.audioEvent,
    audioBytes: event.audioEvent?.audioPcm?.length,
  }
  console.log('EVENT:', JSON.stringify(summary))
})
```

### 4. Bridge Inspector

Inspect the bridge object to understand available methods:

```typescript
const bridge = await waitForEvenAppBridge()
console.log('Bridge methods:', Object.getOwnPropertyNames(Object.getPrototypeOf(bridge)))
console.log('Bridge instance:', bridge)
```

### 5. Simulator for Rapid Iteration

Use the BxNxM simulator for fast iteration — no need to scan QR codes or wait for BLE:

```bash
# Terminal 1: simulator
cd even-dev && npm start

# Terminal 2: your app
cd my-app && npm run dev

# Open http://localhost:3000 in browser
# Your app loads in the simulator immediately
```

### 6. Remote Debugging via Chrome DevTools

When testing on a real phone via QR code:

1. Connect phone to computer via USB
2. Enable USB debugging on Android (Settings → Developer Options)
3. Open `chrome://inspect` in Chrome on your computer
4. Find the Even App WebView and click "inspect"
5. Full Chrome DevTools available — console, network, sources, memory

> ⚠️ iOS does not support remote WebView debugging without a jailbreak. Use Android for debugging, then test on iOS.

---

## Known SDK Bugs & Limitations (as of SDK 0.0.9)

| Issue | Workaround |
|-------|-----------|
| Text container click event arrives with `eventType === undefined` | Check `t === undefined \|\| t === CLICK_EVENT` |
| `createStartUpPageContainer` is one-time only | Use flag, never call twice |
| Image container max 200×100px (not full canvas) | Use text containers for full-canvas layouts |
| No `removeEventListener` — listeners accumulate | Use registration flag |
| `window.localStorage` not persisted | Use `bridge.getLocalStorage` / `bridge.setLocalStorage` |
| Concurrent image updates corrupt BLE transfer | Use sequential queue |
| List container `isEventCapture` has no effect on scroll | Use text container for scroll capture |
| IMU data only available when wearing detection is active | Check `isWearing` before using IMU |
| `audioControl` must be called after page creation | Always init page first |
| No official way to detect SDK version at runtime | Check `package.json` at build time |
| `rebuildPageContainer` resets scroll position | Expected — save scroll state manually |
| Font is proportional — `padEnd` alignment is approximate | Use separate containers for columns |
| Emoji characters unreliable | Use Unicode block/geometric chars instead |
| No official error codes for `rebuildPageContainer` | Assume 0 = success, non-zero = error |

---

## Emergency Recovery Patterns

### App is stuck / unresponsive

```typescript
// Nuclear option: force a full page rebuild
async function emergencyReset(bridge: EvenAppBridge) {
  try {
    await bridge.audioControl(false)
  } catch {}

  await bridge.rebuildPageContainer(
    new RebuildPageContainer({
      containerTotalNum: 1,
      textObject: [
        new TextContainerProperty({
          containerID: 1,
          containerName: 'recovery',
          content: 'Recovering...\n\nDouble-tap to restart',
          xPosition: 0,
          yPosition: 0,
          width: 576,
          height: 288,
          borderWidth: 0,
          borderColor: 0,
          paddingLength: 16,
          isEventCapture: 1,
        }),
      ],
    })
  )
}
```

### Wrap all renders in try/catch

```typescript
async function safeRebuild(bridge: EvenAppBridge, config: any): Promise<boolean> {
  try {
    const result = await bridge.rebuildPageContainer(new RebuildPageContainer(config))
    return result === 0
  } catch (err) {
    console.error('rebuildPageContainer threw:', err)
    return false
  }
}
```

### Global error handler

```typescript
window.addEventListener('unhandledrejection', async (event) => {
  console.error('Unhandled promise rejection:', event.reason)
  if (bridge) {
    await bridge.textContainerUpgrade({
      containerID: 1,
      containerName: 'main',
      content: `Error: ${String(event.reason).slice(0, 100)}\n\nDouble-tap to reload`,
    }).catch(() => {})
  }
})

window.addEventListener('error', (event) => {
  console.error('Global error:', event.message)
})
```

---

*End of `08_TROUBLESHOOTING.md`*

*Next file: `09_ADVANCED_TOPICS.md` — IMU/sensor data, callEvenApp, Kotlin Multiplatform, even-better-sdk community library, evenhub-cli deep dive, and advanced BLE patterns.*

`08_TROUBLESHOOTING.md` is complete. Here's what's inside:

### File Structure

| Section | What it covers |
|---------|---------------|
| **Quick Diagnosis Index** | Symptom → section jump table for fast lookup |
| **App Won't Load** | Dev server binding, QR IP, network isolation, HTTPS |
| **Bridge Never Ready** | Timeout pattern, script loading order, duplicate calls |
| **Blank Display** | `onLaunchSource` timing, wrong launch source, empty content, zero dimensions |
| **Page Creation Errors** | All 4 return codes with causes and fixes, `containerTotalNum` mismatch |
| **Events Not Firing** | Missing `isEventCapture`, registration order, listener accumulation, wrong container type |
| **Click Events Missing** | The `eventType === undefined` quirk — the #1 most common bug |
| **Simulator vs Hardware** | Full difference table + universal event handler that works on both |
| **Image Display Issues** | Timing, base64 prefix, 200×100 limit, concurrent updates, ID mismatch |
| **Image Flicker** | Rebuild race condition, adaptive frame rate pattern |
| **Audio Issues** | PCM format table, auto-stop, background cleanup |
| **Memory & Stability** | Listener accumulation, interval leaks, unbounded PCM/image queues |
| **Text Rendering** | Non-monospace font, overflow, unsafe Unicode ranges, newlines |
| **Session / Lifecycle** | `createStartUpPageContainer` one-time rule, state persistence, duplicate `onLaunchSource` |
| **Storage Issues** | `window.localStorage` vs bridge storage, async, type coercion |
| **Network Issues** | Permissions sandbox, CORS, HTTPS enforcement, API key security |
| **Packaging Errors** | Build order, entrypoint, JSON validation, bundle size |
| **Package ID Errors** | Rules table + validation regex |
| **Debugging Techniques** | Console logging, on-glasses debug display, event logger, Chrome DevTools remote debugging |
| **Known SDK Bugs** | Complete table of all known quirks with workarounds |
| **Emergency Recovery** | Nuclear reset pattern, global error handler |