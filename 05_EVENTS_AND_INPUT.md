# 05 — Events, Input & Device APIs

> **Source**: `@evenrealities/even_hub_sdk` v0.0.9, `nickustinov/even-g2-notes` (input-events.md, device-apis.md, page-lifecycle.md, error-codes.md). All facts verified April 2026.
> **Cross-reference**: `03_SDK_API_REFERENCE.md` for method signatures, `04_UI_CONTAINERS.md` for container setup, `06_BLE_PROTOCOL.md` for low-level hardware details.

---

## Event System Overview

All runtime events from the glasses flow through a **single callback** registered with `bridge.onEvenHubEvent()`. This includes:

- User input (tap, double-tap, swipe)
- List and text container interactions
- System lifecycle events (foreground/background)
- IMU (gyroscope/accelerometer) data frames
- Microphone PCM audio data

```typescript
const unsubscribe = bridge.onEvenHubEvent((event: EvenHubEvent) => {
  if (event.listEvent)  { /* list container interaction */ }
  if (event.textEvent)  { /* text container interaction */ }
  if (event.sysEvent)   { /* system event or IMU data */ }
  if (event.audioEvent) { /* microphone PCM frame */ }
})

// Call unsubscribe() to remove the listener
```

---

## `EvenHubEvent` Structure

```typescript
type EvenHubEvent = {
  listEvent?:  List_ItemEvent        // from list containers
  textEvent?:  Text_ItemEvent        // from text containers
  sysEvent?:   Sys_ItemEvent         // system-level events + IMU
  audioEvent?: { audioPcm: Uint8Array } // microphone PCM
  jsonData?:   Record<string, any>   // raw payload for debugging
}
```

Only **one** of `listEvent`, `textEvent`, `sysEvent`, or `audioEvent` is populated per callback invocation.

---

## Event Types (`OsEventTypeList`)

| Enum | Value | Source / Meaning |
|------|-------|-----------------|
| `CLICK_EVENT` | `0` | Ring tap, temple tap — **arrives as `undefined` in SDK** |
| `SCROLL_TOP_EVENT` | `1` | Internal scroll reached top boundary |
| `SCROLL_BOTTOM_EVENT` | `2` | Internal scroll reached bottom boundary |
| `DOUBLE_CLICK_EVENT` | `3` | Ring double-tap, temple double-tap |
| `FOREGROUND_ENTER_EVENT` | `4` | App came to foreground |
| `FOREGROUND_EXIT_EVENT` | `5` | App went to background |
| `ABNORMAL_EXIT_EVENT` | `6` | Unexpected disconnect / abnormal exit |
| `IMU_DATA_REPORT` | *(value TBC)* | IMU data frame from `imuControl` |

> ⚠️ **Critical quirk — `CLICK_EVENT = 0` becomes `undefined`:** The SDK's `fromJson` normalises `0` to `undefined` in many cases. **Always** check both:
> ```typescript
> const isClick = eventType === OsEventTypeList.CLICK_EVENT || eventType === undefined
> ```

---

## Event Routing: Which Event Type Fires?

The container with `isEventCapture: 1` determines which event type you receive:

| Active container type | Scroll gesture result | Click result |
|----------------------|----------------------|--------------|
| **List** (`isEventCapture: 1`) | Firmware moves selection highlight natively; boundary events arrive as `listEvent` | `listEvent` with `currentSelectItemIndex` |
| **Text** (`isEventCapture: 1`) | Firmware scrolls text internally; boundary events arrive as `textEvent` | `textEvent` |
| **Simulator only** | `sysEvent` for button clicks | `sysEvent` |

> ⚠️ **Simulator vs real device:** The simulator sends `sysEvent` for button clicks. Real hardware sends `textEvent` or `listEvent` depending on the active container. Always handle all three event sources in your event handler.

---

## List Container Events (`listEvent`)

Fires when the user interacts with a list container that has `isEventCapture: 1`.

### `List_ItemEvent` Fields

| Field | Type | Notes |
|-------|------|-------|
| `containerID` | number | Which list container fired the event |
| `containerName` | string | Which list container fired the event |
| `currentSelectItemName` | string | Text label of the currently selected item |
| `currentSelectItemIndex` | number? | 0-based index of selected item. **May be `undefined` for index 0** |
| `eventType` | `OsEventTypeList` | Event type. `CLICK_EVENT=0` may arrive as `undefined` |

