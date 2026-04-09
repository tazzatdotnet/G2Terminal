---

# 03 — SDK API Reference

> **Source**: Official `@evenrealities/even_hub_sdk` npm package (v0.0.9), `nickustinov/even-g2-notes`. All facts verified April 2026.
> **Note**: This is the complete, authoritative reference for every method, model, enum, and return type in the SDK. Cross-reference with `04_UI_CONTAINERS.md` for container layout details and `05_EVENTS_AND_INPUT.md` for event handling.

---

## Package Info

| Field | Value |
|-------|-------|
| Package | `@evenrealities/even_hub_sdk` |
| Version | `0.0.9` |
| License | MIT |
| Author | Whiskee (Even Realities) |
| Dependencies | 0 (zero external dependencies) |
| Node.js | `^20.0.0 \|\| >=22.0.0` |
| Published | March 25, 2026 |

```bash
npm install @evenrealities/even_hub_sdk
```

---

## Top-Level Exports

```typescript
import {
  // Bridge
  waitForEvenAppBridge,
  EvenAppBridge,

  // Page containers
  CreateStartUpPageContainer,
  RebuildPageContainer,

  // Container types
  TextContainerProperty,
  ListContainerProperty,
  ListItemContainerProperty,
  ImageContainerProperty,

  // Update types
  TextContainerUpgrade,
  ImageRawDataUpdate,

  // Enums
  EvenAppMethod,
  OsEventTypeList,
  DeviceModel,
  DeviceConnectType,
  ImuReportPace,

  // Constants
  LAUNCH_SOURCE_APP_MENU,
  LAUNCH_SOURCE_GLASSES_MENU,
} from '@evenrealities/even_hub_sdk'
```

---

## Bridge Initialization

### `waitForEvenAppBridge(): Promise<EvenAppBridge>`

**Recommended method.** Waits asynchronously for the native bridge to be injected into the WebView's `window` object by the Flutter Even App. Resolves when the bridge is ready.

```typescript
const bridge = await waitForEvenAppBridge()
```

> ⚠️ Always `await` this at the top of your `main()` function. Never call any bridge methods before this resolves. The bridge is injected by the native app — it is not available synchronously on page load.

### `EvenAppBridge.getInstance(): EvenAppBridge`

Synchronous singleton accessor. **Only use after `waitForEvenAppBridge()` has already resolved** — e.g. in helper functions called later in the app lifecycle.

```typescript
// Safe: called after main() has awaited waitForEvenAppBridge()
function someHelper() {
  const bridge = EvenAppBridge.getInstance()
  // ...
}
```

> ⚠️ Calling `getInstance()` before the bridge is initialized will throw or return `null`. Never use at module load time.

---

## `EvenAppBridge` Methods

### Information Methods

#### `getUserInfo(): Promise<UserInfo>`

Returns the currently logged-in Even Hub user's profile.

```typescript
const user = await bridge.getUserInfo()
console.log(user.uid)     // number
console.log(user.name)    // string
console.log(user.avatar)  // string (URL)
console.log(user.country) // string
```

#### `getDeviceInfo(): Promise<DeviceInfo | null>`

Returns information about the connected glasses (or ring). Returns `null` if no device is connected.

```typescript
const device = await bridge.getDeviceInfo()
if (device) {
  console.log(device.model)           // DeviceModel.G1 | G2 | Ring1
  console.log(device.sn)              // serial number string
  console.log(device.status.connectType)   // DeviceConnectType
  console.log(device.status.batteryLevel)  // 0–100
  console.log(device.status.isWearing)     // boolean
  console.log(device.status.isCharging)    // boolean
  console.log(device.status.isInCase)      // boolean
  console.log(device.isGlasses())     // true for G1/G2
  console.log(device.isRing())        // true for Ring1
}
```

> **Note**: `DeviceInfo.updateStatus(status)` only updates when `status.sn === device.sn`. Mismatched serial numbers are silently ignored.

---

### Storage Methods

#### `setLocalStorage(key: string, value: string): Promise<boolean>`

