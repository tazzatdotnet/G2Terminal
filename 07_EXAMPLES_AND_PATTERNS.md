---

# 07 — Examples, Patterns & Complete App Recipes

> **Source**: `nickustinov/even-g2-notes` (ui-patterns.md, browser-ui.md, simulator.md, packaging.md, display.md, page-lifecycle.md), `dmyster145/EvenChess`, `fuutott/rdt-even-g2-rddit-client`, `nickustinov/weather-even-g2`, `fabioglimb/even-toolkit`. All facts verified April 2026.
> **Cross-reference**: `02_SDK_QUICKSTART.md` for initialization, `03_SDK_API_REFERENCE.md` for method signatures, `04_UI_CONTAINERS.md` for container rules, `05_EVENTS_AND_INPUT.md` for event handling.

---

## Complete Project Scaffold

Every Even Hub app has this minimal structure:

```
my-app/
├── index.html          ← Entry point (phone settings UI + glasses app mount)
├── package.json        ← Dependencies and scripts
├── vite.config.ts      ← Dev server config
├── tsconfig.json       ← TypeScript config
├── app.json            ← App manifest (for packaging)
└── src/
    ├── main.ts         ← App bootstrap (bridge init, launch source)
    └── styles.css      ← Stylesheet
```

### `index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>My Even App</title>
  <link rel="stylesheet" href="/src/styles.css" />
</head>
<body>
  <div id="app"></div>
  <script type="module" src="/src/main.ts"></script>
</body>
</html>
```

### `package.json`

```json
{
  "name": "my-even-app",
  "version": "1.0.0",
  "scripts": {
    "dev":   "vite --host 0.0.0.0 --port 5173",
    "build": "vite build",
    "qr":    "evenhub qr --http --port 5173",
    "pack":  "npm run build && evenhub pack app.json dist -o myapp.ehpk"
  },
  "dependencies": {
    "@evenrealities/even_hub_sdk": "^0.0.9"
  },
  "devDependencies": {
    "@evenrealities/evenhub-cli": "latest",
    "typescript": "^5.9.3",
    "vite": "^7.3.1"
  }
}
```

### `vite.config.ts`

```typescript
import { defineConfig } from 'vite'

export default defineConfig({
  server: {
    host: true,
    port: 5173,
  },
})
```

### `app.json`

```json
{
  "package_id": "com.yourname.myapp",
  "edition": "202601",
  "name": "My App",
  "version": "1.0.0",
  "min_app_version": "0.1.0",
  "tagline": "Short description shown in Even Hub",
  "description": "Longer description of what the app does",
  "author": "Your Name",
  "entrypoint": "index.html",
  "permissions": {
    "network": ["api.example.com"]
  }
}
```

> ⚠️ **`package_id` rules:** Reverse-domain format. Each segment starts with a lowercase letter, contains only lowercase letters or numbers. **No hyphens.** `com.myname.myapp` ✅ — `com.my-name.my-app` ❌

---

## Development Workflow

```bash
# 1. Install dependencies
npm install

# 2. Start dev server (keep running)
npm run dev

# 3. In a second terminal: generate QR code
npm run qr
# → Scan with Even App on your phone to load on glasses

# 4. Edit code → Vite hot-reloads automatically

# 5. Package for distribution
npm run pack
# → Creates myapp.ehpk
```

> ⚠️ Use your machine's **local network IP** (e.g. `192.168.x.x`), not `localhost` — the phone needs to reach your dev server over the network. Vite prints the network URL when starting with `--host 0.0.0.0`.

---

## App Bootstrap Pattern (Production-Grade)

The canonical `src/main.ts` that every app should start from:

```typescript
import {
  waitForEvenAppBridge,
  EvenAppBridge,
  CreateStartUpPageContainer,
  TextContainerProperty,
  OsEventTypeList,
  EvenHubEvent,
} from '@evenrealities/even_hub_sdk'

// ── App state ────────────────────────────────────────────────────────────────
interface AppState {
  bridge: EvenAppBridge | null
  pageIndex: number
  selectedIndex: number
  isListening: boolean
  refreshInterval: ReturnType<typeof setInterval> | null
}

const state: AppState = {
  bridge: null,
  pageIndex: 0,
  selectedIndex: 0,
  isListening: false,
  refreshInterval: null,
}

// ── Entry point ───────────────────────────────────────────────────────────────
async function main() {
  // 1. Wait for bridge
  const bridge = await waitForEvenAppBridge()
  state.bridge = bridge

  // 2. Register launch source listener IMMEDIATELY after bridge ready
  bridge.onLaunchSource(async (source) => {
    if (source === 'glassesMenu') {
      // Opened from glasses — initialize glasses UI
      await initGlassesUI(bridge)
    } else {
      // Opened from phone app menu — show phone settings UI
      initPhoneUI()
    }
  })

  // 3. Monitor device status
  bridge.onDeviceStatusChanged((status) => {
    if (status.isDisconnected() || status.isConnectionFailed()) {
      onDisconnected()
    }
    if (status.isConnected()) {
      onConnected(status.batteryLevel ?? 0)
    }
  })
}

// ── Glasses UI init ───────────────────────────────────────────────────────────
async function initGlassesUI(bridge: EvenAppBridge) {
  // Create startup page
  const result = await bridge.createStartUpPageContainer(
    new CreateStartUpPageContainer({
      containerTotalNum: 1,
      textObject: [
        new TextContainerProperty({
          containerID: 1,
          containerName: 'main',
          content: 'Loading...',
          xPosition: 0,
          yPosition: 0,
          width: 576,
          height: 288,
          borderWidth: 0,
          borderColor: 0,
          paddingLength: 8,
          isEventCapture: 1,
        }),
      ],
    })
  )

  if (result !== 0) {
    const errors = ['success', 'invalid config', 'oversize', 'out of memory']
    console.error('Page creation failed:', errors[result] ?? result)
    return
  }

  // Register event handler
  bridge.onEvenHubEvent(createEventHandler(bridge, state))

  // Register lifecycle handlers
  bridge.onEvenHubEvent((event) => {
    if (event.sysEvent?.eventType === OsEventTypeList.FOREGROUND_ENTER_EVENT) {
      onForeground(bridge)
    }
    if (event.sysEvent?.eventType === OsEventTypeList.FOREGROUND_EXIT_EVENT) {
      onBackground(bridge)
    }
  })

  // Load initial content
  await renderPage(bridge, state)
}

// ── Phone UI init ─────────────────────────────────────────────────────────────
function initPhoneUI() {
  document.getElementById('app')!.innerHTML = `
    <div style="padding: 20px; font-family: sans-serif;">
      <h1>My Even App</h1>
      <p>Open this app from your glasses menu to use it.</p>
    </div>
  `
}

// ── Lifecycle ─────────────────────────────────────────────────────────────────
function onForeground(bridge: EvenAppBridge) {
  renderPage(bridge, state)
  state.refreshInterval = setInterval(() => renderPage(bridge, state), 30_000)
}

function onBackground(bridge: EvenAppBridge) {
  if (state.refreshInterval) {
    clearInterval(state.refreshInterval)
    state.refreshInterval = null
  }
  bridge.audioControl(false).catch(() => {})
}

function onConnected(battery: number) {
  console.log(`Connected. Battery: ${battery}%`)
}

function onDisconnected() {
  console.log('Disconnected')
}

// ── Render ────────────────────────────────────────────────────────────────────
async function renderPage(bridge: EvenAppBridge, state: AppState) {
  // Your rendering logic here
}

// ── Event handler ─────────────────────────────────────────────────────────────
function createEventHandler(bridge: EvenAppBridge, state: AppState) {
  let lastScrollTime = 0
  const SCROLL_COOLDOWN = 300

  return (event: EvenHubEvent) => {
    const isClick = (t: OsEventTypeList | undefined) =>
      t === OsEventTypeList.CLICK_EVENT || t === undefined
    const isDouble = (t: OsEventTypeList | undefined) =>
      t === OsEventTypeList.DOUBLE_CLICK_EVENT

    if (event.textEvent) {
      const { eventType } = event.textEvent
      if (isClick(eventType)) { handleClick(bridge, state); return }
      if (isDouble(eventType)) { handleDoubleClick(bridge, state); return }
      const now = Date.now()
      if (now - lastScrollTime < SCROLL_COOLDOWN) return
      lastScrollTime = now
      if (eventType === OsEventTypeList.SCROLL_BOTTOM_EVENT) handleScrollDown(bridge, state)
      if (eventType === OsEventTypeList.SCROLL_TOP_EVENT) handleScrollUp(bridge, state)
      return
    }

    if (event.listEvent) {
      const { eventType, currentSelectItemIndex } = event.listEvent
      const index = currentSelectItemIndex ?? state.selectedIndex
      if (isClick(eventType)) { state.selectedIndex = index; handleListClick(bridge, state, index) }
      return
    }

    // Simulator sends sysEvent for button clicks
    if (event.sysEvent) {
      const { eventType } = event.sysEvent
      if (isClick(eventType)) { handleClick(bridge, state); return }
      if (isDouble(eventType)) { handleDoubleClick(bridge, state); return }
      const now = Date.now()
      if (now - lastScrollTime < SCROLL_COOLDOWN) return
      lastScrollTime = now
      if (eventType === OsEventTypeList.SCROLL_BOTTOM_EVENT) handleScrollDown(bridge, state)
      if (eventType === OsEventTypeList.SCROLL_TOP_EVENT) handleScrollUp(bridge, state)
    }
  }
}

function handleClick(bridge: EvenAppBridge, state: AppState) { /* ... */ }
function handleDoubleClick(bridge: EvenAppBridge, state: AppState) { /* ... */ }
function handleScrollDown(bridge: EvenAppBridge, state: AppState) { /* ... */ }
function handleScrollUp(bridge: EvenAppBridge, state: AppState) { /* ... */ }
function handleListClick(bridge: EvenAppBridge, state: AppState, index: number) { /* ... */ }