### List Event Handling Pattern

```typescript
let selectedIndex = 0  // track in app state as fallback

bridge.onEvenHubEvent((event) => {
  const { listEvent } = event
  if (!listEvent) return

  const { eventType, currentSelectItemIndex, currentSelectItemName } = listEvent

  // CLICK_EVENT = 0 may arrive as undefined
  const isClick = eventType === OsEventTypeList.CLICK_EVENT || eventType === undefined
  const isDoubleClick = eventType === OsEventTypeList.DOUBLE_CLICK_EVENT

  // currentSelectItemIndex may be missing for index 0 — fall back to app state
  const index = currentSelectItemIndex ?? selectedIndex

  if (isClick) {
    console.log(`Selected: ${currentSelectItemName} (index ${index})`)
    handleSelection(index)
  } else if (isDoubleClick) {
    handleDoubleClick(index)
  } else if (eventType === OsEventTypeList.SCROLL_TOP_EVENT) {
    // User reached top boundary of list
    selectedIndex = 0
  } else if (eventType === OsEventTypeList.SCROLL_BOTTOM_EVENT) {
    // User reached bottom boundary of list
    selectedIndex = listItems.length - 1
  }
})
```

> ⚠️ **List scroll is handled natively by firmware.** You do NOT need to call `rebuildPageContainer` on every scroll gesture — the firmware moves the selection highlight automatically. You only need to respond to click/double-click events and boundary events.

> ⚠️ **`currentSelectItemIndex` missing for index 0.** The SDK normalises `0` to `undefined`. Always maintain `selectedIndex` in your app state and use it as a fallback.

---

## Text Container Events (`textEvent`)

Fires when the user interacts with a text container that has `isEventCapture: 1`.

### `Text_ItemEvent` Fields

| Field | Type | Notes |
|-------|------|-------|
| `containerID` | number | Which text container fired the event |
| `containerName` | string | Which text container fired the event |
| `eventType` | `OsEventTypeList` | Event type. `CLICK_EVENT=0` may arrive as `undefined` |

### Text Event Handling Pattern

```typescript
// Swipe throttle — prevent duplicate actions from rapid scroll events
let lastScrollTime = 0
const SCROLL_COOLDOWN_MS = 300

bridge.onEvenHubEvent((event) => {
  const { textEvent } = event
  if (!textEvent) return

  const { eventType } = textEvent
  const isClick = eventType === OsEventTypeList.CLICK_EVENT || eventType === undefined
  const isDoubleClick = eventType === OsEventTypeList.DOUBLE_CLICK_EVENT
  const isScrollDown = eventType === OsEventTypeList.SCROLL_BOTTOM_EVENT
  const isScrollUp = eventType === OsEventTypeList.SCROLL_TOP_EVENT

  if (isClick) {
    handleClick()
    return
  }

  if (isDoubleClick) {
    handleDoubleClick()
    return
  }

  // Throttle scroll events
  const now = Date.now()
  if (now - lastScrollTime < SCROLL_COOLDOWN_MS) return
  lastScrollTime = now

  if (isScrollDown) {
    handleScrollDown()
  } else if (isScrollUp) {
    handleScrollUp()
  }
})
```

> ⚠️ **Scroll events fire rapidly.** Always use a cooldown (300ms recommended) to prevent duplicate actions from a single swipe gesture.

---

## System Events (`sysEvent`)

Fires for lifecycle events, IMU data, and (on the simulator) button clicks.

### `Sys_ItemEvent` Fields

| Field | Type | Notes |
|-------|------|-------|
| `eventType` | `OsEventTypeList` | Event type |
| `imuData` | `IMU_Report_Data?` | Present only when IMU is enabled and reporting |

### System Event Handling Pattern

