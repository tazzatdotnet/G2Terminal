# 02 — SDK Quickstart

> **Source**: Official npm packages, `nickustinov/even-g2-notes`, `BxNxM/even-dev`, official GitHub repos. All facts verified April 2026.
> **SDK Version**: `@evenrealities/even_hub_sdk` v0.0.9

---

## Prerequisites

- **Node.js** 18+ and npm
- **iPhone** with the **Even App** installed (for testing on device)
- **Even Realities G2 glasses** (for hardware testing)
- A code editor (VS Code recommended)
- Basic TypeScript/JavaScript knowledge

You do **not** need Xcode, Android Studio, or any native mobile toolchain for Path A (WebView apps).

---

## Step 1 — Create Your Project

```bash
npm create vite@latest my-even-app -- --template vanilla-ts
cd my-even-app
npm install
```

Any framework works: React, Vue, Svelte, vanilla TS. Vite is the standard build tool for Even Hub apps.

---

## Step 2 — Install the SDK

```bash
npm install @evenrealities/even_hub_sdk
```

Install the CLI as a dev dependency (for QR sideloading and packaging):

```bash
npm install -D @evenrealities/evenhub-cli
```

---

## Step 3 — Configure Vite

Update `vite.config.ts` to expose your dev server on the local network (required for QR sideloading):

```typescript
import { defineConfig } from 'vite'

export default defineConfig({
  server: {
    host: true,       // expose on 0.0.0.0 (all interfaces)
    port: 5173,
  },
})
```

---

## Step 4 — Minimal Project Structure

```
my-even-app/
├── index.html          ← entry point
├── package.json
├── vite.config.ts
├── app.json            ← required for packaging/publishing
├── tsconfig.json
└── src/
    ├── main.ts         ← app bootstrap
    └── styles.css
```

**`index.html`** — standard HTML, loads your TypeScript entry:

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

**`package.json`** — recommended scripts:

```json
{
  "name": "my-even-app",
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
    "@evenrealities/evenhub-cli": "^0.1.11",
    "typescript": "^5.0.0",
    "vite": "^5.0.0"
  }
}
```

---

## Step 5 — Minimal Working App ("Hello Glasses")

This is the smallest possible Even Hub app that displays text on the glasses.

**`src/main.ts`**:

```typescript
import {
  waitForEvenAppBridge,
  CreateStartUpPageContainer,
  TextContainerProperty,
} from '@evenrealities/even_hub_sdk'

async function main() {
  // 1. Wait for the bridge — ALWAYS await this, never skip it
  const bridge = await waitForEvenAppBridge()

  // 2. Listen for launch source — fires ONCE after WebView loads
  bridge.onLaunchSource((source) => {
    console.log('Launch source:', source)
    // 'appMenu'     = opened from phone app menu
    // 'glassesMenu' = opened from glasses menu (initialize glasses UI here)
    if (source === 'glassesMenu') {
      initGlassesUI(bridge)
    }
  })
}

async function initGlassesUI(bridge: Awaited<ReturnType<typeof waitForEvenAppBridge>>) {
  // 3. Create the initial page — call this EXACTLY ONCE per session
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
          content: 'Hello, G2! 👓',
          isEventCapture: 1,           // exactly ONE container must be 1
        }),
      ],
    })
  )

  console.log('createStartUpPageContainer result:', result)
  // 0 = success, 1 = invalid, 2 = oversize, 3 = outOfMemory
}

main().catch(console.error)
```

---

## Step 6 — Run the Simulator

The Even Hub Simulator lets you develop without physical glasses.

```bash
# Run the simulator against your dev server
npx @evenrealities/evenhub-simulator@latest http://localhost:5173

# Or install globally
npm install -g @evenrealities/evenhub-simulator
evenhub-simulator http://localhost:5173
```

Start your dev server first, then launch the simulator:

```bash
# Terminal 1
npm run dev

# Terminal 2
npx @evenrealities/evenhub-simulator@latest http://localhost:5173
```

The simulator opens a desktop window showing the G2 display (576×288px). You can click to simulate tap events and use keyboard shortcuts to simulate ring input.

> ⚠️ **Simulator limitations**: Image size constraints are not enforced, font rendering differs, `imuData` is always `null`, `eventSource` is hardcoded to `1`, status events are hardcoded. Always test on real hardware before publishing.

---

## Step 7 — Test on Real Glasses (QR Sideload)

```bash
# Terminal 1 — start dev server
npm run dev

# Terminal 2 — generate QR code
npm run qr
# or: npx evenhub qr --url "http://192.168.x.x:5173"
```

> ⚠️ Use your machine's **local network IP** (e.g. `192.168.1.42`), not `localhost`. Vite prints the network URL when starting with `--host 0.0.0.0`.

1. Open the **Even App** on your iPhone
2. Tap the QR scan button
3. Scan the QR code in your terminal
4. Your app loads on the glasses