main()
```

---

## Recipe 1: Simple Text Dashboard (Single Screen)

The simplest possible useful app — displays live data on one screen.

```typescript
import {
  waitForEvenAppBridge,
  CreateStartUpPageContainer,
  TextContainerProperty,
  OsEventTypeList,
} from '@evenrealities/even_hub_sdk'

async function main() {
  const bridge = await waitForEvenAppBridge()

  bridge.onLaunchSource(async (source) => {
    if (source !== 'glassesMenu') return

    // Create single full-screen text container
    const result = await bridge.createStartUpPageContainer(
      new CreateStartUpPageContainer({
        containerTotalNum: 1,
        textObject: [
          new TextContainerProperty({
            containerID: 1,
            containerName: 'dashboard',
            content: buildDashboard(),
            xPosition: 0,
            yPosition: 0,
            width: 576,
            height: 288,
            borderWidth: 0,
            borderColor: 0,
            paddingLength: 8,
            isEventCapture: 1,
          }),
        ],
      })
    )

    if (result !== 0) return

    // Refresh every 60 seconds
    setInterval(async () => {
      await bridge.textContainerUpgrade({
        containerID: 1,
        containerName: 'dashboard',
        content: buildDashboard(),
      })
    }, 60_000)

    // Double-tap to force refresh
    bridge.onEvenHubEvent((event) => {
      const t = event.textEvent?.eventType ?? event.sysEvent?.eventType
      if (t === OsEventTypeList.DOUBLE_CLICK_EVENT) {
        bridge.textContainerUpgrade({
          containerID: 1,
          containerName: 'dashboard',
          content: buildDashboard(),
        })
      }
    })
  })
}

function buildDashboard(): string {
  const now = new Date()
  const time = now.toLocaleTimeString('en-US', { hour: '2-digit', minute: '2-digit' })
  const date = now.toLocaleDateString('en-US', { weekday: 'long', month: 'short', day: 'numeric' })

  return [
    `${time}`,
    `${date}`,
    ``,
    `━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━`,
    ``,
    `Tap to refresh`,
  ].join('\n')
}

main()
```

---

## Recipe 2: Multi-Page Text Reader

Paginated long-form content with page indicator and scroll navigation.

```typescript
import {
  waitForEvenAppBridge,
  CreateStartUpPageContainer,
  RebuildPageContainer,
  TextContainerProperty,
  OsEventTypeList,
} from '@evenrealities/even_hub_sdk'

// ── Text pagination ───────────────────────────────────────────────────────────
function paginateText(text: string, maxChars = 400): string[] {
  const pages: string[] = []
  const words = text.split(' ')
  let current = ''

  for (const word of words) {
    const candidate = current ? `${current} ${word}` : word
    if (candidate.length > maxChars) {
      if (current) pages.push(current.trim())
      current = word
    } else {
      current = candidate
    }
  }
  if (current.trim()) pages.push(current.trim())
  return pages
}

// ── State ─────────────────────────────────────────────────────────────────────
const LONG_TEXT = `Your long article or book content goes here. It will be automatically
paginated into ~400 character pages with word-boundary breaks. Users scroll
through pages with swipe gestures and see a page indicator at the bottom.`

const pages = paginateText(LONG_TEXT)
let pageIndex = 0

// ── Render ────────────────────────────────────────────────────────────────────
function buildPageContent(index: number): string {
  const page = pages[index]
  const indicator = `\n\n─────────────────────────────\n${index + 1} / ${pages.length}`
  return page + indicator
}

async function renderCurrentPage(bridge: any, isFirst = false) {
  const content = buildPageContent(pageIndex)

  if (isFirst) {
    await bridge.createStartUpPageContainer(
      new CreateStartUpPageContainer({
        containerTotalNum: 1,
        textObject: [
          new TextContainerProperty({
            containerID: 1,
            containerName: 'reader',
            content,
            xPosition: 0,
            yPosition: 0,
            width: 576,
            height: 288,
            borderWidth: 0,
            borderColor: 0,
            paddingLength: 8,
            isEventCapture: 1,
          }),
        ],
      })
    )
  } else {
    // textContainerUpgrade is faster and flicker-free on real hardware
    await bridge.textContainerUpgrade({
      containerID: 1,
      containerName: 'reader',
      content,
    })
  }
}

// ── Main ──────────────────────────────────────────────────────────────────────
async function main() {
  const bridge = await waitForEvenAppBridge()

  bridge.onLaunchSource(async (source) => {
    if (source !== 'glassesMenu') return

    await renderCurrentPage(bridge, true)

    let lastScrollTime = 0
    const COOLDOWN = 300

    bridge.onEvenHubEvent(async (event) => {
      const t = event.textEvent?.eventType ?? event.sysEvent?.eventType
      const isClick = t === OsEventTypeList.CLICK_EVENT || t === undefined
      const isDouble = t === OsEventTypeList.DOUBLE_CLICK_EVENT
      const isDown = t === OsEventTypeList.SCROLL_BOTTOM_EVENT
      const isUp = t === OsEventTypeList.SCROLL_TOP_EVENT

      if (isDouble) {
        // Double-tap: go back to first page
        pageIndex = 0
        await renderCurrentPage(bridge)
        return
      }

      if (isClick) return  // single tap: no action in this app

      const now = Date.now()
      if (now - lastScrollTime < COOLDOWN) return
      lastScrollTime = now

      if (isDown && pageIndex < pages.length - 1) {
        pageIndex++
        await renderCurrentPage(bridge)
      } else if (isUp && pageIndex > 0) {
        pageIndex--
        await renderCurrentPage(bridge)
      }
    })
  })
}

main()
```

---

## Recipe 3: Native List Menu with Actions

Uses the native list widget for a scrollable menu. Firmware handles scroll highlighting automatically.

```typescript
import {
  waitForEvenAppBridge,
  CreateStartUpPageContainer,
  RebuildPageContainer,
  ListContainerProperty,
  ListItemContainerProperty,
  TextContainerProperty,
  OsEventTypeList,
} from '@evenrealities/even_hub_sdk'

const MENU_ITEMS = ['Weather', 'News', 'Timer', 'Notes', 'Settings']
let currentView: 'menu' | 'detail' = 'menu'
let selectedItem = ''

async function showMenu(bridge: any, isFirst = false) {
  const config = {
    containerTotalNum: 1,
    listObject: [
      new ListContainerProperty({
        containerID: 1,
        containerName: 'main-menu',
        xPosition: 0,
        yPosition: 0,
        width: 576,
        height: 288,
        borderWidth: 1,
        borderColor: 13,
        borderRadius: 6,
        paddingLength: 5,
        isEventCapture: 1,
        itemContainer: new ListItemContainerProperty({
          itemCount: MENU_ITEMS.length,
          itemWidth: 0,  // auto-fill
          isItemSelectBorderEn: 1,
          itemName: MENU_ITEMS,
        }),
      }),
    ],
  }

  if (isFirst) {
    await bridge.createStartUpPageContainer(new CreateStartUpPageContainer(config))
  } else {
    await bridge.rebuildPageContainer(new RebuildPageContainer(config))
  }
  currentView = 'menu'
}

async function showDetail(bridge: any, item: string) {
  const content = getDetailContent(item)
  await bridge.rebuildPageContainer(
    new RebuildPageContainer({
      containerTotalNum: 1,
      textObject: [
        new TextContainerProperty({
          containerID: 1,
          containerName: 'detail',
          content: `${item}\n\n${content}\n\n─────────────────\nDouble-tap to go back`,
          xPosition: 0,
          yPosition: 0,
          width: 576,
          height: 288,
          borderWidth: 0,
          borderColor: 0,
          paddingLength: 8,
          isEventCapture: 1,
        }),
      ],
    })
  )
  currentView = 'detail'
}

function getDetailContent(item: string): string {
  const content: Record<string, string> = {
    'Weather': 'Fetching weather...',
    'News': 'Fetching headlines...',
    'Timer': 'Timer feature coming soon.',
    'Notes': 'No notes yet.',
    'Settings': 'Version 1.0.0\nEven Hub SDK 0.0.9',
  }
  return content[item] ?? 'No content.'
}

async function main() {
  const bridge = await waitForEvenAppBridge()

  bridge.onLaunchSource(async (source) => {
    if (source !== 'glassesMenu') return

    await showMenu(bridge, true)

    let selectedIndex = 0

    bridge.onEvenHubEvent(async (event) => {
      // ── List events (menu view) ──────────────────────────────────────────
      if (event.listEvent && currentView === 'menu') {
        const { eventType, currentSelectItemIndex, currentSelectItemName } = event.listEvent
        const index = currentSelectItemIndex ?? selectedIndex
        const isClick = eventType === OsEventTypeList.CLICK_EVENT || eventType === undefined

        if (isClick) {
          selectedIndex = index
          selectedItem = currentSelectItemName ?? MENU_ITEMS[index]
          await showDetail(bridge, selectedItem)
        }
        return
      }

      // ── Text events (detail view) ────────────────────────────────────────
      if ((event.textEvent || event.sysEvent) && currentView === 'detail') {
        const t = event.textEvent?.eventType ?? event.sysEvent?.eventType
        const isDouble = t === OsEventTypeList.DOUBLE_CLICK_EVENT
        if (isDouble) {
          await showMenu(bridge)
        }
      }
    })
  })
}

main()
```

---

## Recipe 4: Image Canvas App (Game / Custom UI)

For apps that render custom graphics using the Canvas API and display them as images on the glasses.

```typescript
import {
  waitForEvenAppBridge,
  CreateStartUpPageContainer,
  TextContainerProperty,
  ImageContainerProperty,
  ImageRawDataUpdate,
  OsEventTypeList,
} from '@evenrealities/even_hub_sdk'