```typescript
bridge.onEvenHubEvent((event) => {
  const { sysEvent } = event
  if (!sysEvent) return

  switch (sysEvent.eventType) {
    case OsEventTypeList.FOREGROUND_ENTER_EVENT:
      // App came to foreground — resume, refresh data
      onAppForeground()
      break

    case OsEventTypeList.FOREGROUND_EXIT_EVENT:
      // App went to background — pause, save state
      onAppBackground()
      break

    case OsEventTypeList.ABNORMAL_EXIT_EVENT:
      // Unexpected disconnect — clean up
      onAbnormalExit()
      break

    case OsEventTypeList.IMU_DATA_REPORT:
      if (sysEvent.imuData) {
        const { x, y, z } = sysEvent.imuData
        handleImuData(x, y, z)
      }
      break

    // Simulator only: button clicks arrive as sysEvent
    case OsEventTypeList.CLICK_EVENT:
    case undefined:
      handleClick()
      break

    case OsEventTypeList.DOUBLE_CLICK_EVENT:
      handleDoubleClick()
      break

    case OsEventTypeList.SCROLL_TOP_EVENT:
      handleScrollUp()
      break

    case OsEventTypeList.SCROLL_BOTTOM_EVENT:
      handleScrollDown()
      break
  }
})
```

---

## Lifecycle Events: Foreground / Background

The glasses app can be sent to the background (e.g. user opens another app on the glasses) and brought back to the foreground.

```typescript
bridge.onEvenHubEvent((event) => {
  const { sysEvent } = event
  if (!sysEvent) return

  if (sysEvent.eventType === OsEventTypeList.FOREGROUND_ENTER_EVENT) {
    // App is now visible on glasses
    // Good time to: refresh data, resume timers, re-render UI
    resumeApp()
  }

  if (sysEvent.eventType === OsEventTypeList.FOREGROUND_EXIT_EVENT) {
    // App is now hidden
    // Good time to: pause timers, save state, stop audio
    pauseApp()
  }

  if (sysEvent.eventType === OsEventTypeList.ABNORMAL_EXIT_EVENT) {
    // Unexpected disconnect — glasses disconnected mid-session
    // Clean up resources, reset state
    handleDisconnect()
  }
})
```

**Best practices for foreground/background:**

```typescript
let refreshInterval: ReturnType<typeof setInterval> | null = null

function resumeApp() {
  // Refresh data immediately on foreground
  fetchAndRender()
  // Resume periodic refresh
  refreshInterval = setInterval(fetchAndRender, 30_000)
}

function pauseApp() {
  // Stop periodic refresh to save battery
  if (refreshInterval) {
    clearInterval(refreshInterval)
    refreshInterval = null
  }
  // Close microphone if open
  bridge?.audioControl(false)
}
```

---

## Launch Source (`onLaunchSource`)

Fires **exactly once** after the WebView finishes loading. Tells you how the app was opened.

```typescript
const unsubscribe = bridge.onLaunchSource((source) => {
  if (source === 'glassesMenu') {
    // Opened from glasses menu — initialize glasses UI immediately
    initGlassesUI()
  } else if (source === 'appMenu') {
    // Opened from phone app menu — show phone settings UI
    initPhoneUI()
  }
})
```

| Value | Constant | Meaning |
|-------|----------|---------|
| `'glassesMenu'` | `LAUNCH_SOURCE_GLASSES_MENU` | User opened from glasses menu |
| `'appMenu'` | `LAUNCH_SOURCE_APP_MENU` | User opened from phone app menu |

> ⚠️ Register this listener **early** — it fires once after loading and will be missed if registered too late. Register it before `await waitForEvenAppBridge()` resolves if possible, or immediately after.

**Full launch pattern:**

```typescript
async function main() {
  const bridge = await waitForEvenAppBridge()

  // Register launch source listener immediately
  bridge.onLaunchSource(async (source) => {
    if (source === 'glassesMenu') {
      // Initialize glasses display
      const result = await bridge.createStartUpPageContainer(
        new CreateStartUpPageContainer({ ... })
      )
      if (result !== 0) {
        console.error('Page creation failed:', result)
        return
      }
      // Start app logic
      startApp(bridge)
    } else {
      // Phone UI — no glasses initialization needed
      showPhoneSettings()
    }
  })
}

main()
```

---

## Device Status Events (`onDeviceStatusChanged`)

Fires whenever the glasses connection state, battery, wearing status, or charging state changes.