Persists a key-value pair on the phone side via the Even Hub bridge. Returns `true` on success.

```typescript
const ok = await bridge.setLocalStorage('theme', 'dark')
```

#### `getLocalStorage(key: string): Promise<string>`

Retrieves a previously stored value. Returns an **empty string** (not `null`) if the key does not exist.

```typescript
const theme = await bridge.getLocalStorage('theme')
// Returns '' if not set — check for empty string, not null
```

> ⚠️ **Critical**: Browser `localStorage` does NOT survive app or glasses restarts inside the `.ehpk` WebView. Always use `bridge.setLocalStorage` / `bridge.getLocalStorage` for any data that must persist across sessions.

> ⚠️ There is **no `removeLocalStorage`** method. To "delete" a key, write an empty string and treat empty strings as absent when reading.

**Recommended pattern — in-memory cache wrapper:**

```typescript
const cache = new Map<string, string>()

async function initStorage(bridge: EvenAppBridge, keys: string[]) {
  await Promise.all(keys.map(async (key) => {
    const value = await bridge.getLocalStorage(key)
    if (value) cache.set(key, value)
  }))
}

function getItem(key: string): string | null {
  return cache.get(key) ?? null
}

function setItem(bridge: EvenAppBridge, key: string, value: string): void {
  cache.set(key, value)
  void bridge.setLocalStorage(key, value).catch(() => {})
}
```

Pre-load all keys at startup, read synchronously from cache, write-through to bridge in background.

---

### Event Listener Methods

All listener methods return an **unsubscribe function** — call it to remove the listener.

#### `onLaunchSource(callback: (source: LaunchSource) => void): () => void`

Fires **exactly once** after the WebView finishes loading. Tells you how the app was opened.

```typescript
const unsubscribe = bridge.onLaunchSource((source) => {
  if (source === 'glassesMenu') {
    // User opened from glasses menu — initialize glasses UI here
    initGlassesUI()
  } else if (source === 'appMenu') {
    // User opened from phone app menu
    initPhoneUI()
  }
})
// unsubscribe() to remove listener
```

| Value | Meaning |
|-------|---------|
| `'glassesMenu'` | Opened from the glasses menu |
| `'appMenu'` | Opened from the phone app menu |

> ⚠️ Register this listener **early** — it fires once after loading and will be missed if registered too late.

#### `onDeviceStatusChanged(callback: (status: DeviceStatus) => void): () => void`

Fires whenever the glasses connection state, battery, wearing status, or charging state changes.

```typescript
const unsubscribe = bridge.onDeviceStatusChanged((status) => {
  if (status.isConnected()) {
    console.log('Connected, battery:', status.batteryLevel)
  }
  if (status.isDisconnected()) {
    console.log('Glasses disconnected')
  }
  console.log('Wearing:', status.isWearing)
  console.log('Charging:', status.isCharging)
  console.log('In case:', status.isInCase)
})
```

#### `onEvenHubEvent(callback: (event: EvenHubEvent) => void): () => void`

The main event bus. Receives all runtime events from the glasses: input events, IMU data, audio PCM, and system events.

```typescript
const unsubscribe = bridge.onEvenHubEvent((event) => {
  if (event.listEvent) {
    // List container interaction
    const { currentSelectItemIndex, currentSelectItemName, eventType } = event.listEvent
  } else if (event.textEvent) {
    // Text container scroll boundary or click
    const { containerID, containerName, eventType } = event.textEvent
  } else if (event.sysEvent) {
    // System event: foreground/background, IMU data, abnormal exit
    const { eventType, imuData } = event.sysEvent
    if (imuData) {
      const { x, y, z } = imuData  // IMU_Report_Data, protobuf float
    }
  } else if (event.audioEvent) {
    // Microphone PCM data
    const pcm: Uint8Array = event.audioEvent.audioPcm
  }
})
```

> ⚠️ `CLICK_EVENT = 0` becomes `undefined` in the SDK's `fromJson` normalisation. Always check:
> ```typescript
> const isClick = eventType === OsEventTypeList.CLICK_EVENT || eventType === undefined
> ```