Vite hot reload works — code changes reflect immediately without rescanning the QR code.

---

## Step 8 — Add `app.json` (Required for Publishing)

```bash
npx evenhub init
# Creates ./app.json with template values
```

Or create it manually:

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
    "network": ["api.example.com"],
    "fs": ["./assets"]
  }
}
```

**`package_id` rules:**
- Reverse-domain format: `com.yourname.appname`
- Each segment: starts with lowercase letter, contains only lowercase letters or numbers
- **No hyphens** — `com.my-name.my-app` is **invalid**
- Valid: `com.myname.myapp`, `com.john123.weatherapp`

Use `["*"]` for `permissions.network` if your app connects to user-configured or dynamic servers.

---

## Step 9 — Package for Distribution

```bash
npm run pack
# Runs: vite build && evenhub pack app.json dist -o myapp.ehpk
```

Or manually:

```bash
npx vite build
npx evenhub pack app.json dist -o myapp.ehpk
```

Add `*.ehpk` to your `.gitignore`.

---

## Complete App Skeleton (Production-Ready Pattern)

This is the recommended starting point for any real Even Hub app:

```typescript
import {
  waitForEvenAppBridge,
  EvenAppBridge,
  CreateStartUpPageContainer,
  RebuildPageContainer,
  TextContainerProperty,
  ListContainerProperty,
  ListItemContainerProperty,
  ImageContainerProperty,
  ImageRawDataUpdate,
  TextContainerUpgrade,
  OsEventTypeList,
  ImuReportPace,
  DeviceConnectType,
} from '@evenrealities/even_hub_sdk'

// ─── App State ────────────────────────────────────────────────────────────────

let bridge: EvenAppBridge | null = null
let isPageCreated = false
const storageCache = new Map<string, string>()

// ─── Storage Helpers ──────────────────────────────────────────────────────────

async function initStorage(keys: string[]) {
  if (!bridge) return
  await Promise.all(keys.map(async (key) => {
    const value = await bridge!.getLocalStorage(key)
    if (value) storageCache.set(key, value)
  }))
}

function getItem(key: string): string | null {
  return storageCache.get(key) ?? null
}

function setItem(key: string, value: string): void {
  storageCache.set(key, value)
  void bridge?.setLocalStorage(key, value).catch(() => {})
}

// ─── UI Helpers ───────────────────────────────────────────────────────────────

async function showPage(text: string) {
  if (!bridge) return

  const container = new TextContainerProperty({
    containerID: 1,
    containerName: 'main',
    xPosition: 0,
    yPosition: 0,
    width: 576,
    height: 288,
    borderWidth: 0,
    borderColor: 5,
    paddingLength: 8,
    content: text,
    isEventCapture: 1,
  })

  if (!isPageCreated) {
    const result = await bridge.createStartUpPageContainer(
      new CreateStartUpPageContainer({ containerTotalNum: 1, textObject: [container] })
    )
    if (result === 0) isPageCreated = true
  } else {
    await bridge.rebuildPageContainer(
      new RebuildPageContainer({ containerTotalNum: 1, textObject: [container] })
    )
  }
}

// ─── Event Handler ────────────────────────────────────────────────────────────

function handleEvent(event: any) {
  const { listEvent, textEvent, sysEvent } = event

  // Determine event type — CLICK_EVENT = 0 becomes undefined in SDK
  const eventType = listEvent?.eventType ?? textEvent?.eventType ?? sysEvent?.eventType
  const isClick = eventType === OsEventTypeList.CLICK_EVENT || eventType === undefined

  if (listEvent && isClick) {
    const index = listEvent.currentSelectItemIndex ?? 0
    const name = listEvent.currentSelectItemName
    console.log('List item selected:', index, name)
    // Handle navigation here
  }

  if (textEvent) {
    if (eventType === OsEventTypeList.SCROLL_TOP_EVENT) {
      console.log('Scrolled to top')
    }
    if (eventType === OsEventTypeList.SCROLL_BOTTOM_EVENT) {
      console.log('Scrolled to bottom — load next page')
    }
  }
}

// ─── Main ─────────────────────────────────────────────────────────────────────

async function main() {
  // 1. Get bridge
  bridge = await waitForEvenAppBridge()

  // 2. Pre-load storage
  await initStorage(['lastPage', 'preferences'])

  // 3. Monitor device status
  bridge.onDeviceStatusChanged((status) => {
    console.log('Device status:', status)
    if (status.connectType === DeviceConnectType.connected) {
      console.log('Glasses connected, battery:', status.batteryLevel)
    }
  })

  // 4. Register event handler
  bridge.onEvenHubEvent(handleEvent)

  // 5. Auto-connect (no manual button required)
  // The bridge handles connection automatically — just initialize UI on launch

  // 6. Handle launch source
  bridge.onLaunchSource(async (source) => {
    if (source === 'glassesMenu') {
      await showPage('Loading...')
      // Initialize your app here
      await showPage('Welcome to My App\n\nSwipe to navigate')
    }
  })
}