```typescript
const unsubscribe = bridge.onDeviceStatusChanged((status: DeviceStatus) => {
  console.log('Connect type:', status.connectType)
  console.log('Battery:', status.batteryLevel, '%')
  console.log('Wearing:', status.isWearing)
  console.log('Charging:', status.isCharging)
  console.log('In case:', status.isInCase)

  if (status.isConnected()) {
    onGlassesConnected()
  } else if (status.isDisconnected()) {
    onGlassesDisconnected()
  } else if (status.isConnectionFailed()) {
    onConnectionFailed()
  }
})
```

### `DeviceStatus` Helper Methods

| Method | Returns | Meaning |
|--------|---------|---------|
| `isNone()` | boolean | State not yet initialized |
| `isConnected()` | boolean | Fully connected |
| `isConnecting()` | boolean | Connection in progress |
| `isDisconnected()` | boolean | Disconnected |
| `isConnectionFailed()` | boolean | Connection attempt failed |

### `DeviceConnectType` Enum

| Enum | Meaning |
|------|---------|
| `DeviceConnectType.none` | Not initialized |
| `DeviceConnectType.connecting` | Connection in progress |
| `DeviceConnectType.connected` | Fully connected |
| `DeviceConnectType.disconnected` | Disconnected |
| `DeviceConnectType.connectionFailed` | Connection attempt failed |

**Wearing detection pattern:**

```typescript
bridge.onDeviceStatusChanged((status) => {
  if (status.isWearing === true) {
    // Glasses are on the user's face — show content
    showActiveContent()
  } else if (status.isWearing === false) {
    // Glasses removed — optionally pause/dim
    showIdleContent()
  }
})
```

---

## Audio / Microphone

### Opening and Closing the Microphone

```typescript
// Prerequisites: createStartUpPageContainer must have been called first
await bridge.audioControl(true)   // open microphone
await bridge.audioControl(false)  // close microphone
```

> ⚠️ `audioControl` requires `createStartUpPageContainer` to have been called successfully first. Calling it before page initialization will fail silently or throw.

### PCM Audio Format

| Property | Value |
|----------|-------|
| Sample rate | **16 kHz** |
| Frame length | **10ms** (dtUs: 10000) |
| Bytes per frame | **40** |
| Format | **PCM S16LE** (signed 16-bit little-endian, mono) |
| Delivery | `event.audioEvent.audioPcm` as `Uint8Array` |

### Receiving PCM Data

```typescript
await bridge.audioControl(true)

bridge.onEvenHubEvent((event) => {
  if (!event.audioEvent) return
  const pcm: Uint8Array = event.audioEvent.audioPcm
  // 40 bytes per frame, 16kHz, S16LE mono
  processPcmFrame(pcm)
})
```

### Streaming to Speech-to-Text API

```typescript
// Accumulate frames and send to STT API in chunks
const PCM_FRAMES_PER_CHUNK = 50  // 50 frames × 10ms = 500ms chunks
const pcmBuffer: Uint8Array[] = []

bridge.onEvenHubEvent(async (event) => {
  if (!event.audioEvent) return

  pcmBuffer.push(event.audioEvent.audioPcm)

  if (pcmBuffer.length >= PCM_FRAMES_PER_CHUNK) {
    // Merge frames into single buffer
    const totalBytes = pcmBuffer.reduce((sum, f) => sum + f.length, 0)
    const merged = new Uint8Array(totalBytes)
    let offset = 0
    for (const frame of pcmBuffer) {
      merged.set(frame, offset)
      offset += frame.length
    }
    pcmBuffer.length = 0

    // Send to STT API (e.g. Whisper, Google STT, Deepgram)
    const transcript = await sendToSttApi(merged)
    if (transcript) {
      await bridge.textContainerUpgrade(new TextContainerUpgrade({
        containerID: 1,
        containerName: 'transcript',
        content: transcript,
      }))
    }
  }
})
```

### Voice Assistant Pattern (Full Flow)

```typescript
let isListening = false

async function toggleListening() {
  if (isListening) {
    await bridge.audioControl(false)
    isListening = false
    await updateStatus('Tap to speak')
  } else {
    await bridge.audioControl(true)
    isListening = true
    await updateStatus('Listening...')
  }
}

// Tap to toggle listening
bridge.onEvenHubEvent(async (event) => {
  const eventType = event.textEvent?.eventType ?? event.sysEvent?.eventType
  const isClick = eventType === OsEventTypeList.CLICK_EVENT || eventType === undefined

  if (isClick) {
    await toggleListening()
  }

  if (event.audioEvent && isListening) {
    accumulatePcm(event.audioEvent.audioPcm)
  }
})
```