// ── Canvas renderer ───────────────────────────────────────────────────────────
// Image container max: 200x100px
const IMG_W = 200
const IMG_H = 100

function renderFrame(frameData: any): string {
  // Create offscreen canvas
  const canvas = document.createElement('canvas')
  canvas.width = IMG_W
  canvas.height = IMG_H
  const ctx = canvas.getContext('2d')!

  // Draw your content (greyscale — colour is discarded by host)
  ctx.fillStyle = '#000000'
  ctx.fillRect(0, 0, IMG_W, IMG_H)

  // Example: draw a simple shape
  ctx.fillStyle = '#FFFFFF'
  ctx.fillRect(frameData.x, frameData.y, 20, 20)

  // Return base64 PNG (strip data URL prefix)
  return canvas.toDataURL('image/png').replace(/^data:image\/png;base64,/, '')
}

// ── Image update queue (sequential — never concurrent) ────────────────────────
let imageUpdateInProgress = false
const imageUpdateQueue: (() => Promise<void>)[] = []

async function processImageQueue() {
  if (imageUpdateInProgress || imageUpdateQueue.length === 0) return
  imageUpdateInProgress = true
  const task = imageUpdateQueue.shift()!
  await task()
  imageUpdateInProgress = false
  processImageQueue()
}

function queueImageUpdate(bridge: any, base64: string) {
  imageUpdateQueue.push(async () => {
    await bridge.updateImageRawData(new ImageRawDataUpdate({
      containerID: 2,
      containerName: 'canvas',
      imageData: base64,
    }))
  })
  processImageQueue()
}

// ── Main ──────────────────────────────────────────────────────────────────────
async function main() {
  const bridge = await waitForEvenAppBridge()

  bridge.onLaunchSource(async (source) => {
    if (source !== 'glassesMenu') return

    // CRITICAL: Use a text container for event capture (not image container)
    // Place text container BEHIND image (lower containerID = drawn first)
    const result = await bridge.createStartUpPageContainer(
      new CreateStartUpPageContainer({
        containerTotalNum: 2,
        textObject: [
          new TextContainerProperty({
            containerID: 1,
            containerName: 'evt',
            content: ' ',  // single space — invisible but captures events
            xPosition: 0,
            yPosition: 0,
            width: 576,
            height: 288,
            borderWidth: 0,
            borderColor: 0,
            paddingLength: 0,
            isEventCapture: 1,  // ← captures ALL event types including scroll
          }),
        ],
        imageObject: [
          new ImageContainerProperty({
            containerID: 2,
            containerName: 'canvas',
            // Centre the 200x100 image on the 576x288 canvas
            xPosition: 188,  // (576 - 200) / 2
            yPosition: 94,   // (288 - 100) / 2
            width: IMG_W,
            height: IMG_H,
          }),
        ],
      })
    )

    if (result !== 0) return

    // Initial render
    const gameState = { x: 90, y: 40 }
    queueImageUpdate(bridge, renderFrame(gameState))

    // Handle input — events arrive as textEvent (not listEvent)
    let lastScrollTime = 0
    bridge.onEvenHubEvent((event) => {
      // Events come from the text container (isEventCapture: 1)
      const t = event.textEvent?.eventType ?? event.sysEvent?.eventType
      const isClick = t === OsEventTypeList.CLICK_EVENT || t === undefined
      const isDown = t === OsEventTypeList.SCROLL_BOTTOM_EVENT
      const isUp = t === OsEventTypeList.SCROLL_TOP_EVENT

      const now = Date.now()
      if (!isClick && now - lastScrollTime < 200) return
      if (!isClick) lastScrollTime = now

      if (isDown) gameState.y = Math.min(IMG_H - 20, gameState.y + 10)
      if (isUp) gameState.y = Math.max(0, gameState.y - 10)
      if (isClick) gameState.x = Math.random() * (IMG_W - 20)

      queueImageUpdate(bridge, renderFrame(gameState))
    })
  })
}

main()
```

> ⚠️ **Image container event capture quirk:** Image containers do NOT have `isEventCapture`. You MUST use a text container with `isEventCapture: 1` placed behind the image. A hidden 1×1 list with 1 item does NOT generate scroll events. Use a full-screen text container with content `' '` (single space).

> ⚠️ **Image container max size is 200×100px** — it cannot cover the full 576×288 canvas. Centre it: `xPosition = (576-200)/2 = 188`, `yPosition = (288-100)/2 = 94`.

---

## Recipe 5: Voice Assistant (Microphone + STT + LLM)

Full voice assistant flow: tap to listen, STT, LLM response, display on glasses.

```typescript
import {
  waitForEvenAppBridge,
  CreateStartUpPageContainer,
  TextContainerProperty,
  OsEventTypeList,
} from '@evenrealities/even_hub_sdk'

// ── PCM accumulator ───────────────────────────────────────────────────────────
// SDK delivers PCM S16LE, 16kHz, 40 bytes/frame (10ms)
const PCM_FRAMES_PER_CHUNK = 100  // 100 × 10ms = 1 second chunks
const pcmBuffer: Uint8Array[] = []

function mergePcmFrames(frames: Uint8Array[]): Uint8Array {
  const total = frames.reduce((s, f) => s + f.length, 0)
  const merged = new Uint8Array(total)
  let offset = 0
  for (const f of frames) { merged.set(f, offset); offset += f.length }
  return merged
}

// ── STT + LLM ─────────────────────────────────────────────────────────────────
async function transcribeAndRespond(pcmChunks: Uint8Array[]): Promise<string> {
  const allPcm = mergePcmFrames(pcmChunks)

  // Convert PCM to WAV for STT API
  const wav = pcmToWav(allPcm, 16000)
  const formData = new FormData()
  formData.append('file', new Blob([wav], { type: 'audio/wav' }), 'audio.wav')
  formData.append('model', 'whisper-1')

  const sttResp = await fetch('https://api.openai.com/v1/audio/transcriptions', {
    method: 'POST',
    headers: { 'Authorization': `Bearer ${YOUR_API_KEY}` },
    body: formData,
  })
  const { text: transcript } = await sttResp.json()
  if (!transcript) return 'Could not understand audio.'

  // Send to LLM
  const llmResp = await fetch('https://api.openai.com/v1/chat/completions', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${YOUR_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model: 'gpt-4o-mini',
      messages: [
        { role: 'system', content: 'You are a concise assistant for smart glasses. Keep responses under 200 characters.' },
        { role: 'user', content: transcript },
      ],
      max_tokens: 100,
    }),
  })
  const llmData = await llmResp.json()
  return llmData.choices?.[0]?.message?.content ?? 'No response.'
}

// ── PCM → WAV conversion ──────────────────────────────────────────────────────
function pcmToWav(pcm: Uint8Array, sampleRate: number): ArrayBuffer {
  const numChannels = 1
  const bitsPerSample = 16
  const byteRate = sampleRate * numChannels * bitsPerSample / 8
  const blockAlign = numChannels * bitsPerSample / 8
  const dataSize = pcm.length
  const buffer = new ArrayBuffer(44 + dataSize)
  const view = new DataView(buffer)

  const writeStr = (offset: number, str: string) => {
    for (let i = 0; i < str.length; i++) view.setUint8(offset + i, str.charCodeAt(i))
  }

  writeStr(0, 'RIFF')
  view.setUint32(4, 36 + dataSize, true)
  writeStr(8, 'WAVE')
  writeStr(12, 'fmt ')
  view.setUint32(16, 16, true)
  view.setUint16(20, 1, true)
  view.setUint16(22, numChannels, true)
  view.setUint32(24, sampleRate, true)
  view.setUint32(28, byteRate, true)
  view.setUint16(32, blockAlign, true)
  view.setUint16(34, bitsPerSample, true)
  writeStr(36, 'data')
  view.setUint32(40, dataSize, true)
  new Uint8Array(buffer, 44).set(pcm)
  return buffer
}

// ── UI helpers ────────────────────────────────────────────────────────────────
async function updateStatus(bridge: any, text: string) {
  await bridge.textContainerUpgrade({
    containerID: 1,
    containerName: 'status',
    content: text,
  })
}