main().catch(console.error)
```

---

## SDK Initialization: Two Methods

```typescript
import { waitForEvenAppBridge, EvenAppBridge } from '@evenrealities/even_hub_sdk'

// Method 1: Async wait (RECOMMENDED) — resolves when bridge is ready
const bridge = await waitForEvenAppBridge()

// Method 2: Synchronous singleton — ONLY use after bridge is already initialized
// (e.g. in a helper function called after main() has awaited waitForEvenAppBridge)
const bridge = EvenAppBridge.getInstance()
```

> ⚠️ Never call `EvenAppBridge.getInstance()` at module load time or before `waitForEvenAppBridge()` has resolved. The bridge is injected by the native app and is not available synchronously on page load.

---

## Page Lifecycle: The Three Rules

| Rule | Detail |
|------|--------|
| `createStartUpPageContainer` | Called **exactly once** per session. Establishes the initial page. |
| `rebuildPageContainer` | Called for **all subsequent page changes**. Same parameter shape as `createStartUpPageContainer`. Causes a full redraw (brief flicker on hardware). |
| `textContainerUpgrade` | **Partial text update** — updates text in an existing container without a full rebuild. Faster, flicker-free on hardware. |

```typescript
// First call — initialize
await bridge.createStartUpPageContainer(new CreateStartUpPageContainer({ ... }))

// All subsequent calls — update/navigate
await bridge.rebuildPageContainer(new RebuildPageContainer({ ... }))

// Fast text update (no full rebuild)
await bridge.textContainerUpgrade(new TextContainerUpgrade({
  containerID: 1,
  containerName: 'main',
  contentOffset: 0,
  contentLength: 100,
  content: 'Updated text',
}))
```

---

## `onLaunchSource` Pattern

`onLaunchSource` fires **exactly once** after the WebView finishes loading. Always register it early.

```typescript
bridge.onLaunchSource((source) => {
  if (source === 'glassesMenu') {
    // User opened the app from the glasses menu
    // This is the correct time to initialize the glasses UI
    initGlassesUI()
  } else if (source === 'appMenu') {
    // User opened the app from the phone app menu
    // Show phone-side UI only, or also initialize glasses
    initPhoneUI()
  }
})
```

| Value | Meaning |
|-------|---------|
| `'glassesMenu'` | App opened from the glasses menu — initialize glasses UI |
| `'appMenu'` | App opened from the phone app menu |

---

## Kotlin Multiplatform Alternative

If you prefer Kotlin/Multiplatform, the official `EH-InNovel` demo shows how to use the SDK from Kotlin:

- **Repo**: [github.com/even-realities/EH-InNovel](https://github.com/even-realities/EH-InNovel)
- Uses **Compose Multiplatform** for the phone UI
- Integrates the Even Hub SDK via Kotlin JS interop
- Demonstrates a full novel reader app with page navigation on the glasses

The SDK API is identical — `waitForEvenAppBridge`, `createStartUpPageContainer`, `rebuildPageContainer`, etc. — just called from Kotlin instead of TypeScript.

---

## Development Workflow Summary

```
1. npm run dev          → start Vite dev server (keep running)
2. npm run qr           → generate QR code for phone sideload
   (or: npx @evenrealities/evenhub-simulator@latest http://localhost:5173 for simulator)
3. Scan QR with Even App → app loads on glasses
4. Edit code → Vite hot reload → changes appear immediately
5. npm run pack         → build + package into .ehpk for distribution
```

---

## even-dev: Multi-App Development Environment

For building and testing multiple apps simultaneously, use the community `even-dev` environment:

```bash
git clone https://github.com/BxNxM/even-dev.git
cd even-dev
npm install
./start-even.sh
```

Features:
- Discovers all apps in the `apps/` directory and registered in `apps.json`
- Auto-installs dependencies for each app
- Launches Vite dev server + simulator together
- Supports external apps via git URL or local path in `apps.json`
- Auto-detects and starts backend servers in `server/` subdirectories
- Supports `vite-plugin.ts` in app root for custom Vite config (Tailwind, path aliases, etc.)

Register external apps:

```json
{
  "chess": "https://github.com/dmyster145/EvenChess",
  "my-local-app": "../my-local-app"
}
```

---

## Browser/Phone UI: even-toolkit

The phone-side settings/config page (the WebView UI the user sees on their phone) can use **even-toolkit** — a React component library built specifically for Even Hub apps:

```bash
npm install even-toolkit
```

```typescript
import { Button, Card, NavBar, Toggle, AppShell } from 'even-toolkit/web'
import { useGlasses } from 'even-toolkit/useGlasses'
```

Features: 55+ React components, 191 pixel-art icons, design tokens, glasses SDK bridge helpers, text pagination utilities, canvas renderer, and more. See `08_COMMUNITY_APPS.md` for full details.

---