---

## IMU (Gyroscope / Accelerometer)

### Enabling IMU Reporting

```typescript
import { ImuReportPace } from '@evenrealities/even_hub_sdk'

// Prerequisites: createStartUpPageContainer must have been called first
await bridge.imuControl(true, ImuReportPace.P500)   // enable at pace 500
await bridge.imuControl(false)                       // disable
```

> ⚠️ `imuControl` requires `createStartUpPageContainer` to have been called successfully first.

> ⚠️ On the simulator, `imuData` is **always `null`**. IMU features must be tested on real hardware.

### `ImuReportPace` Values

These are protocol-side pace codes — **not physical Hz**. Higher = faster reporting.

| Enum | Value |
|------|-------|
| `ImuReportPace.P100` | 100 (slowest) |
| `ImuReportPace.P200` | 200 |
| `ImuReportPace.P300` | 300 |
| `ImuReportPace.P400` | 400 |
| `ImuReportPace.P500` | 500 (mid) |
| `ImuReportPace.P600` | 600 |
| `ImuReportPace.P700` | 700 |
| `ImuReportPace.P800` | 800 |
| `ImuReportPace.P900` | 900 |
| `ImuReportPace.P1000` | 1000 (fastest) |

### Receiving IMU Data

```typescript
await bridge.imuControl(true, ImuReportPace.P500)

bridge.onEvenHubEvent((event) => {
  const { sysEvent } = event
  if (!sysEvent?.imuData) return
  if (sysEvent.eventType !== OsEventTypeList.IMU_DATA_REPORT) return

  const { x, y, z } = sysEvent.imuData  // protobuf float values
  console.log(`IMU: x=${x.toFixed(3)}, y=${y.toFixed(3)}, z=${z.toFixed(3)}`)
})
```

### `IMU_Report_Data` Fields

| Field | Type | Notes |
|-------|------|-------|
| `x` | number | Protobuf float |
| `y` | number | Protobuf float |
| `z` | number | Protobuf float |

### Gesture Detection with IMU

```typescript
// Head nod detection (Y-axis threshold)
const NOD_THRESHOLD = 2.0
const NOD_COOLDOWN_MS = 800
let lastNodTime = 0

bridge.onEvenHubEvent((event) => {
  const { sysEvent } = event
  if (!sysEvent?.imuData) return

  const { y } = sysEvent.imuData
  const now = Date.now()

  if (Math.abs(y) > NOD_THRESHOLD && now - lastNodTime > NOD_COOLDOWN_MS) {
    lastNodTime = now
    if (y > 0) {
      handleNodDown()
    } else {
      handleNodUp()
    }
  }
})

// Head shake detection (X-axis)
const SHAKE_THRESHOLD = 3.0
let lastShakeTime = 0

bridge.onEvenHubEvent((event) => {
  const { sysEvent } = event
  if (!sysEvent?.imuData) return

  const { x } = sysEvent.imuData
  const now = Date.now()

  if (Math.abs(x) > SHAKE_THRESHOLD && now - lastShakeTime > NOD_COOLDOWN_MS) {
    lastShakeTime = now
    handleHeadShake()
  }
})
```

---

## TouchBar Input

The G2 glasses have a **TouchBar** on the temple. It generates tap and double-tap events that arrive as `textEvent` or `listEvent` (depending on which container has `isEventCapture: 1`).

| Gesture | Event |
|---------|-------|
| Single tap | `CLICK_EVENT` (value `0`, arrives as `undefined`) |
| Double tap | `DOUBLE_CLICK_EVENT` (value `3`) |
| Swipe forward | `SCROLL_BOTTOM_EVENT` (value `2`) |
| Swipe backward | `SCROLL_TOP_EVENT` (value `1`) |

> ⚠️ There is no distinction between TouchBar input and R1 Ring input at the SDK level. Both generate the same event types.

---

## R1 Smart Ring Input

The R1 Smart Ring connects separately via BLE and generates the same event types as the TouchBar. The SDK does not distinguish between ring and temple input.