> ⚠️ The simulator sends `sysEvent` for button clicks; real hardware sends `textEvent` or `listEvent`. Handle all three.

---

### Page / UI Methods

#### `createStartUpPageContainer(container: CreateStartUpPageContainer): Promise<StartUpPageCreateResult>`

**Called exactly once per session.** Establishes the initial glasses UI. Must be called before `audioControl`, `imuControl`, or any other UI method.

```typescript
import {
  CreateStartUpPageContainer,
  TextContainerProperty,
} from '@evenrealities/even_hub_sdk'

const result = await bridge.createStartUpPageContainer(
  new CreateStartUpPageContainer({
    containerTotalNum: 1,
    textObject: [
      new TextContainerProperty({
        containerID: 1,
        containerName: 'main',       // max 16 chars
        xPosition: 0,
        yPosition: 0,
        width: 576,
        height: 288,
        borderWidth: 0,
        borderColor: 5,
        paddingLength: 8,
        content: 'Hello G2',         // max 1000 chars at startup
        isEventCapture: 1,           // exactly ONE container must be 1
      }),
    ],
  })
)
// result: 0=success, 1=invalid, 2=oversize, 3=outOfMemory
```

**Return values (`StartUpPageCreateResult`):**

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | Invalid container configuration |
| `2` | Oversize — data too large for BLE transfer |
| `3` | Out of memory on glasses |

**Rules:**
- Call **exactly once** — subsequent calls are silently ignored
- `containerTotalNum` must match the actual total number of containers passed (across all types)
- Exactly **one** container must have `isEventCapture: 1`
- Max `containerTotalNum`: 12 (practical limit: 4 containers per page)
- Max `textObject` items: 8
- Image containers are created as **empty placeholders** — call `updateImageRawData` immediately after to populate them
- Content limit: **1000 characters** per text container at startup

#### `rebuildPageContainer(container: RebuildPageContainer): Promise<boolean>`

Replaces the entire page. Used for **all page changes after the initial startup**. Same parameter shape as `createStartUpPageContainer`.

```typescript
import { RebuildPageContainer, TextContainerProperty } from '@evenrealities/even_hub_sdk'

const success = await bridge.rebuildPageContainer(
  new RebuildPageContainer({
    containerTotalNum: 1,
    textObject: [
      new TextContainerProperty({
        containerID: 1,
        containerName: 'main',
        xPosition: 0,
        yPosition: 0,
        width: 576,
        height: 288,
        borderWidth: 0,
        borderColor: 5,
        paddingLength: 8,
        content: 'Page 2 content',
        isEventCapture: 1,
      }),
    ],
  })
)
```

**Behaviour:**
- Full redraw — all containers destroyed and recreated
- Any internal scroll position or list selection state is lost
- Brief flicker on real hardware
- Content limit: **1000 characters** per text container (same as startup)
- Returns `boolean` (true = success)

#### `textContainerUpgrade(container: TextContainerUpgrade): Promise<boolean>`

Updates text content in an **existing** container without a full page rebuild. Faster and flicker-free on real hardware.

```typescript
import { TextContainerUpgrade } from '@evenrealities/even_hub_sdk'

const success = await bridge.textContainerUpgrade(
  new TextContainerUpgrade({
    containerID: 1,
    containerName: 'main',    // must match existing container
    contentOffset: 0,         // start position in existing content string
    contentLength: 50,        // length of content to replace
    content: 'Updated text',  // max 2000 chars
  })
)
```

**Fields:**

| Field | Type | Notes |
|-------|------|-------|
| `containerID` | number | Must match existing container |
| `containerName` | string | Must match existing container, max 16 chars |
| `contentOffset` | number? | Start position in existing content string |
| `contentLength` | number? | Length of content to replace |
| `content` | string | Replacement text, max **2000 chars** |

> ⚠️ On the simulator, this still causes a visual redraw. On real hardware, it's smooth.

#### `updateImageRawData(data: ImageRawDataUpdate): Promise<ImageRawDataUpdateResult>`