// ── Main ──────────────────────────────────────────────────────────────────────
async function main() {
  const bridge = await waitForEvenAppBridge()

  bridge.onLaunchSource(async (source) => {
    if (source !== 'glassesMenu') return

    const result = await bridge.createStartUpPageContainer(
      new CreateStartUpPageContainer({
        containerTotalNum: 1,
        textObject: [
          new TextContainerProperty({
            containerID: 1,
            containerName: 'status',
            content: 'Tap to speak',
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
    if (result !== 0) return

    let isListening = false
    const allPcmChunks: Uint8Array[] = []

    bridge.onEvenHubEvent(async (event) => {
      // ── Toggle listening on tap ──────────────────────────────────────────
      const t = event.textEvent?.eventType ?? event.sysEvent?.eventType
      const isClick = t === OsEventTypeList.CLICK_EVENT || t === undefined

      if (isClick) {
        if (!isListening) {
          // Start listening
          isListening = true
          allPcmChunks.length = 0
          await bridge.audioControl(true)
          await updateStatus(bridge, '● Listening...\n\nTap to stop')
        } else {
          // Stop listening
          isListening = false
          await bridge.audioControl(false)
          await updateStatus(bridge, '⟳ Processing...')

          try {
            const response = await transcribeAndRespond(allPcmChunks)
            await updateStatus(bridge, response + '\n\n─────────────────\nTap to speak again')
          } catch (err) {
            await updateStatus(bridge, 'Error. Tap to try again.')
          }
        }
        return
      }

      // ── Accumulate PCM ───────────────────────────────────────────────────
      if (event.audioEvent && isListening) {
        allPcmChunks.push(event.audioEvent.audioPcm)

        // Auto-stop at 30 seconds (30s × 100 frames/s = 3000 frames)
        if (allPcmChunks.length >= 3000) {
          isListening = false
          await bridge.audioControl(false)
          await updateStatus(bridge, '⟳ Processing (max duration)...')
          const response = await transcribeAndRespond(allPcmChunks)
          await updateStatus(bridge, response + '\n\n─────────────────\nTap to speak again')
        }
      }
    })
  })
}

const YOUR_API_KEY = 'sk-...'  // Store server-side in production!
main()
```

---

## Recipe 6: Multi-Screen App with Navigation

A full multi-screen app with a navigation stack, back button, and per-screen state.

```typescript
import {
  waitForEvenAppBridge,
  EvenAppBridge,
  CreateStartUpPageContainer,
  RebuildPageContainer,
  TextContainerProperty,
  ListContainerProperty,
  ListItemContainerProperty,
  OsEventTypeList,
} from '@evenrealities/even_hub_sdk'

// ── Screen definitions ────────────────────────────────────────────────────────
type Screen = 'home' | 'list' | 'detail' | 'settings'

interface NavState {
  stack: Screen[]
  listIndex: number
  detailContent: string
}

const nav: NavState = {
  stack: ['home'],
  listIndex: 0,
  detailContent: '',
}

function currentScreen(): Screen {
  return nav.stack[nav.stack.length - 1]
}

function pushScreen(screen: Screen) {
  nav.stack.push(screen)
}

function popScreen(): Screen | undefined {
  if (nav.stack.length > 1) nav.stack.pop()
  return currentScreen()
}

// ── Screen renderers ──────────────────────────────────────────────────────────
const LIST_ITEMS = ['Article 1', 'Article 2', 'Article 3', 'Article 4', 'Article 5']

async function renderScreen(bridge: EvenAppBridge, isFirst = false) {
  const screen = currentScreen()

  switch (screen) {
    case 'home':
      await renderHome(bridge, isFirst)
      break
    case 'list':
      await renderList(bridge, isFirst)
      break
    case 'detail':
      await renderDetail(bridge, isFirst)
      break
    case 'settings':
      await renderSettings(bridge, isFirst)
      break
  }
}

async function renderHome(bridge: EvenAppBridge, isFirst: boolean) {
  const content = [
    'My App',
    '━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━',
    '',
    '▶ Swipe down  → Browse articles',
    '▶ Double-tap  → Settings',
    '',
    'Welcome back!',
  ].join('\n')

  await applyPage(bridge, isFirst, {
    containerTotalNum: 1,
    textObject: [makeTextContainer(1, 'home', content)],
  })
}

async function renderList(bridge: EvenAppBridge, isFirst: boolean) {
  await applyPage(bridge, isFirst, {
    containerTotalNum: 1,
    listObject: [
      new ListContainerProperty({
        containerID: 1,
        containerName: 'articles',
        xPosition: 0,
        yPosition: 0,
        width: 576,
        height: 288,
        borderWidth: 1,
        borderColor: 13,
        borderRadius: 6,
        paddingLength: 5,
        isEventCapture: 1,
        itemContainer: new ListItemContainerProperty({
          itemCount: LIST_ITEMS.length,
          itemWidth: 0,
          isItemSelectBorderEn: 1,
          itemName: LIST_ITEMS,
        }),
      }),
    ],
  })
}

async function renderDetail(bridge: EvenAppBridge, isFirst: boolean) {
  const content = nav.detailContent + '\n\n─────────────────────────────\nDouble-tap to go back'
  await applyPage(bridge, isFirst, {
    containerTotalNum: 1,
    textObject: [makeTextContainer(1, 'detail', content)],
  })
}

async function renderSettings(bridge: EvenAppBridge, isFirst: boolean) {
  const content = [
    'Settings',
    '━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━',
    '',
    'Version: 1.0.0',
    'SDK: 0.0.9',
    '',
    'Double-tap to go back',
  ].join('\n')

  await applyPage(bridge, isFirst, {
    containerTotalNum: 1,
    textObject: [makeTextContainer(1, 'settings', content)],
  })
}

// ── Helpers ───────────────────────────────────────────────────────────────────
function makeTextContainer(id: number, name: string, content: string) {
  return new TextContainerProperty({
    containerID: id,
    containerName: name,
    content,
    xPosition: 0,
    yPosition: 0,
    width: 576,
    height: 288,
    borderWidth: 0,
    borderColor: 0,
    paddingLength: 8,
    isEventCapture: 1,
  })
}

async function applyPage(bridge: EvenAppBridge, isFirst: boolean, config: any) {
  if (isFirst) {
    await bridge.createStartUpPageContainer(new CreateStartUpPageContainer(config))
  } else {
    await bridge.rebuildPageContainer(new RebuildPageContainer(config))
  }
}

// ── Main ──────────────────────────────────────────────────────────────────────
async function main() {
  const bridge = await waitForEvenAppBridge()

  bridge.onLaunchSource(async (source) => {
    if (source !== 'glassesMenu') return

    await renderScreen(bridge, true)

    let lastScrollTime = 0
    const COOLDOWN = 300

    bridge.onEvenHubEvent(async (event) => {
      const screen = currentScreen()

      // ── List events ──────────────────────────────────────────────────────
      if (event.listEvent && screen === 'list') {
        const { eventType, currentSelectItemIndex, currentSelectItemName } = event.listEvent
        const isClick = eventType === OsEventTypeList.CLICK_EVENT || eventType === undefined
        if (isClick) {
          nav.listIndex = currentSelectItemIndex ?? nav.listIndex
          nav.detailContent = `${currentSelectItemName ?? LIST_ITEMS[nav.listIndex]}\n\nThis is the full content of the article. It can be much longer and will scroll internally if it exceeds the container height.`
          pushScreen('detail')
          await renderScreen(bridge)
        }
        return
      }

      // ── Text / sys events ────────────────────────────────────────────────
      const t = event.textEvent?.eventType ?? event.sysEvent?.eventType
      const isClick = t === OsEventTypeList.CLICK_EVENT || t === undefined
      const isDouble = t === OsEventTypeList.DOUBLE_CLICK_EVENT
      const isDown = t === OsEventTypeList.SCROLL_BOTTOM_EVENT
      const isUp = t === OsEventTypeList.SCROLL_TOP_EVENT

      if (isDouble) {
        if (screen === 'home') {
          pushScreen('settings')
        } else {
          popScreen()
        }
        await renderScreen(bridge)
        return
      }

      if (isClick) return

      const now = Date.now()
      if (now - lastScrollTime < COOLDOWN) return
      lastScrollTime = now

      if (screen === 'home') {
        if (isDown) { pushScreen('list'); await renderScreen(bridge) }
      }
    })
  })
}

main()
```

---

## Recipe 7: Weather App (Real-World Pattern)

Based on [nickustinov/weather-even-g2](https://github.com/nickustinov/weather-even-g2) — demonstrates multi-column layout, image overlays, and external API calls with no backend.

```typescript
// Architecture: no backend needed — browser calls Open-Meteo directly (free, CORS-enabled)
// [G2 glasses] <--BLE--> [Even app] <--HTTP--> [Open-Meteo API]

import {
  waitForEvenAppBridge,
  CreateStartUpPageContainer,
  RebuildPageContainer,
  TextContainerProperty,
  ImageContainerProperty,
  ImageRawDataUpdate,
  OsEventTypeList,
} from '@evenrealities/even_hub_sdk'

// ── Screens ───────────────────────────────────────────────────────────────────
// 5 screens: weekly, current, precipitation, wind, hourly
// Each uses up to 4 containers (the max per page)

// Screen 1: Weekly forecast — 4 containers
// ┌─────────────────────────────────────────────────────────────────────────┐
// │ Header: "London  18°C  Partly Cloudy"                                   │ ← text container (full width, top 72px)
// ├──────────────┬──────────────────────┬──────────────────────────────────┤
// │ Day names    │ Temperatures         │ Conditions                       │ ← 3 text containers side by side
// │ Mon          │ 18° / 12°            │ Partly Cloudy                    │
// │ Tue          │ 21° / 14°            │ Sunny                            │
// │ Wed          │ 16° / 10°            │ Rain                             │
// │ Thu          │ 19° / 13°            │ Cloudy                           │
// │ Fri          │ 22° / 15°            │ Sunny                            │
// └──────────────┴──────────────────────┴──────────────────────────────────┘

async function renderWeeklyScreen(bridge: any, data: WeatherData, isFirst: boolean) {
  const header = `${data.city}  ${data.current.temp}°C  ${data.current.condition}`
  const days = data.weekly.map(d => d.name).join('\n')
  const temps = data.weekly.map(d => `${d.high}° / ${d.low}°`).join('\n')
  const conditions = data.weekly.map(d => d.condition).join('\n')

  const config = {
    containerTotalNum: 4,
    textObject: [
      new TextContainerProperty({
        containerID: 1, containerName: 'header',
        content: header,
        xPosition: 0, yPosition: 0, width: 576, height: 72,
        borderWidth: 0, borderColor: 0, paddingLength: 8,
        isEventCapture: 1,
      }),
      new TextContainerProperty({
        containerID: 2, containerName: 'days',
        content: days,
        xPosition: 0, yPosition: 72, width: 100, height: 216,
        borderWidth: 0, borderColor: 0, paddingLength: 4,
        isEventCapture: 0,
      }),
      new TextContainerProperty({
        containerID: 3, containerName: 'temps',
        content: temps,
        xPosition: 100, yPosition: 72, width: 200, height: 216,
        borderWidth: 0, borderColor: 0, paddingLength: 4,
        isEventCapture: 0,
      }),
      new TextContainerProperty({
        containerID: 4, containerName: 'conditions',
        content: conditions,
        xPosition: 300, yPosition: 72, width: 276, height: 216,
        borderWidth: 0, borderColor: 0, paddingLength: 4,
        isEventCapture: 0,
      }),
    ],
  }

  if (isFirst) {
    await bridge.createStartUpPageContainer(new CreateStartUpPageContainer(config))
  } else {
    await bridge.rebuildPageContainer(new RebuildPageContainer(config))
  }
}

// Screen 2: Current conditions — 3 text containers + 1 image (weather icon)
async function renderCurrentScreen(bridge: any, data: WeatherData) {
  const labels = ['Feels like', 'Humidity', 'Wind', 'UV Index', 'Visibility'].join('\n')
  const values = [
    `${data.current.feelsLike}°C`,
    `${data.current.humidity}%`,
    `${data.current.windSpeed} km/h ${data.current.windDir}`,
    `${data.current.uvIndex}`,
    `${data.current.visibility} km`,
  ].join('\n')

  await bridge.rebuildPageContainer(
    new RebuildPageContainer({
      containerTotalNum: 4,
      textObject: [
        new TextContainerProperty({
          containerID: 1, containerName: 'header',
          content: `${data.city}  ${data.current.temp}°C`,
          xPosition: 0, yPosition: 0, width: 376, height: 72,
          borderWidth: 0, borderColor: 0, paddingLength: 8,
          isEventCapture: 1,
        }),
        new TextContainerProperty({
          containerID: 2, containerName: 'labels',
          content: labels,
          xPosition: 0, yPosition: 72, width: 180, height: 216,
          borderWidth: 0, borderColor: 0, paddingLength: 8,
          isEventCapture: 0,
        }),
        new TextContainerProperty({
          containerID: 3, containerName: 'values',
          content: values,
          xPosition: 180, yPosition: 72, width: 196, height: 216,
          borderWidth: 0, borderColor: 0, paddingLength: 8,
          isEventCapture: 0,
        }),
      ],
      imageObject: [
        new ImageContainerProperty({
          containerID: 4, containerName: 'icon',
          xPosition: 376, yPosition: 0,
          width: 100, height: 100,
        }),
      ],
    })
  )

  // Send weather icon image (must be after page creation)
  const iconBase64 = await renderWeatherIcon(data.current.condition)
  await bridge.updateImageRawData(new ImageRawDataUpdate({
    containerID: 4,
    containerName: 'icon',
    imageData: iconBase64,
  }))
}

// Screen 3: Precipitation bar chart using Unicode block characters
async function renderPrecipScreen(bridge: any, data: WeatherData) {
  const MAX_BAR = 20  // max bar width in chars
  const maxMm = Math.max(...data.hourly.map(h => h.precip), 1)

  const times = data.hourly.slice(0, 9).map(h => h.time).join('\n')
  const bars = data.hourly.slice(0, 9).map(h => {
    const filled = Math.round((h.precip / maxMm) * MAX_BAR)
    return '█'.repeat(filled) + '░'.repeat(MAX_BAR - filled) + ` ${h.precip}mm`
  }).join('\n')

  await bridge.rebuildPageContainer(
    new RebuildPageContainer({
      containerTotalNum: 3,
      textObject: [
        new TextContainerProperty({
          containerID: 1, containerName: 'header',
          content: `Precipitation  Total: ${data.daily.totalPrecip}mm`,
          xPosition: 0, yPosition: 0, width: 476, height: 72,
          borderWidth: 0, borderColor: 0, paddingLength: 8,
          isEventCapture: 1,
        }),
        new TextContainerProperty({
          containerID: 2, containerName: 'times',
          content: times,
          xPosition: 0, yPosition: 72, width: 80, height: 216,
          borderWidth: 0, borderColor: 0, paddingLength: 4,
          isEventCapture: 0,
        }),
        new TextContainerProperty({
          containerID: 3, containerName: 'bars',
          content: bars,
          xPosition: 80, yPosition: 72, width: 396, height: 216,
          borderWidth: 0, borderColor: 0, paddingLength: 4,
          isEventCapture: 0,
        }),
      ],
    })
  )
}

// ── Type stubs ────────────────────────────────────────────────────────────────
interface WeatherData {
  city: string
  current: { temp: number; feelsLike: number; condition: string; humidity: number; windSpeed: number; windDir: string; uvIndex: number; visibility: number }
  weekly: Array<{ name: string; high: number; low: number; condition: string }>
  hourly: Array<{ time: string; precip: number }>
  daily: { totalPrecip: number }
}

async function renderWeatherIcon(condition: string): Promise<string> {
  // Draw icon on canvas, return base64 PNG
  const canvas = document.createElement('canvas')
  canvas.width = 100; canvas.height = 100
  const ctx = canvas.getContext('2d')!
  ctx.fillStyle = '#000'
  ctx.fillRect(0, 0, 100, 100)
  // ... draw icon based on condition ...
  return canvas.toDataURL('image/png').replace(/^data:image\/png;base64,/, '')
}
```

---

## UI Patterns Reference

### Pattern 1: Fake Button Menu with Cursor

Since there are no button widgets, simulate interactive items with a `>` cursor:

```typescript
const MENU_ITEMS = ['Option A', 'Option B', 'Option C', 'Exit']
let cursorIndex = 0

function buildCursorMenu(): string {
  return MENU_ITEMS.map((item, i) =>
    i === cursorIndex ? `> ${item}` : `  ${item}`
  ).join('\n')
}

// On scroll: move cursor, update text (no full rebuild needed)
async function moveCursor(bridge: any, direction: 'up' | 'down') {
  if (direction === 'down') cursorIndex = Math.min(cursorIndex + 1, MENU_ITEMS.length - 1)
  if (direction === 'up') cursorIndex = Math.max(cursorIndex - 1, 0)

  // textContainerUpgrade is faster than rebuildPageContainer — no flicker
  await bridge.textContainerUpgrade({
    containerID: 1,
    containerName: 'menu',
    content: buildCursorMenu(),
  })
}

// On click: execute selected action
function executeSelected() {
  const selected = MENU_ITEMS[cursorIndex]
  console.log('Selected:', selected)
}
```

### Pattern 2: Selection Highlight with Border Toggle

Simulate a list using multiple text containers, each representing a row. Toggle `borderWidth` to show selection:

```typescript
// 3 rows × 96px height = 288px total
const ROW_HEIGHT = 96
const ITEMS = ['Row 1', 'Row 2', 'Row 3']
let selectedRow = 0

async function renderRows(bridge: any, isFirst: boolean) {
  const containers = ITEMS.map((item, i) =>
    new TextContainerProperty({
      containerID: i + 1,
      containerName: `row-${i}`,
      content: item,
      xPosition: 0,
      yPosition: i * ROW_HEIGHT,
      width: 576,
      height: ROW_HEIGHT,
      borderWidth: i === selectedRow ? 2 : 0,  // ← selection highlight
      borderColor: 13,
      borderRadius: 4,
      paddingLength: 8,
      isEventCapture: i === 0 ? 1 : 0,  // only first container captures events
    })
  )

  const config = { containerTotalNum: 3, textObject: containers }
  if (isFirst) {
    await bridge.createStartUpPageContainer(new CreateStartUpPageContainer(config))
  } else {
    await bridge.rebuildPageContainer(new RebuildPageContainer(config))
  }
}
```

> ⚠️ This pattern requires `rebuildPageContainer` to change the border — you cannot update individual container properties in-place. Use `textContainerUpgrade` only for text content changes.

### Pattern 3: Unicode Progress Bar

```typescript
function progressBar(value: number, max: number, width = 20): string {
  const filled = Math.round((value / max) * width)
  const empty = width - filled
  return '━'.repeat(filled) + '─'.repeat(empty)
}

// Usage:
const bar = progressBar(7, 10, 20)  // → "━━━━━━━━━━━━━━──────"
const display = `Progress\n${bar} ${Math.round((7/10)*100)}%`
```

**Block-based vertical bar (for charts):**

```typescript
const BLOCKS = ['▁', '▂', '▃', '▄', '▅', '▆', '▇', '█']

function blockBar(value: number, max: number): string {
  const level = Math.round((value / max) * (BLOCKS.length - 1))
  return BLOCKS[level]
}

// Horizontal bar chart using left-block characters:
function hBar(value: number, max: number, width = 15): string {
  const LEFT_BLOCKS = ['▏', '▎', '▍', '▌', '▋', '▊', '▉', '█']
  const fullBlocks = Math.floor((value / max) * width)
  const remainder = ((value / max) * width) - fullBlocks
  const partialIdx = Math.floor(remainder * LEFT_BLOCKS.length)
  return '█'.repeat(fullBlocks) + (remainder > 0 ? LEFT_BLOCKS[partialIdx] : '') + '░'.repeat(width - fullBlocks - (remainder > 0 ? 1 : 0))
}
```

### Pattern 4: Page Indicator

```typescript
function pageIndicator(current: number, total: number): string {
  // Dot-style: ● ● ○ ○ ○
  return Array.from({ length: total }, (_, i) =>
    i === current ? '●' : '○'
  ).join(' ')
}

// Arrow-style: ← 3 / 7 →
function arrowIndicator(current: number, total: number): string {
  const left = current > 0 ? '←' : ' '
  const right = current < total - 1 ? '→' : ' '
  return `${left} ${current + 1} / ${total} ${right}`
}
```

### Pattern 5: Multi-Column Layout

```typescript
// Align two columns using fixed-width padding
function twoColumn(labels: string[], values: string[], labelWidth = 16): string {
  return labels.map((label, i) => {
    const padded = label.padEnd(labelWidth, ' ')
    return `${padded}${values[i] ?? ''}`
  }).join('\n')
}

// Usage:
const content = twoColumn(
  ['Temperature', 'Humidity', 'Wind', 'UV Index'],
  ['18°C', '65%', '12 km/h NW', '3 (Moderate)'],
  14
)
// → "Temperature   18°C"
//   "Humidity      65%"
//   "Wind          12 km/h NW"
//   "UV Index      3 (Moderate)"
```

> ⚠️ The G2 font is **not monospaced** — different characters have different widths. Column alignment using `padEnd` is approximate. Test on real hardware or the simulator to verify alignment.

### Pattern 6: Header + Body Layout

```typescript
// Split 576×288 canvas into header (top) + body (bottom)
const HEADER_HEIGHT = 48
const BODY_HEIGHT = 288 - HEADER_HEIGHT  // 240

// Header container
new TextContainerProperty({
  containerID: 1, containerName: 'header',
  content: 'My App  ●  18:42',
  xPosition: 0, yPosition: 0,
  width: 576, height: HEADER_HEIGHT,
  borderWidth: 1, borderColor: 5,
  paddingLength: 6,
  isEventCapture: 0,
})

// Body container (event capture here)
new TextContainerProperty({
  containerID: 2, containerName: 'body',
  content: bodyContent,
  xPosition: 0, yPosition: HEADER_HEIGHT,
  width: 576, height: BODY_HEIGHT,
  borderWidth: 0, borderColor: 0,
  paddingLength: 8,
  isEventCapture: 1,
})
```

### Pattern 7: Status Line with Battery

```typescript
function statusLine(battery: number, isWearing: boolean, extra = ''): string {
  const bat = battery >= 80 ? '█████' :
              battery >= 60 ? '████░' :
              battery >= 40 ? '███░░' :
              battery >= 20 ? '██░░░' : '█░░░░'
  const wear = isWearing ? '◉' : '○'
  return `${wear} ${bat} ${battery}%  ${extra}`.trim()
}
```

---

## Unicode Cheat Sheet for G2 Apps

The G2 firmware font supports these Unicode ranges. Use them freely in text containers.

## Progress & Charts

| Char | Use |
|------|-----|
| `━` U+2501 | Thick horizontal line (filled bar) |
| `─` U+2500 | Thin horizontal line (empty bar) |
| `█▇▆▅▄▃▂▁` U+2588–U+2581 | Full → 1/8 block (vertical bar levels) |
| `▉▊▋▌▍▎▏` U+2589–U+258F | Left-side partial blocks (horizontal fill) |
| `░▒▓` U+2591–U+2593 | Light / medium / dark shade (empty bar fill) |
| `●` U+25CF | Filled circle (selected page dot) |
| `○` U+25CB | Empty circle (unselected page dot) |
| `◉` U+25C9 | Bullseye (wearing indicator) |

### Separators & Dividers

| Char | Use |
|------|-----|
| `━━━━━` U+2501 | Thick horizontal rule |
| `─────` U+2500 | Thin horizontal rule |
| `═════` U+2550 | Double horizontal rule |
| `│` U+2502 | Vertical separator |
| `║` U+2551 | Double vertical separator |
| `┼` U+253C | Cross junction |
| `╋` U+254B | Heavy cross junction |

### Box Drawing (for bordered sections)

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Content here                                                           │
└─────────────────────────────────────────────────────────────────────────┘

╔═════════════════════════════════════════════════════════════════════════╗
║  Double-border section                                                  ║
╚═════════════════════════════════════════════════════════════════════════╝
```

| Char | Position |
|------|----------|
| `┌` U+250C | Top-left corner |
| `┐` U+2510 | Top-right corner |
| `└` U+2514 | Bottom-left corner |
| `┘` U+2518 | Bottom-right corner |
| `─` U+2500 | Top/bottom edge |
| `│` U+2502 | Left/right edge |
| `╔╗╚╝` | Double-line corners |
| `═` U+2550 | Double-line horizontal |
| `║` U+2551 | Double-line vertical |

### Arrows & Navigation

| Char | Use |
|------|-----|
| `←` U+2190 | Back / previous page |
| `→` U+2192 | Forward / next page |
| `↑` U+2191 | Up |
| `↓` U+2193 | Down |
| `▲` U+25B2 | Up (filled) |
| `▼` U+25BC | Down (filled) |
| `◀` U+25C0 | Left (filled) |
| `▶` U+25B6 | Right (filled) / play |

### Status & Icons

| Char | Use |
|------|-----|
| `✓` U+2713 | Checkmark / done |
| `✗` U+2717 | Cross / error |
| `★` U+2605 | Filled star (rating) |
| `☆` U+2606 | Empty star (rating) |
| `⚡` U+26A1 | Lightning / power |
| `🔋` | Battery (may not render — use block chars instead) |
| `◎` U+25CE | Target / focus |
| `⊙` U+2299 | Sun / brightness |
| `⊗` U+2297 | Error / close |
| `⊕` U+2295 | Add / plus |

> ⚠️ **Emoji support is unreliable on G2.** Stick to Unicode block characters (U+2500–U+25FF, U+2600–U+26FF) for reliable rendering. Test any character on the simulator before shipping.

### Typography Tips

```typescript
// Section header with separator
function sectionHeader(title: string, width = 30): string {
  const line = '━'.repeat(width)
  return `${title}\n${line}`
}

// Key-value pair with dot leader
function dotLeader(key: string, value: string, totalWidth = 28): string {
  const dots = '.'.repeat(Math.max(1, totalWidth - key.length - value.length))
  return `${key}${dots}${value}`
}

// Examples:
sectionHeader('Weather', 20)
// → "Weather\n━━━━━━━━━━━━━━━━━━━━"

dotLeader('Temperature', '18°C', 24)
// → "Temperature.........18°C"
```

---

## Simulator Usage Guide

The Even Hub Simulator lets you develop and test apps without physical glasses.

### Option A: BxNxM Unified Simulator

```bash
# Clone the simulator
git clone https://github.com/BxNxM/even-dev
cd even-dev

# Install dependencies
npm install

# Start simulator (serves on http://localhost:3000)
npm start

# In a second terminal: start your app dev server
cd my-app
npm run dev  # serves on http://localhost:5173
```

Open `http://localhost:3000` in your browser. The simulator shows:
- A virtual G2 display (576×288px)
- TouchBar buttons (single tap, double tap, scroll up, scroll down)
- R1 ring button simulation
- Console output from your app

### Option B: evenhub-cli QR Code (Real Device)

```bash
# Generate QR code pointing to your dev server
npx evenhub qr --http --port 5173

# Scan with Even App on your phone
# Your app loads on the glasses immediately
```

### Simulator vs Real Hardware Differences

| Behaviour | Simulator | Real Hardware |
|-----------|-----------|---------------|
| Event source | `sysEvent` | `textEvent` or `listEvent` |
| Click event type | `CLICK_EVENT` | `undefined` (for text containers) |
| Image rendering | Instant | ~200–500ms per image |
| Font rendering | Browser font | Firmware LVGL font |
| Column alignment | Accurate | Slightly different (non-monospace) |
| `createStartUpPageContainer` | Reusable | One-time only per session |
| Audio | Simulated | Real LC3 → PCM pipeline |
| Battery info | Mock values | Real battery level |

> ⚠️ **Always handle both `sysEvent` and `textEvent`/`listEvent`** in your event handler. The simulator sends `sysEvent` for button clicks; real hardware sends `textEvent` (for text containers) or `listEvent` (for list containers). Your event handler must check both.

### Simulator Event Mapping

```typescript
// Correct pattern — handles both simulator and real hardware
bridge.onEvenHubEvent((event) => {
  // Real hardware: text container events
  if (event.textEvent) {
    handleTextEvent(event.textEvent.eventType)
    return
  }

  // Real hardware: list container events
  if (event.listEvent) {
    handleListEvent(event.listEvent)
    return
  }

  // Simulator: sys events (also fires on real hardware for system events)
  if (event.sysEvent) {
    const { eventType } = event.sysEvent
    // Skip lifecycle events — only handle input events
    if (
      eventType === OsEventTypeList.FOREGROUND_ENTER_EVENT ||
      eventType === OsEventTypeList.FOREGROUND_EXIT_EVENT
    ) return
    handleTextEvent(eventType)
  }
})
```

---

## Packaging & Distribution

### Build and Package

```bash
# 1. Build production bundle
npm run build
# → Creates dist/ folder

# 2. Package as .ehpk
npx evenhub pack app.json dist -o myapp.ehpk
# → Creates myapp.ehpk

# 3. Install on device (via Even App)
# Share the .ehpk file → user opens it in Even App → installs
```

### `app.json` Full Reference

```json
{
  "package_id": "com.yourname.myapp",
  "edition": "202601",
  "name": "My App",
  "version": "1.0.0",
  "min_app_version": "0.1.0",
  "tagline": "One-line description shown in Even Hub store",
  "description": "Longer description. Shown on app detail page.",
  "author": "Your Name",
  "entrypoint": "index.html",
  "permissions": {
    "network": [
      "api.openai.com",
      "api.open-meteo.com"
    ]
  },
  "icons": {
    "48": "icons/icon-48.png",
    "96": "icons/icon-96.png"
  }
}
```

### `package_id` Rules

| Rule | Example |
|------|---------|
| Reverse-domain format | `com.yourname.myapp` ✅ |
| Each segment starts with lowercase letter | `com.1name.app` ❌ |
| Only lowercase letters and numbers | `com.yourName.myApp` ❌ |
| No hyphens | `com.your-name.my-app` ❌ |
| No underscores | `com.your_name.my_app` ❌ |
| Minimum 3 segments | `com.myapp` ❌ |

### `edition` Field

The `edition` field is a date string in `YYYYMM` format. It represents the SDK/platform edition your app targets:

```json
"edition": "202601"   // January 2026
```

Use the current year+month. This is not a version number — it tells Even Hub which platform features your app expects.

### Permissions

The `permissions.network` array lists the hostnames your app is allowed to contact. Requests to unlisted hosts will be blocked by the Even Hub WebView sandbox.

```json
"permissions": {
  "network": [
    "api.openai.com",
    "api.anthropic.com",
    "api.open-meteo.com",
    "wttr.in",
    "hacker-news.firebaseio.com"
  ]
}
```

> ⚠️ **Wildcard subdomains are not supported.** List each hostname explicitly. If your app calls `api.example.com` and `cdn.example.com`, list both.

### Version Bump Workflow

```bash
# 1. Update version in app.json
# 2. Update version in package.json (optional but good practice)
# 3. Rebuild and repackage
npm run build
npx evenhub pack app.json dist -o myapp-v1.1.0.ehpk
```

---

## Local Storage Patterns

> ⚠️ **Use `bridge.getLocalStorage` / `bridge.setLocalStorage` — NOT `window.localStorage`.** The WebView's `localStorage` is not persisted across sessions on the G2. The SDK's local storage is.

### Basic Storage

```typescript
// Save user preferences
async function savePrefs(bridge: EvenAppBridge, prefs: object) {
  await bridge.setLocalStorage({
    key: 'prefs',
    value: JSON.stringify(prefs),
  })
}

// Load user preferences
async function loadPrefs(bridge: EvenAppBridge): Promise<object | null> {
  const result = await bridge.getLocalStorage({ key: 'prefs' })
  if (!result?.value) return null
  try {
    return JSON.parse(result.value)
  } catch {
    return null
  }
}
```

### Typed Storage Helper

```typescript
class AppStorage {
  constructor(private bridge: EvenAppBridge) {}

  async get<T>(key: string, defaultValue: T): Promise<T> {
    const result = await this.bridge.getLocalStorage({ key })
    if (!result?.value) return defaultValue
    try {
      return JSON.parse(result.value) as T
    } catch {
      return defaultValue
    }
  }

  async set<T>(key: string, value: T): Promise<void> {
    await this.bridge.setLocalStorage({
      key,
      value: JSON.stringify(value),
    })
  }

  async remove(key: string): Promise<void> {
    await this.bridge.setLocalStorage({ key, value: '' })
  }
}

// Usage:
const storage = new AppStorage(bridge)
const city = await storage.get<string>('city', 'London')
await storage.set('city', 'Tokyo')
```

### Storage Limits

| Property | Value |
|----------|-------|
| Max key length | Not officially documented — keep under 64 chars |
| Max value length | Not officially documented — keep under 4KB |
| Persistence | Survives app restarts and phone reboots |
| Scope | Per app (isolated by `package_id`) |
| Type | String only — serialize objects with `JSON.stringify` |

---

## Error Handling Patterns

### `createStartUpPageContainer` Return Codes

```typescript
const result = await bridge.createStartUpPageContainer(config)

switch (result) {
  case 0:
    console.log('Success')
    break
  case 1:
    console.error('Invalid config — check container properties')
    break
  case 2:
    console.error('Oversize — total container area exceeds canvas')
    break
  case 3:
    console.error('Out of memory — reduce number of containers or content size')
    break
  default:
    console.error('Unknown error:', result)
}
```

### Defensive Rendering

```typescript
async function safeRender(bridge: EvenAppBridge, content: string): Promise<boolean> {
  try {
    const result = await bridge.textContainerUpgrade({
      containerID: 1,
      containerName: 'main',
      content: content.slice(0, 2000),  // guard against oversized content
    })
    return result === 0
  } catch (err) {
    console.error('Render failed:', err)
    return false
  }
}
```

### Network Error Display

```typescript
async function fetchWithFallback<T>(
  url: string,
  fallback: T,
  bridge: EvenAppBridge
): Promise<T> {
  try {
    const resp = await fetch(url)
    if (!resp.ok) throw new Error(`HTTP ${resp.status}`)
    return await resp.json() as T
  } catch (err) {
    // Show error on glasses
    await bridge.textContainerUpgrade({
      containerID: 1,
      containerName: 'main',
      content: `⚠ Network error\n\n${String(err).slice(0, 100)}\n\nDouble-tap to retry`,
    })
    return fallback
  }
}
```

### Reconnection Handling

```typescript
bridge.onDeviceStatusChanged(async (status) => {
  if (status.isDisconnected() || status.isConnectionFailed()) {
    // Stop all timers
    if (state.refreshInterval) {
      clearInterval(state.refreshInterval)
      state.refreshInterval = null
    }
    // Stop microphone if active
    if (state.isListening) {
      state.isListening = false
      bridge.audioControl(false).catch(() => {})
    }
    console.log('Glasses disconnected')
  }

  if (status.isConnected()) {
    console.log(`Reconnected. Battery: ${status.batteryLevel}%`)
    // Re-render current page after reconnect
    // Note: createStartUpPageContainer cannot be called again —
    // use textContainerUpgrade or rebuildPageContainer instead
    await renderCurrentPage(bridge, state)
    // Restart refresh timer
    state.refreshInterval = setInterval(() => renderCurrentPage(bridge, state), 30_000)
  }
})
```

> ⚠️ **After reconnect, do NOT call `createStartUpPageContainer` again.** It is a one-time call per session. Use `textContainerUpgrade` or `rebuildPageContainer` to refresh content after reconnection.

---

## Performance Patterns

### Debounce Scroll Events

```typescript
// Scroll events can fire rapidly — debounce to avoid render queue buildup
let lastScrollTime = 0
const SCROLL_COOLDOWN_MS = 300

function shouldHandleScroll(): boolean {
  const now = Date.now()
  if (now - lastScrollTime < SCROLL_COOLDOWN_MS) return false
  lastScrollTime = now
  return true
}
```

### Throttle API Calls

```typescript
let lastFetchTime = 0
const FETCH_COOLDOWN_MS = 60_000  // 1 minute minimum between fetches

async function throttledFetch(url: string): Promise<any> {
  const now = Date.now()
  if (now - lastFetchTime < FETCH_COOLDOWN_MS) {
    console.log('Throttled — using cached data')
    return null  // caller uses cached state
  }
  lastFetchTime = now
  const resp = await fetch(url)
  return resp.json()
}
```

### Image Update Queue (Never Concurrent)

```typescript
// Images MUST be sent sequentially — never two at once
class ImageQueue {
  private queue: Array<() => Promise<void>> = []
  private running = false

  enqueue(task: () => Promise<void>) {
    this.queue.push(task)
    this.drain()
  }

  private async drain() {
    if (this.running || this.queue.length === 0) return
    this.running = true
    while (this.queue.length > 0) {
      const task = this.queue.shift()!
      try { await task() } catch (e) { console.error('Image task failed:', e) }
    }
    this.running = false
  }
}

const imageQueue = new ImageQueue()

function sendImage(bridge: EvenAppBridge, base64: string) {
  imageQueue.enqueue(() =>
    bridge.updateImageRawData(new ImageRawDataUpdate({
      containerID: 2,
      containerName: 'canvas',
      imageData: base64,
    }))
  )
}
```

### Prefer `textContainerUpgrade` over `rebuildPageContainer`

```typescript
// SLOW: full page rebuild — causes visible flicker, resets scroll position
await bridge.rebuildPageContainer(new RebuildPageContainer({ ... }))

// FAST: in-place text update — no flicker, no layout recalculation
await bridge.textContainerUpgrade({
  containerID: 1,
  containerName: 'main',
  content: newContent,
})

// Rule of thumb:
// - Changing TEXT CONTENT only → textContainerUpgrade
// - Changing LAYOUT (positions, sizes, borders, container count) → rebuildPageContainer
// - Switching between LIST and TEXT views → rebuildPageContainer
```

---

## Community App Reference

These open-source apps demonstrate real-world patterns. Study them as reference implementations.

### Chess App — `dmyster145/EvenChess`

**URL:** https://github.com/dmyster145/EvenChess  
**Pattern:** Image canvas app (game), complex state machine, Canvas API rendering

Key techniques:
- Full chess board rendered as 200×100px PNG image
- Game state encoded in compact string for local storage
- Scroll = move cursor, tap = select/place piece
- Unicode chess pieces in text overlay container
- Stockfish WASM for AI opponent (loaded from CDN)

```typescript
// Chess board rendering pattern (simplified)
function renderBoard(board: ChessBoard): string {
  const canvas = document.createElement('canvas')
  canvas.width = 200; canvas.height = 100
  const ctx = canvas.getContext('2d')!

  const CELL_W = 200 / 8  // 25px per cell
  const CELL_H = 100 / 8  // 12.5px per cell

  for (let row = 0; row < 8; row++) {
    for (let col = 0; col < 8; col++) {
      // Checkerboard pattern
      ctx.fillStyle = (row + col) % 2 === 0 ? '#FFFFFF' : '#888888'
      ctx.fillRect(col * CELL_W, row * CELL_H, CELL_W, CELL_H)

      // Piece
      const piece = board[row][col]
      if (piece) {
        ctx.fillStyle = piece.color === 'white' ? '#FFFFFF' : '#000000'
        ctx.font = `${CELL_H * 0.8}px serif`
        ctx.textAlign = 'center'
        ctx.fillText(PIECE_CHARS[piece.type], col * CELL_W + CELL_W/2, row * CELL_H + CELL_H * 0.85)
      }
    }
  }

  return canvas.toDataURL('image/png').replace(/^data:image\/png;base64,/, '')
}
```

### Reddit Client — `fuutott/rdt-even-g2-rddit-client`

**URL:** https://github.com/fuutott/rdt-even-g2-rddit-client  
**Pattern:** Multi-page list + detail, external API, paginated content

Key techniques:
- List container for subreddit post list
- Text container for post detail (paginated)
- Reddit JSON API (no auth required for public posts)
- Local storage for saved subreddits
- Scroll cooldown to prevent rapid page flipping

```typescript
// Reddit API pattern (no auth needed for public data)
async function fetchSubreddit(subreddit: string, limit = 10) {
  const url = `https://www.reddit.com/r/${subreddit}/hot.json?limit=${limit}`
  const resp = await fetch(url, {
    headers: { 'User-Agent': 'EvenG2App/1.0' }
  })
  const data = await resp.json()
  return data.data.children.map((c: any) => ({
    title: c.data.title,
    score: c.data.score,
    author: c.data.author,
    numComments: c.data.num_comments,
    selftext: c.data.selftext,
    url: c.data.url,
  }))
}

// Post list display
function formatPostList(posts: any[]): string[] {
  return posts.map((p, i) =>
    `${i + 1}. ${p.title.slice(0, 40)}${p.title.length > 40 ? '…' : ''}`
  )
}

// Post detail display (paginated)
function formatPostDetail(post: any): string {
  const header = `${post.title}\n\n↑${post.score}  💬${post.numComments}  u/${post.author}\n\n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n\n`
  const body = post.selftext || '[Link post — no text content]'
  return header + body
}
```

### Weather App — `nickustinov/weather-even-g2`

**URL:** https://github.com/nickustinov/weather-even-g2  
**Pattern:** Multi-screen dashboard, multi-column layout, image icons, no backend

Key techniques:
- 5 screens navigated by swipe (weekly, current, precipitation, wind, hourly)
- Open-Meteo API (free, no API key, CORS-enabled)
- Weather icons drawn on Canvas API
- Multi-column text layout using fixed-width padding
- Location stored in local storage

```typescript
// Open-Meteo API (free, no key required)
async function fetchWeather(lat: number, lon: number) {
  const url = new URL('https://api.open-meteo.com/v1/forecast')
  url.searchParams.set('latitude', String(lat))
  url.searchParams.set('longitude', String(lon))
  url.searchParams.set('current', 'temperature_2m,relative_humidity_2m,wind_speed_10m,wind_direction_10m,weather_code')
  url.searchParams.set('daily', 'temperature_2m_max,temperature_2m_min,precipitation_sum,weather_code')
  url.searchParams.set('hourly', 'precipitation,wind_speed_10m')
  url.searchParams.set('forecast_days', '7')
  url.searchParams.set('timezone', 'auto')

  const resp = await fetch(url.toString())
  return resp.json()
}
```

### Pong Game — Community

**Pattern:** Real-time game loop, image canvas, IMU input

Key techniques:
- `setInterval` game loop at ~10 FPS (100ms)
- Ball and paddle positions updated each frame
- Canvas rendered to base64 PNG each frame
- IMU head tilt controls paddle position
- Image queue ensures frames never overlap

```typescript
// Game loop pattern
const GAME_FPS = 10
let gameLoopInterval: ReturnType<typeof setInterval> | null = null

function startGameLoop(bridge: EvenAppBridge, state: GameState) {
  if (gameLoopInterval) clearInterval(gameLoopInterval)
  gameLoopInterval = setInterval(async () => {
    updatePhysics(state)
    const frame = renderGameFrame(state)
    sendImage(bridge, frame)  // uses ImageQueue — never concurrent
  }, 1000 / GAME_FPS)
}

function stopGameLoop() {
  if (gameLoopInterval) {
    clearInterval(gameLoopInterval)
    gameLoopInterval = null
  }
}

// IMU input for paddle control
bridge.onEvenHubEvent((event) => {
  if (event.sysEvent?.imuData) {
    const { pitch } = event.sysEvent.imuData
    // Map head tilt to paddle position
    state.paddleY = Math.max(0, Math.min(IMG_H - PADDLE_H,
      IMG_H / 2 + pitch * 2
    ))
  }
})
```

### Snake Game — Community

**Pattern:** Grid-based game, keyboard-style input via TouchBar

Key techniques:
- Grid state stored as 2D array
- Scroll up/down = turn left/right
- Tap = pause/resume
- Double-tap = restart
- Score displayed in text container overlay

```typescript
// Snake direction mapping
type Direction = 'UP' | 'DOWN' | 'LEFT' | 'RIGHT'

function turnLeft(dir: Direction): Direction {
  const turns: Record<Direction, Direction> = {
    UP: 'LEFT', LEFT: 'DOWN', DOWN: 'RIGHT', RIGHT: 'UP'
  }
  return turns[dir]
}

function turnRight(dir: Direction): Direction {
  const turns: Record<Direction, Direction> = {
    UP: 'RIGHT', RIGHT: 'DOWN', DOWN: 'LEFT', LEFT: 'UP'
  }
  return turns[dir]
}

// Input mapping
bridge.onEvenHubEvent((event) => {
  const t = event.textEvent?.eventType ?? event.sysEvent?.eventType
  if (t === OsEventTypeList.SCROLL_BOTTOM_EVENT) state.direction = turnRight(state.direction)
  if (t === OsEventTypeList.SCROLL_TOP_EVENT) state.direction = turnLeft(state.direction)
  if (t === OsEventTypeList.CLICK_EVENT || t === undefined) state.paused = !state.paused
  if (t === OsEventTypeList.DOUBLE_CLICK_EVENT) resetGame(state)
})
```

---

## Complete App Checklist

Use this checklist before shipping any Even Hub app:

### Initialization
- [ ] `waitForEvenAppBridge()` called before any SDK method
- [ ] `onLaunchSource` registered immediately after bridge ready
- [ ] `createStartUpPageContainer` called only once per session
- [ ] Return code of `createStartUpPageContainer` checked (0 = success)
- [ ] Phone UI shown when `source !== 'glassesMenu'`

### Containers
- [ ] Exactly one container has `isEventCapture: 1`
- [ ] No more than 4 containers per page
- [ ] No more than 8 text containers total across all pages
- [ ] All containers fit within 576×288px canvas
- [ ] No overlapping containers (unless intentional — image over text)
- [ ] Image containers are max 200×100px

### Events
- [ ] Both `textEvent` and `sysEvent` handled (for simulator compatibility)
- [ ] Both `listEvent` and `sysEvent` handled for list containers
- [ ] Scroll cooldown implemented (300ms minimum)
- [ ] Lifecycle events (`FOREGROUND_ENTER_EVENT`, `FOREGROUND_EXIT_EVENT`) handled
- [ ] Double-tap has a defined action (back, refresh, or toggle)

### Images
- [ ] Image updates are queued (never concurrent)
- [ ] Image data is base64 PNG (strip `data:image/png;base64,` prefix)
- [ ] Image dimensions ≤ 200×100px
- [ ] Canvas drawn in greyscale (colour is discarded)

### Audio
- [ ] `audioControl(false)` called in `FOREGROUND_EXIT_EVENT` handler
- [ ] `audioControl(false)` called on disconnect
- [ ] PCM frames accumulated correctly (40 bytes/frame, 16kHz S16LE)
- [ ] Auto-stop at 30 seconds maximum

### Storage
- [ ] `bridge.getLocalStorage` / `bridge.setLocalStorage` used (not `window.localStorage`)
- [ ] Values serialized with `JSON.stringify` / `JSON.parse`
- [ ] Missing keys handled gracefully (default values)

### Networking
- [ ] All hostnames listed in `app.json` `permissions.network`
- [ ] Network errors caught and displayed on glasses
- [ ] API keys stored server-side (not in client bundle)
- [ ] Fetch calls throttled (avoid hammering APIs)

### Packaging
- [ ] `package_id` follows reverse-domain format (no hyphens, no uppercase)
- [ ] `edition` set to current year+month (`YYYYMM`)
- [ ] `entrypoint` points to correct HTML file
- [ ] `npm run build` succeeds before packaging
- [ ] `.ehpk` file tested on real device before distribution

### Performance
- [ ] `textContainerUpgrade` used for text-only updates (not `rebuildPageContainer`)
- [ ] `rebuildPageContainer` used only when layout changes
- [ ] Refresh intervals cleared in `FOREGROUND_EXIT_EVENT`
- [ ] Refresh intervals restarted in `FOREGROUND_ENTER_EVENT`
- [ ] No memory leaks (event listeners not duplicated on re-render)

---

## Quick Reference: Method Decision Tree

```
Need to update content on glasses?
│
├── Only changing TEXT in an existing container?
│   └── → bridge.textContainerUpgrade({ containerID, containerName, content })
│
├── Changing LAYOUT (positions, sizes, borders) or switching container types?
│   └── → bridge.rebuildPageContainer(new RebuildPageContainer({ ... }))
│
├── First render of the session?
│   └── → bridge.createStartUpPageContainer(new CreateStartUpPageContainer({ ... }))
│       ⚠️ ONE TIME ONLY — never call again after first render
│
└── Updating an IMAGE in an existing image container?
    └── → bridge.updateImageRawData(new ImageRawDataUpdate({ containerID, containerName, imageData }))
        ⚠️ Queue these — never send two image updates concurrently
```

---

## Key Numbers to Memorise

| Value | Meaning |
|-------|---------|
| `576 × 288` | Full canvas size in pixels |
| `200 × 100` | Max image container size in pixels |
| `4` | Max containers per page |
| `8` | Max text containers total |
| `1` | Exactly one `isEventCapture: 1` per page |
| `0` | Success return code from `createStartUpPageContainer` |
| `40` | Bytes per PCM audio frame (10ms at 16kHz S16LE) |
| `16000` | PCM sample rate (Hz) |
| `300` | Recommended scroll cooldown (ms) |
| `30` | Max microphone recording duration (seconds) |
| `0.0.9` | Current SDK version (as of April 2026) |
| `202601` | Example `edition` field value |

---