| Gesture | Event |
|---------|-------|
| Single tap | `CLICK_EVENT` (value `0`, arrives as `undefined`) |
| Double tap | `DOUBLE_CLICK_EVENT` (value `3`) |
| Swipe up/forward | `SCROLL_BOTTOM_EVENT` (value `2`) |
| Swipe down/backward | `SCROLL_TOP_EVENT` (value `1`) |

**Ring availability check:**

```typescript
const device = await bridge.getDeviceInfo()
if (device?.isRing()) {
  // R1 ring is connected
  console.log('Ring SN:', device.sn)
}

// Monitor ring connection
bridge.onDeviceStatusChanged((status) => {
  // status.sn identifies which device changed
  // Cross-reference with getDeviceInfo() to determine if it's the ring
})
```

---

## Complete Event Handler (Production Pattern)

A robust event handler that covers all event sources, handles the `CLICK_EVENT = 0` quirk, throttles scroll events, and works on both simulator and real hardware:

```typescript
import {
  EvenHubEvent,
  OsEventTypeList,
  TextContainerUpgrade,
} from '@evenrealities/even_hub_sdk'

interface AppState {
  pageIndex: number
  selectedIndex: number
  isListening: boolean
}

function createEventHandler(bridge: EvenAppBridge, state: AppState) {
  let lastScrollTime = 0
  const SCROLL_COOLDOWN_MS = 300

  function isClickEvent(eventType: OsEventTypeList | undefined): boolean {
    return eventType === OsEventTypeList.CLICK_EVENT || eventType === undefined
  }

  function isDoubleClickEvent(eventType: OsEventTypeList | undefined): boolean {
    return eventType === OsEventTypeList.DOUBLE_CLICK_EVENT
  }

  function isScrollThrottled(): boolean {
    const now = Date.now()
    if (now - lastScrollTime < SCROLL_COOLDOWN_MS) return true
    lastScrollTime = now
    return false
  }

  return bridge.onEvenHubEvent((event: EvenHubEvent) => {
    // ── List events ──────────────────────────────────────────────────────────
    if (event.listEvent) {
      const { eventType, currentSelectItemIndex, currentSelectItemName } = event.listEvent

      // currentSelectItemIndex may be undefined for index 0
      const index = currentSelectItemIndex ?? state.selectedIndex

      if (isClickEvent(eventType)) {
        state.selectedIndex = index
        handleListClick(index, currentSelectItemName ?? '')
      } else if (isDoubleClickEvent(eventType)) {
        handleListDoubleClick(index)
      } else if (eventType === OsEventTypeList.SCROLL_TOP_EVENT) {
        state.selectedIndex = 0
      } else if (eventType === OsEventTypeList.SCROLL_BOTTOM_EVENT) {
        state.selectedIndex = -1  // unknown — firmware handles highlight
      }
      return
    }

    // ── Text events ──────────────────────────────────────────────────────────
    if (event.textEvent) {
      const { eventType } = event.textEvent

      if (isClickEvent(eventType)) {
        handleTextClick()
        return
      }

      if (isDoubleClickEvent(eventType)) {
        handleTextDoubleClick()
        return
      }

      if (isScrollThrottled()) return

      if (eventType === OsEventTypeList.SCROLL_BOTTOM_EVENT) {
        handleScrollDown(state, bridge)
      } else if (eventType === OsEventTypeList.SCROLL_TOP_EVENT) {
        handleScrollUp(state, bridge)
      }
      return
    }

    // ── System events (also handles simulator button clicks) ─────────────────
    if (event.sysEvent) {
      const { eventType, imuData } = event.sysEvent

      if (eventType === OsEventTypeList.FOREGROUND_ENTER_EVENT) {
        onForeground(bridge)
        return
      }

      if (eventType === OsEventTypeList.FOREGROUND_EXIT_EVENT) {
        onBackground(bridge)
        return
      }

      if (eventType === OsEventTypeList.ABNORMAL_EXIT_EVENT) {
        onAbnormalExit()
        return
      }

      if (imuData) {
        handleImu(imuData.x, imuData.y, imuData.z)
        return
      }

      // Simulator sends sysEvent for button clicks
      if (isClickEvent(eventType)) {
        handleTextClick()
        return
      }

      if (isDoubleClickEvent(eventType)) {
        handleTextDoubleClick()
        return
      }

      if (isScrollThrottled()) return

      if (eventType === OsEventTypeList.SCROLL_BOTTOM_EVENT) {
        handleScrollDown(state, bridge)
      } else if (eventType === OsEventTypeList.SCROLL_TOP_EVENT) {
        handleScrollUp(state, bridge)
      }
      return
    }

    // ── Audio events ─────────────────────────────────────────────────────────
    if (event.audioEvent && state.isListening) {
      processPcmFrame(event.audioEvent.audioPcm)
    }
  })
}
```