Sends image data to an existing image container. Image containers are empty placeholders until this is called.

```typescript
import { ImageRawDataUpdate } from '@evenrealities/even_hub_sdk'

// Method 1: PNG byte array (recommended)
const pngBytes = Array.from(new Uint8Array(await pngBlob.arrayBuffer()))
const result = await bridge.updateImageRawData(
  new ImageRawDataUpdate({
    containerID: 3,
    containerName: 'logo',
    imageData: pngBytes,
  })
)

// Method 2: base64 PNG string
const base64 = canvas.toDataURL('image/png').replace(/^data:image\/png;base64,/, '')
const result = await bridge.updateImageRawData(
  new ImageRawDataUpdate({
    containerID: 3,
    containerName: 'logo',
    imageData: base64,
  })
)
```

**`imageData` accepted formats:**

| Format | Notes |
|--------|-------|
| `number[]` | Recommended — PNG file bytes or raw greyscale pixel values |
| `string` | Base64-encoded PNG (strip `data:image/png;base64,` prefix) |
| `Uint8Array` | Auto-converted to `number[]` by SDK |
| `ArrayBuffer` | Auto-converted to `number[]` by SDK |

**Return values (`ImageRawDataUpdateResult`):**

| Value | Meaning |
|-------|---------|
| `success` | OK |
| `imageException` | Image processing error |
| `imageSizeInvalid` | Dimensions don't match container or out of range |
| `imageToGray4Failed` | Greyscale conversion failed |
| `sendFailed` | BLE send failed |

> ⚠️ **Never send images concurrently.** Always `await` one `updateImageRawData` call before starting the next. Queue image updates sequentially.

> ⚠️ Do **not** perform 1-bit dithering in your app. The host's 4-bit downsampling is better. Manual Floyd-Steinberg dithering creates noisy green dots.

#### `shutDownPageContainer(exitMode?: number): Promise<boolean>`

Exits the app.

```typescript
await bridge.shutDownPageContainer(0)  // immediate exit
await bridge.shutDownPageContainer(1)  // show exit confirmation to user
```

| `exitMode` | Behaviour |
|-----------|-----------|
| `0` (default) | Immediate exit |
| `1` | Show exit confirmation overlay — user decides |

---

### Audio / Sensor Methods

#### `audioControl(isOpen: boolean): Promise<boolean>`

Opens or closes the microphone.

```typescript
await bridge.audioControl(true)   // open microphone
await bridge.audioControl(false)  // close microphone
```

**Prerequisites:** `createStartUpPageContainer` must have been called successfully first.

**PCM format** (received via `onEvenHubEvent` as `audioEvent.audioPcm`):
- Sample rate: **16 kHz**
- Frame length: **10ms** (dtUs: 10000)
- Bytes per frame: **40**
- Format: **PCM S16LE** (signed 16-bit little-endian, mono)

```typescript
bridge.onEvenHubEvent((event) => {
  if (event.audioEvent) {
    const pcm: Uint8Array = event.audioEvent.audioPcm
    // 40 bytes per frame, 16kHz, S16LE mono
    // Process with Web Audio API, send to speech-to-text API, etc.
  }
})
```

#### `imuControl(isOpen: boolean, reportFrq?: ImuReportPace): Promise<boolean>`

Enables or disables IMU (gyroscope/accelerometer) data reporting.

```typescript
import { ImuReportPace } from '@evenrealities/even_hub_sdk'

await bridge.imuControl(true, ImuReportPace.P500)   // enable at pace 500
await bridge.imuControl(false)                       // disable (pace defaults to P100)
```

**`ImuReportPace` values** — these are protocol-side pace codes, **not physical Hz**:

| Enum | Value |
|------|-------|
| `ImuReportPace.P100` | 100 |
| `ImuReportPace.P200` | 200 |
| `ImuReportPace.P300` | 300 |
| `ImuReportPace.P400` | 400 |
| `ImuReportPace.P500` | 500 |
| `ImuReportPace.P600` | 600 |
| `ImuReportPace.P700` | 700 |
| `ImuReportPace.P800` | 800 |
| `ImuReportPace.P900` | 900 |
| `ImuReportPace.P1000` | 1000 |