---

## Device Info API

### Getting Device Info

```typescript
const device = await bridge.getDeviceInfo()

if (device === null) {
  console.log('No device connected')
  return
}

console.log('Model:', device.model)           // DeviceModel.G1 | G2 | Ring1
console.log('Serial:', device.sn)             // string
console.log('Connect type:', device.status.connectType)
console.log('Battery:', device.status.batteryLevel, '%')
console.log('Wearing:', device.status.isWearing)
console.log('Charging:', device.status.isCharging)
console.log('In case:', device.status.isInCase)
console.log('Is glasses:', device.isGlasses()) // true for G1/G2
console.log('Is ring:', device.isRing())        // true for Ring1
```

### `DeviceModel` Enum

| Enum | Meaning |
|------|---------|
| `DeviceModel.G1` | Even Realities G1 glasses |
| `DeviceModel.G2` | Even Realities G2 glasses |
| `DeviceModel.Ring1` | R1 Smart Ring |

### Real-Time Device Monitoring

```typescript
// Monitor connection, battery, wearing state
const unsubscribe = bridge.onDeviceStatusChanged((status) => {
  if (status.isConnected()) {
    updateBatteryDisplay(status.batteryLevel ?? 0)
  }

  if (status.isDisconnected() || status.isConnectionFailed()) {
    showDisconnectedUI()
  }

  if (status.isWearing === false) {
    // Glasses removed — pause content
    pauseContent()
  }
})

// Clean up on app exit
async function cleanup() {
  unsubscribe()
  await bridge.audioControl(false)
  await bridge.imuControl(false)
}
```

> ⚠️ `DeviceInfo.updateStatus(status)` only updates when `status.sn === device.sn`. Mismatched serial numbers are silently ignored.

---

## User Info API

```typescript
const user = await bridge.getUserInfo()

console.log('UID:', user.uid)       // number
console.log('Name:', user.name)     // string
console.log('Avatar:', user.avatar) // URL string
console.log('Country:', user.country) // string
```

Useful for personalising the app experience (e.g. greeting the user by name, storing per-user preferences).

---

## SDK Storage API

The **only** persistent storage available in an Even Hub app. Browser `localStorage` does **not** survive app or glasses restarts inside the `.ehpk` WebView.

```typescript
// Write
const ok = await bridge.setLocalStorage('theme', 'dark')  // returns boolean

// Read (returns '' if key not set — never null)
const theme = await bridge.getLocalStorage('theme')
if (theme === '') {
  // Key not set — use default
}
```

> ⚠️ There is **no `removeLocalStorage`** method. To "delete" a key, write an empty string and treat empty strings as absent when reading.

### Recommended: In-Memory Cache Wrapper

```typescript
const cache = new Map<string, string>()

// Call at startup, before any UI renders
async function initStorage(bridge: EvenAppBridge, keys: string[]) {
  await Promise.all(keys.map(async (key) => {
    const value = await bridge.getLocalStorage(key)
    if (value) cache.set(key, value)
  }))
}

// Synchronous read from cache
function getItem(key: string): string | null {
  return cache.get(key) ?? null
}

// Write-through: update cache immediately, persist in background
function setItem(bridge: EvenAppBridge, key: string, value: string): void {
  cache.set(key, value)
  void bridge.setLocalStorage(key, value).catch(() => {})
}

// Usage
await initStorage(bridge, ['theme', 'lastPage', 'userPrefs'])
const theme = getItem('theme') ?? 'light'
setItem(bridge, 'theme', 'dark')
```

---

## What the SDK Does NOT Expose