**IMU data** arrives via `onEvenHubEvent` as `sysEvent.imuData`:

```typescript
bridge.onEvenHubEvent((event) => {
  const sys = event.sysEvent
  if (!sys?.imuData) return
  if (sys.eventType !== OsEventTypeList.IMU_DATA_REPORT) return

  const { x, y, z } = sys.imuData  // IMU_Report_Data, protobuf float
  console.log('IMU:', x, y, z)
})
```

> ⚠️ On the simulator, `imuData` is always `null`. Test IMU features on real hardware.

**Prerequisites:** `createStartUpPageContainer` must have been called successfully first.

---

### Generic Escape Hatch

#### `callEvenApp(method: EvenAppMethod | string, params?: any): Promise<any>`

Low-level method for calling any native Even App function by name. All typed methods are wrappers around this. Use for undocumented or future methods.

```typescript
import { EvenAppMethod } from '@evenrealities/even_hub_sdk'

// Using the enum
const user = await bridge.callEvenApp(EvenAppMethod.GetUserInfo)

// Using a raw string (for undocumented/future methods)
const result = await bridge.callEvenApp('someNativeMethod', { param: 'value' })
```

**`EvenAppMethod` enum — complete mapping:**

| Enum Value | Native String | Typed Wrapper |
|-----------|---------------|---------------|
| `GetUserInfo` | `'getUserInfo'` | `bridge.getUserInfo()` |
| `GetGlassesInfo` | `'getGlassesInfo'` | `bridge.getDeviceInfo()` |
| `SetLocalStorage` | `'setLocalStorage'` | `bridge.setLocalStorage()` |
| `GetLocalStorage` | `'getLocalStorage'` | `bridge.getLocalStorage()` |
| `CreateStartUpPageContainer` | `'createStartUpPageContainer'` | `bridge.createStartUpPageContainer()` |
| `RebuildPageContainer` | `'rebuildPageContainer'` | `bridge.rebuildPageContainer()` |
| `UpdateImageRawData` | `'updateImageRawData'` | `bridge.updateImageRawData()` |
| `TextContainerUpgrade` | `'textContainerUpgrade'` | `bridge.textContainerUpgrade()` |
| `AudioControl` | `'audioControl'` | `bridge.audioControl()` |
| `ShutDownPageContainer` | `'shutDownPageContainer'` | `bridge.shutDownPageContainer()` |

> **Note**: `GetGlassesInfo` is the underlying native method name. The SDK's public `getDeviceInfo()` wraps it.

---

## Data Models

### `UserInfo`

| Property | Type | Notes |
|----------|------|-------|
| `uid` | `number` | User ID |
| `name` | `string` | Display name |
| `avatar` | `string` | Avatar URL |
| `country` | `string` | Country code |

**Methods:** `toJson()`, `fromJson(json)` (static), `createDefault()` (static)

---

### `DeviceInfo`

| Property | Type | Notes |
|----------|------|-------|
| `model` | `DeviceModel` | Read-only. G1, G2, or Ring1 |
| `sn` | `string` | Read-only. Serial number |
| `status` | `DeviceStatus` | Mutable. Current device state |

**Methods:**
- `updateStatus(status: DeviceStatus)` — updates only if `status.sn === device.sn`
- `isGlasses(): boolean` — true for G1/G2
- `isRing(): boolean` — true for Ring1
- `toJson()`, `fromJson(json)` (static)

---

### `DeviceStatus`

| Property | Type | Notes |
|----------|------|-------|
| `sn` | `string` | Read-only. Serial number |
| `connectType` | `DeviceConnectType` | Connection state |
| `isWearing` | `boolean?` | Whether glasses are being worn |
| `batteryLevel` | `number?` | 0–100 |
| `isCharging` | `boolean?` | Whether charging |
| `isInCase` | `boolean?` | Whether in charging case |