| Missing capability | Notes |
|-------------------|-------|
| Direct BLE access | No raw BLE commands from SDK |
| Arbitrary pixel drawing | Limited to list/text/image container model |
| `imgEvent` | Defined in protocol but not in SDK types |
| Audio output | No speaker on G2 hardware |
| Text alignment | Always left-aligned, top-aligned |
| Font size/weight/family | Single LVGL firmware font |
| Background colour or fill | Display background is always black |
| Per-item styling in lists | All items identical styling |
| Programmatic scroll position | Firmware handles internal scrolling; no get/set API |
| Animations or transitions | No animation support |
| Colour images | 4-bit greyscale only |
| Camera | No camera on G2 hardware |
| Speaker / audio output | No speaker on G2 hardware |

---

## Error Codes Reference

### `createStartUpPageContainer` → `StartUpPageCreateResult`

| Code | Meaning | Action |
|------|---------|--------|
| `0` | Success | Proceed |
| `1` | Invalid container configuration | Check `containerTotalNum`, `isEventCapture`, container fields |
| `2` | Oversize — data too large for BLE | Reduce content length or container count |
| `3` | Out of memory on glasses | Reduce content, simplify layout |

```typescript
const result = await bridge.createStartUpPageContainer(...)
if (result !== 0) {
  const messages = ['success', 'invalid config', 'oversize', 'out of memory']
  console.error('Page creation failed:', messages[result] ?? result)
  return
}
```

### `rebuildPageContainer` → `boolean`

Returns `true` on success. Internal SDK constants: `APP_REQUEST_REBUILD_PAGE_SUCCESS` / `APP_REQUEST_REBUILD_PAGE_FAILD` *(typo in SDK source)*.

### `textContainerUpgrade` → `boolean`

Returns `true` on success. Internal SDK constants: `APP_REQUEST_UPGRADE_TEXT_DATA_SUCCESS` / `APP_REQUEST_UPGRADE_TEXT_DATA_FAILED`.

### `updateImageRawData` → `ImageRawDataUpdateResult`

| Value | Meaning | Action |
|-------|---------|--------|
| `success` | OK | — |
| `imageException` | Image processing error | Check image data format |
| `imageSizeInvalid` | Dimensions don't match container or out of range | Check width (20–200) and height (20–100) |
| `imageToGray4Failed` | Greyscale conversion failed | Check image data integrity |
| `sendFailed` | BLE send failed | Retry; check connection |

### `shutDownPageContainer` → `boolean`

Returns `true` on success. Internal SDK constants: `APP_REQUEST_UPGRADE_SHUTDOWN_SUCCESS` / `APP_REQUEST_UPGRADE_SHUTDOWN_FAILED`.

---

## SDK JSON Compatibility

The SDK handles multiple key naming conventions from the host via `pickLoose()` — normalises keys by removing underscores and lowercasing:

| Convention | Example |
|-----------|---------|
| camelCase | `containerID` |
| PascalCase | `ContainerID` |
| Proto-style | `Container_ID` |

Event data is accepted in multiple shapes:

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

You only need to use camelCase in your code. The SDK handles the rest.

---

## Critical Gotchas Summary

| Gotcha | Detail |
|--------|--------|
| `CLICK_EVENT = 0` → `undefined` | Always check `eventType === 0 \|\| eventType === undefined` |
| Scroll events fire rapidly | Use 300ms cooldown to prevent duplicate actions |
| `currentSelectItemIndex` missing for 0 | Fall back to tracking `selectedIndex` in app state |
| Simulator sends `sysEvent` for clicks | Real hardware sends `textEvent` or `listEvent` — handle all three |
| List takes over scroll | Scroll events arrive as `listEvent`, firmware moves highlight natively |
| `audioControl` needs startup first | `createStartUpPageContainer` must succeed first |
| `imuControl` needs startup first | Same prerequisite as `audioControl` |
| IMU always null on simulator | Test IMU features on real hardware only |
| No browser `localStorage` | Use `bridge.setLocalStorage` / `bridge.getLocalStorage` |
| `onLaunchSource` fires once | Register early — missed if registered too late |
| `FOREGROUND_EXIT_EVENT` | Stop audio, pause timers, save state |
| `ABNORMAL_EXIT_EVENT` | Unexpected disconnect — clean up resources |
| No ring vs TouchBar distinction | Both generate identical event types at SDK level |
| No programmatic scroll control | Firmware handles internal scrolling — no get/set API |

---