**Methods:**
- `isNone()` — state not yet initialized
- `isConnected()` — fully connected
- `isConnecting()` — connection in progress
- `isDisconnected()` — disconnected
- `isConnectionFailed()` — connection attempt failed
- `toJson()`, `fromJson(json)` (static), `createDefault(sn?)` (static)

---

### `EvenHubEvent`

| Property | Type | Notes |
|----------|------|-------|
| `listEvent` | `List_ItemEvent?` | From list containers |
| `textEvent` | `Text_ItemEvent?` | From text containers |
| `sysEvent` | `Sys_ItemEvent?` | System events, IMU data |
| `audioEvent` | `{ audioPcm: Uint8Array }?` | Microphone PCM |
| `jsonData` | `Record<string, any>?` | Raw payload for debugging |

**`List_ItemEvent` fields:**

| Field | Type | Notes |
|-------|------|-------|
| `containerID` | number | Which list container |
| `containerName` | string | Which list container |
| `currentSelectItemName` | string | Text of selected item |
| `currentSelectItemIndex` | number | 0-based index. May be missing for index 0 — fall back to app state |
| `eventType` | `OsEventTypeList` | Event type. `CLICK_EVENT=0` may arrive as `undefined` |

**`Text_ItemEvent` fields:**

| Field | Type | Notes |
|-------|------|-------|
| `containerID` | number | Which text container |
| `containerName` | string | Which text container |
| `eventType` | `OsEventTypeList` | Event type |

**`Sys_ItemEvent` fields:**

| Field | Type | Notes |
|-------|------|-------|
| `eventType` | `OsEventTypeList` | Event type |
| `imuData` | `IMU_Report_Data?` | Present when IMU is enabled |

**`IMU_Report_Data` fields:**

| Field | Type | Notes |
|-------|------|-------|
| `x` | `number` | Protobuf float |
| `y` | `number` | Protobuf float |
| `z` | `number` | Protobuf float |

---

### Container Property Models

#### `CreateStartUpPageContainer` / `RebuildPageContainer`

Both have identical structure:

| Field | Type | Range | Notes |
|-------|------|-------|-------|
| `containerTotalNum` | number | 1–12 | Must equal total containers across all types |
| `listObject` | `ListContainerProperty[]?` | — | Optional |
| `textObject` | `TextContainerProperty[]?` | max 8 | Optional |
| `imageObject` | `ImageContainerProperty[]?` | — | Optional |
| `widgetId` | number? | — | Usually auto-filled by bridge |

#### `TextContainerProperty`

| Field | Type | Range | Notes |
|-------|------|-------|-------|
| `xPosition` | number | 0–576 | Left edge |
| `yPosition` | number | 0–288 | Top edge |
| `width` | number | 0–576 | Container width |
| `height` | number | 0–288 | Container height |
| `borderWidth` | number | 0–5 | 0 = no border |
| `borderColor` | number | 0–16 | Greyscale level |
| `borderRadius` | number | 0–10 | Rounded corners |
| `paddingLength` | number | 0–32 | Uniform padding |
| `containerID` | number | any | Unique per page |
| `containerName` | string | max 16 chars | Unique per page |
| `content` | string | max 1000 chars (startup/rebuild), max 2000 (upgrade) | Text content |
| `isEventCapture` | number | 0 or 1 | Exactly one per page must be 1 |

#### `ListContainerProperty`

| Field | Type | Range | Notes |
|-------|------|-------|-------|
| `xPosition` | number | 0–576 | Left edge |
| `yPosition` | number | 0–288 | Top edge |
| `width` | number | 0–576 | Container width |
| `height` | number | 0–288 | Container height |
| `borderWidth` | number | 0–5 | 0 = no border |
| `borderColor` | number | 0–15 | Greyscale level (note: 0–15, not 0–16) |
| `borderRadius` | number | 0–10 | Rounded corners |
| `paddingLength` | number | 0–32 | Uniform padding |
| `containerID` | number | any | Unique per page |
| `containerName` | string | max 16 chars | Unique per page |
| `itemContainer` | `ListItemContainerProperty` | — | List items config |
| `isEventCapture` | number | 0 or 1 | Exactly one per page must be 1 |

#### `ListItemContainerProperty`

| Field | Type | Range | Notes |
|-------|------|-------|-------|
| `itemCount` | number | 1–20 | Must match `itemName.length` |
| `itemWidth` | number | pixels | `0` = auto-fill; other = fixed px width |
| `isItemSelectBorderEn` | number | 0 or 1 | Show selection highlight border |
| `itemName` | string[] | max 64 chars each, max 20 items | Item labels |

#### `ImageContainerProperty`

| Field | Type | Range | Notes |
|-------|------|-------|-------|
| `xPosition` | number | 0–576 | Left edge |
| `yPosition` | number | 0–288 | Top edge |
| `width` | number | 20–200 | Hardware enforced |
| `height` | number | 20–100 | Hardware enforced |
| `containerID` | number | any | Unique per page |
| `containerName` | string | max 16 chars | Unique per page |

> ⚠️ Image containers have **no `isEventCapture` property**. To receive events when displaying images, add a full-screen text container with `isEventCapture: 1` behind the image container.

#### `TextContainerUpgrade`

| Field | Type | Notes |
|-------|------|-------|
| `containerID` | number | Must match existing container |
| `containerName` | string | Must match existing container, max 16 chars |
| `contentOffset` | number? | Start position in existing content string |
| `contentLength` | number? | Length of content to replace |
| `content` | string | Replacement text, max **2000 chars** |

#### `ImageRawDataUpdate`

| Field | Type | Notes |
|-------|------|-------|
| `containerID` | number | Must match existing image container |
| `containerName` | string | Must match existing image container |
| `imageData` | `number[] \| string \| Uint8Array \| ArrayBuffer` | Image data in any accepted format |

**Advanced fragmented transfer fields** (rarely needed):

| Field | Type | Notes |
|-------|------|-------|
| `mapSessionId` | any | Session ID for fragmented transfer |
| `mapTotalSize` | any | Total size of fragmented data |
| `compressMode` | any | Compression mode |
| `mapFragmentIndex` | any | Fragment index |
| `mapFragmentPacketSize` | any | Fragment packet size |
| `mapRawData` | any | Raw fragment data |

---

## Enums

### `OsEventTypeList`

| Enum | Value | Source / Meaning |
|------|-------|-----------------|
| `CLICK_EVENT` | `0` | Ring tap, temple tap — **arrives as `undefined` in SDK** |
| `SCROLL_TOP_EVENT` | `1` | Internal scroll reached top boundary |
| `SCROLL_BOTTOM_EVENT` | `2` | Internal scroll reached bottom boundary |
| `DOUBLE_CLICK_EVENT` | `3` | Ring double-tap, temple double-tap |
| `FOREGROUND_ENTER_EVENT` | `4` | App came to foreground |
| `FOREGROUND_EXIT_EVENT` | `5` | App went to background |
| `ABNORMAL_EXIT_EVENT` | `6` | Unexpected disconnect |
| `IMU_DATA_REPORT` | *(value TBC)* | IMU data frame from `imuControl` |

> ⚠️ **`CLICK_EVENT = 0` quirk**: The SDK's `fromJson` normalises `0` to `undefined` in many cases. Always check:
> ```typescript
> const isClick = eventType === OsEventTypeList.CLICK_EVENT || eventType === undefined
> ```

### `DeviceModel`

| Enum | Meaning |
|------|---------|
| `DeviceModel.G1` | Even Realities G1 glasses |
| `DeviceModel.G2` | Even Realities G2 glasses |
| `DeviceModel.Ring1` | R1 Smart Ring |

### `DeviceConnectType`

| Enum | Meaning |
|------|---------|
| `DeviceConnectType.none` | Not initialized |
| `DeviceConnectType.connecting` | Connection in progress |
| `DeviceConnectType.connected` | Fully connected |
| `DeviceConnectType.disconnected` | Disconnected |
| `DeviceConnectType.connectionFailed` | Connection attempt failed |

### `ImuReportPace`

Protocol-side pace codes — **not physical Hz**. Steps of 100 from P100 to P1000.

```typescript
ImuReportPace.P100   // slowest
ImuReportPace.P200
ImuReportPace.P300
ImuReportPace.P400
ImuReportPace.P500   // mid
ImuReportPace.P600
ImuReportPace.P700
ImuReportPace.P800
ImuReportPace.P900
ImuReportPace.P1000  // fastest
```

### `LaunchSource` (type, not enum)

```typescript
type LaunchSource = 'appMenu' | 'glassesMenu'

// Constants also exported:
LAUNCH_SOURCE_APP_MENU     = 'appMenu'
LAUNCH_SOURCE_GLASSES_MENU = 'glassesMenu'
```

---

## Error Codes Reference

### `StartUpPageCreateResult` (from `createStartUpPageContainer`)

| Code | Constant | Meaning |
|------|----------|---------|
| `0` | success | OK |
| `1` | invalid | Invalid container configuration |
| `2` | oversize | Data too large for BLE transfer |
| `3` | outOfMemory | Out of memory on glasses |

### `rebuildPageContainer` return

Returns `boolean`. Internal SDK constants: `APP_REQUEST_REBUILD_PAGE_SUCCESS` / `APP_REQUEST_REBUILD_PAGE_FAILD` *(sic — typo in SDK source)*.

### `textContainerUpgrade` return

Returns `boolean`. Internal SDK constants: `APP_REQUEST_UPGRADE_TEXT_DATA_SUCCESS` / `APP_REQUEST_UPGRADE_TEXT_DATA_FAILED`.

### `ImageRawDataUpdateResult` (from `updateImageRawData`)

| Value | Meaning |
|-------|---------|
| `success` | OK |
| `imageException` | Image processing error |
| `imageSizeInvalid` | Dimensions don't match container or out of range |
| `imageToGray4Failed` | Greyscale conversion failed |
| `sendFailed` | BLE send failed |

### `shutDownPageContainer` return

Returns `boolean`. Internal SDK constants: `APP_REQUEST_UPGRADE_SHUTDOWN_SUCCESS` / `APP_REQUEST_UPGRADE_SHUTDOWN_FAILED`.

---

## SDK JSON Compatibility

The SDK handles multiple key naming conventions from the host via `pickLoose()` — normalises keys by removing underscores and lowercasing:

| Convention | Example |
|-----------|---------|
| camelCase | `containerID` |
| PascalCase | `ContainerID` |
| Proto-style | `Container_ID` |

You only need to use camelCase in your code. The SDK handles the rest.

**Event data is accepted in multiple shapes:**

```typescript
// Shape 1: object with type and jsonData
{ data: { type: 'listEvent', jsonData: { ... } } }

// Shape 2: object with type and data
{ data: { type: 'list_event', data: { ... } } }

// Shape 3: array format
{ data: ['list_event', { ... }] }

// Shape 4: audio event
{ data: { type: 'audioEvent', jsonData: { audioPcm: [...] } } }
```

---

## Critical Gotchas Summary

| Gotcha | Detail |
|--------|--------|
| `CLICK_EVENT = 0` → `undefined` | Always check `eventType === 0 \|\| eventType === undefined` |
| `createStartUpPageContainer` once only | Subsequent calls silently ignored |
| No browser `localStorage` | Use `bridge.setLocalStorage` / `bridge.getLocalStorage` |
| Sequential image sends | Never concurrent — always `await` before next send |
| Image containers are placeholders | Must call `updateImageRawData` after page creation |
| `audioControl` needs startup first | `createStartUpPageContainer` must succeed first |
| `imuControl` needs startup first | Same prerequisite as `audioControl` |
| `containerName` max 16 chars | Silently truncated or causes errors if exceeded |
| `isEventCapture` exactly one | Exactly one container per page must be `1` |
| Simulator vs hardware differences | Image size not enforced, IMU always null, events differ |
| `currentSelectItemIndex` missing for index 0 | Fall back to tracking selection in app state |
| Swipe events fire rapidly | Use 300ms cooldown to prevent duplicate actions |

---