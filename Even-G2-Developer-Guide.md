# Building and Deploying Apps for the Even Realities G2 Smart Glasses

**A complete guide from zero to published app — macOS edition**

*SDK v0.0.9 · CLI v0.1.11 · Simulator v0.6.2 · April 2026*

---

## Part 1 — What You're Building For

Before you write a line of code, internalize what the G2 actually is and what it cannot do. Skipping this section leads to wasted hours designing things the hardware will reject.

### The Hardware

The G2 is a pair of smart glasses with a green micro-LED display projected into your field of view. There is no camera, no speaker, no GPS. The display is driven entirely by your iPhone over Bluetooth 5.2.

| Spec | Value |
|------|-------|
| Display resolution | 576 × 288 pixels per eye |
| Color | Green micro-LED, 4-bit greyscale (16 shades) |
| Font | Single LVGL firmware font — **not monospaced**, no size/weight/family control |
| Text alignment | Left-aligned, top-aligned only |
| Background | Always black (pixels off = black on micro-LED) |
| Input | TouchBar on right temple (tap, double-tap, swipe), optional R1 ring |
| Connectivity | Bluetooth 5.2 to iPhone |
| Camera | None |
| Speaker | None |

### The Software Model

An Even Hub app is a **standard web application** — HTML, CSS, TypeScript — that runs inside a WKWebView on the user's iPhone. You never compile native code. The phone app communicates with the glasses over Bluetooth; the SDK gives you a JavaScript bridge to control the glasses display and receive input events.

**There is no CSS, no DOM, no flexbox on the glasses.** The glasses display is not a browser. You define rectangular containers at absolute pixel coordinates, and the glasses firmware renders them. Your web app's DOM lives on the phone only — it handles your app logic, networking, and state. The glasses are a remote display.

### Hard Constraints to Accept Now

1. **No monospace font.** Characters like `i` and `l` are narrower than `m` and `W`. Column-aligned output will not align. Accept this.
2. **No background colors or images at page creation time.** The display background is always black. Use Unicode box-drawing characters (U+2500–U+25FF) for visual structure.
3. **No font size, bold, italic, or underline.** One font, one size, one weight.
4. **No text centering or right-alignment.** Pad with spaces if you need to fake it.
5. **Max 4 containers per page** (mixed types), max 8 text containers.
6. **`createStartUpPageContainer` is called exactly once per session.** All subsequent updates use `textContainerUpgrade` (flicker-free, up to 2,000 chars) or `rebuildPageContainer` (full layout change, up to 1,000 chars).
7. **`window.localStorage` does not persist** inside the `.ehpk` WebView. Use `bridge.setLocalStorage()` / `bridge.getLocalStorage()` instead.
8. **Visible text capacity is ~400–500 characters** on a full-screen container.

---

## Part 2 — Setting Up Your macOS Development Environment

### 2.1 Prerequisites

You need:

- **Node.js 18+** and npm. Install via [nvm](https://github.com/nvm-sh/nvm) or [nodejs.org](https://nodejs.org).
- **A code editor.** VS Code is the community standard.
- **An iPhone** with the **Even App** installed (for device testing).
- **Even Realities G2 glasses** (for hardware testing — the simulator covers early development).

You do **not** need Xcode, Android Studio, Swift, Kotlin, or any native mobile toolchain.

Verify your Node version:

```bash
node --version   # must be 18+
npm --version
```

### 2.2 Create the Project

```bash
npm create vite@latest my-even-app -- --template vanilla-ts
cd my-even-app
npm install
```

Any framework works (React, Vue, Svelte, vanilla TS). Vite is the standard build tool for Even Hub apps.

### 2.3 Install the SDK and CLI

```bash
# The SDK — your bridge to the glasses
npm install @evenrealities/even_hub_sdk

# The CLI — for QR sideloading, packaging, and publishing
npm install -D @evenrealities/evenhub-cli
```

### 2.4 Configure Vite for Network Access

Your iPhone needs to reach your dev server over the local network. Update `vite.config.ts`:

```typescript
import { defineConfig } from 'vite'

export default defineConfig({
  server: {
    host: true,    // binds to 0.0.0.0 — exposes on all network interfaces
    port: 5173,
  },
})
```

### 2.5 Set Up package.json Scripts

Replace the default scripts in `package.json`:

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

### 2.6 Project Structure

```
my-even-app/
├── index.html          ← entry point (loaded in WKWebView)
├── package.json
├── vite.config.ts
├── app.json            ← required for packaging/publishing
├── tsconfig.json
└── src/
    ├── main.ts         ← app bootstrap
    └── styles.css      ← phone-side UI styles (not glasses)
```

Create `index.html`:

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

---

## Part 3 — Your First App: "Hello, G2!"

This is the smallest possible app that puts text on the glasses display.

### 3.1 `src/main.ts`

```typescript
import {
  waitForEvenAppBridge,
  CreateStartUpPageContainer,
  TextContainerProperty,
} from '@evenrealities/even_hub_sdk'

async function main() {
  // Step 1: Wait for the bridge. This MUST be awaited.
  // It resolves when the WebView ↔ Even App connection is established.
  const bridge = await waitForEvenAppBridge()

  // Step 2: Listen for the launch source. This fires ONCE.
  // You MUST register this early — if you register late, you miss it.
  bridge.onLaunchSource((source) => {
    console.log('Launch source:', source)

    if (source === 'glassesMenu') {
      // User opened the app from the glasses menu.
      // This is when you initialize the glasses display.
      initGlassesUI(bridge)
    } else if (source === 'appMenu') {
      // User opened from the phone app menu.
      // Show phone-side UI, or also init glasses — your choice.
      console.log('Opened from phone menu')
    }
  })
}

async function initGlassesUI(
  bridge: Awaited<ReturnType<typeof waitForEvenAppBridge>>
) {
  // Step 3: Create the startup page. Call this EXACTLY ONCE per session.
  // Calling it again will fail or cause undefined behavior.
  const result = await bridge.createStartUpPageContainer(
    new CreateStartUpPageContainer({
      containerTotalNum: 1,     // MUST match the actual number of containers
      textObject: [
        new TextContainerProperty({
          containerID: 1,
          containerName: 'main',    // max 16 characters
          xPosition: 0,
          yPosition: 0,
          width: 576,               // full display width
          height: 288,              // full display height
          borderWidth: 0,
          borderColor: 5,
          paddingLength: 8,
          content: 'Hello, G2! 👓',
          isEventCapture: 1,        // exactly ONE container per page must be 1
        }),
      ],
    })
  )

  // Return codes: 0=success, 1=invalid config, 2=oversize, 3=out of memory
  if (result !== 0) {
    console.error('Page creation failed with code:', result)
  }
}

main().catch(console.error)
```

### 3.2 Key Concepts in This Code

**`waitForEvenAppBridge()`** — Always the first thing you call. It returns a promise that resolves to the bridge object. Never skip the await.

**`onLaunchSource`** — Fires once to tell you how the user opened your app. If the source is `'glassesMenu'`, the user is wearing the glasses and expects to see something on the display. If it's `'appMenu'`, they opened it from the phone — you might show a settings screen or also initialize the glasses.

**`createStartUpPageContainer`** — Creates the initial display layout. You call this **once, ever, per session.** The `containerTotalNum` field must exactly match the number of container objects you pass. Exactly one container must have `isEventCapture: 1` — this is the container that receives touch/scroll/ring input events.

**`textContainerUpgrade`** — After the initial page is created, use this for all text updates. It's flicker-free, allows up to 2,000 characters, and does not require a full page rebuild. This is the workhorse API for live-data apps.

---

## Part 4 — Using the Simulator

The simulator lets you develop and iterate without touching physical glasses. There are two options.

### 4.1 Option A: Official `evenhub-simulator` (Quick Start)

Start your dev server, then launch the simulator in a second terminal:

```bash
# Terminal 1 — start Vite dev server
npm run dev

# Terminal 2 — launch the simulator pointing at your dev server
npx @evenrealities/evenhub-simulator@latest http://localhost:5173
```

The simulator opens a desktop window showing the 576×288 G2 display. You can click to simulate tap events and use keyboard shortcuts for ring input.

To install it globally instead of using `npx` each time:

```bash
npm install -g @evenrealities/evenhub-simulator
evenhub-simulator http://localhost:5173
```

### 4.2 Option B: `even-dev` Community Environment (Multi-App)

BxNxM's `even-dev` provides a unified launcher for developing and testing multiple apps simultaneously:

```bash
git clone https://github.com/BxNxM/even-dev.git
cd even-dev
npm install
./start-even.sh
```

This discovers all apps in the `apps/` directory, auto-installs their dependencies, and launches both the Vite dev server and simulator together. Open `http://localhost:3000` in your browser to see the virtual G2 display with TouchBar buttons and R1 ring simulation.

### 4.3 Simulator Limitations — Read This Carefully

The simulator diverges from real hardware in important ways. Code that works perfectly in the simulator can break on the glasses.

| Behavior | Simulator | Real Hardware |
|----------|-----------|---------------|
| Event source for clicks | `sysEvent` | `textEvent` or `listEvent` |
| Click event type | `CLICK_EVENT` | `undefined` (for text containers) |
| Image rendering speed | Instant | ~200–500ms per image |
| Font rendering | Browser font (monospace-ish) | Firmware LVGL font (proportional) |
| Column alignment | Accurate | Misaligned (non-monospace) |
| `createStartUpPageContainer` | Can be called multiple times | One-time only per session |
| IMU data | Always `null` | Real accelerometer data |
| Battery info | Mock values | Real battery level |
| Audio | Simulated | Real LC3 → PCM pipeline |

**The #1 gotcha:** In the simulator, click events come through `event.sysEvent`. On real hardware, they come through `event.textEvent` (for text containers) or `event.listEvent` (for list containers). Your event handler must check all three:

```typescript
bridge.onEvenHubEvent((event) => {
  // Real hardware: text container events
  if (event.textEvent) {
    handleInput(event.textEvent.eventType)
    return
  }

  // Real hardware: list container events
  if (event.listEvent) {
    handleListInput(event.listEvent)
    return
  }

  // Simulator: sys events (also fires for lifecycle events on real hardware)
  if (event.sysEvent) {
    const { eventType } = event.sysEvent
    // Skip lifecycle events
    if (
      eventType === OsEventTypeList.FOREGROUND_ENTER_EVENT ||
      eventType === OsEventTypeList.FOREGROUND_EXIT_EVENT
    ) return
    handleInput(eventType)
  }
})
```

**The #2 gotcha:** On real hardware, `eventType` for a click on a text container is `undefined`, not `CLICK_EVENT`. Always check for both:

```typescript
function handleInput(eventType: number | undefined) {
  if (eventType === 0 || eventType === undefined) {
    // This is a click/tap
  }
}
```

---

## Part 5 — Testing on Real Glasses (QR Sideloading)

Once your app works in the simulator, test it on actual hardware. The development loop is remarkably fast — no compile step, no app store, just scan a QR code.

### 5.1 Prerequisites

- Your Mac and iPhone must be on the **same local network** (same Wi-Fi).
- The Even App must be installed on your iPhone.
- Your G2 glasses must be paired with the Even App.
- Your macOS firewall must allow inbound connections on port 5173.

### 5.2 Generate the QR Code

```bash
# Terminal 1 — keep your dev server running
npm run dev

# Terminal 2 — generate the QR code
npm run qr
# This runs: evenhub qr --http --port 5173
```

The CLI auto-detects your local network IP and generates a QR code in the terminal. If it picks the wrong network interface:

```bash
npx evenhub qr --http --host 192.168.1.42 --port 5173
```

Vite also prints the network URL when it starts — look for the `Network:` line:

```
  VITE v5.x.x  ready in 200ms

  ➜  Local:   http://localhost:5173/
  ➜  Network: http://192.168.1.42:5173/    ← this is the IP your phone needs
```

### 5.3 Scan and Load

1. Open the **Even App** on your iPhone.
2. Tap the **QR scan button** (look in the Developer section).
3. Scan the QR code displayed in your terminal.
4. Your app loads on the glasses immediately.

**Hot reload works.** Edit your code, save, and changes appear on the glasses without rescanning the QR code. Vite's HMR pushes updates to the WebView in real time.

### 5.4 Troubleshooting Sideload Failures

If the app doesn't load after scanning:

**Phone can't reach your dev server.** Make sure both devices are on the same Wi-Fi. Check that your firewall isn't blocking port 5173. On macOS: System Settings → Network → Firewall. Or verify the port is open:

```bash
lsof -i :5173
```

**QR code points to the wrong IP.** If your Mac has multiple network interfaces (Ethernet + Wi-Fi + Tailscale), the CLI might pick the wrong one. Specify the IP manually with `--host`.

**HTTPS required for microphone access.** If your app uses the microphone, you must use `--https` instead of `--http`:

```bash
npx evenhub qr --https --port 5173
```

Accept the self-signed certificate on your iPhone when prompted.

---

## Part 6 — Understanding the Display System

This is where Even Hub development differs from anything you've done before. Spend time here.

### 6.1 The Canvas

The glasses display is 576×288 pixels. Origin is top-left `(0, 0)`. X increases rightward, Y increases downward. Everything is 4-bit greyscale — 16 shades of green. Black means the pixel is off.

### 6.2 Container Types

You build the display from containers — rectangular regions positioned with absolute pixel coordinates.

**Text containers** (`TextContainerProperty`) — The workhorse. Renders plain text with automatic word wrapping. If the content overflows and the container has `isEventCapture: 1`, the firmware enables internal scrolling. Content limit: 1,000 chars at creation, 2,000 chars via `textContainerUpgrade`.

**List containers** (`ListContainerProperty`) — A native scrollable list widget. The firmware handles highlighting and scroll behavior. You provide items as an array. Best for menus and selection screens.

**Image containers** (`ImageContainerProperty`) — Displays greyscale images. Size limit: 20–200px width, 20–100px height. Images must be base64-encoded. Push images sequentially — never concurrently.

### 6.3 The Update Lifecycle

```
App starts
    │
    ▼
createStartUpPageContainer()   ← called ONCE, creates initial layout
    │
    ▼
textContainerUpgrade()         ← called many times, updates text in-place
    │                             flicker-free, up to 2,000 chars
    ▼
rebuildPageContainer()         ← called when layout structure changes
    │                             causes brief flicker, up to 1,000 chars
    ▼
(repeat upgrade / rebuild as needed)
```

**Best practice:** Call `createStartUpPageContainer` once at startup. Use `textContainerUpgrade` for all subsequent text changes. Only use `rebuildPageContainer` when you need to change the number or position of containers.

### 6.4 Unicode for Visual Structure

Since you can't draw backgrounds or lines, use Unicode characters to build visual structure:

```
Separators:       ━━━━━━━━━━━━━━━━━━━━
Box drawing:      ┌─────────┐
                  │  Menu   │
                  └─────────┘
Scroll indicators: ▲ ▼
Connection status: ● Connected   ○ Disconnected
Progress bars:     ████░░░░ 50%
Dot leaders:       Temperature.........18°C
```

Stick to the U+2500–U+25FF and U+2600–U+26FF ranges for reliable rendering. Always test a character in the simulator before shipping it.

---

## Part 7 — Handling Events and Input

### 7.1 Event Sources

The G2 has two physical inputs: the TouchBar on the right temple and the optional R1 ring. At the SDK level, both generate identical event types — you cannot distinguish between them.

Events you'll use most often:

| Event | Trigger |
|-------|---------|
| Click / Tap | Single tap on TouchBar or R1 ring single-click |
| Double tap | Double-tap on TouchBar or R1 ring double-click |
| Long press | Long press on TouchBar or R1 ring long-press |
| Scroll up | Swipe forward on TouchBar |
| Scroll down | Swipe backward on TouchBar |
| Scroll top reached | User scrolled to the top boundary of content |
| Scroll bottom reached | User scrolled to the bottom boundary of content |

### 7.2 Lifecycle Events

These fire through `sysEvent` on both simulator and real hardware:

| Event | When |
|-------|------|
| `FOREGROUND_ENTER_EVENT` | App becomes visible on glasses |
| `FOREGROUND_EXIT_EVENT` | App goes to background — stop timers, save state, release audio |
| `ABNORMAL_EXIT_EVENT` | Unexpected disconnect — clean up everything |

### 7.3 Scroll Cooldown

Scroll events fire rapidly. Without a cooldown, a single swipe triggers multiple scroll actions:

```typescript
let lastScrollTime = 0
const SCROLL_COOLDOWN_MS = 300

function handleScroll(direction: 'up' | 'down') {
  const now = Date.now()
  if (now - lastScrollTime < SCROLL_COOLDOWN_MS) return
  lastScrollTime = now

  if (direction === 'up') scrollUp()
  else scrollDown()
}
```

---

## Part 8 — The `app.json` Manifest

Every Even Hub app needs an `app.json` at the project root. This tells the CLI how to package your app and tells Even Hub how to display it in the store.

### 8.1 Generate a Template

```bash
npx evenhub init
```

Or create it manually:

```json
{
  "package_id": "com.yourname.myapp",
  "edition": "202604",
  "name": "My App",
  "version": "1.0.0",
  "min_app_version": "0.1.0",
  "tagline": "Short description shown in Even Hub",
  "description": "Longer description of what the app does.",
  "author": "Your Name",
  "entrypoint": "index.html",
  "permissions": {
    "network": ["api.example.com"],
    "fs": ["./assets"]
  }
}
```

### 8.2 `package_id` Rules

This is strict. Invalid IDs will be rejected.

| Rule | Example |
|------|---------|
| Reverse-domain format, minimum 3 segments | `com.yourname.myapp` ✅ |
| Each segment starts with a lowercase letter | `com.1name.app` ❌ |
| Only lowercase letters and digits | `com.yourName.myApp` ❌ |
| No hyphens | `com.your-name.my-app` ❌ |
| No underscores | `com.your_name.my_app` ❌ |
| Two segments only | `com.myapp` ❌ |

Validation regex: `/^[a-z][a-z0-9]*(\.[a-z][a-z0-9]*)+$/`

### 8.3 `edition` Field

A date string in `YYYYMM` format representing the SDK/platform edition your app targets. Use the current year and month:

```json
"edition": "202604"
```

### 8.4 Network Permissions

The `permissions.network` array lists hostnames your app is allowed to contact. Requests to unlisted hosts are blocked by the Even Hub WebView sandbox.

```json
"permissions": {
  "network": ["api.openai.com", "api.open-meteo.com"]
}
```

Use `["*"]` if your app connects to user-configured or dynamic servers (e.g., an SSH terminal that connects to arbitrary hosts).

### 8.5 Icon Requirements

| Property | Value |
|----------|-------|
| Format | PNG |
| Size | 512 × 512 px |
| Background | Transparent or solid |
| Safe zone | Keep content within inner 400 × 400 px |

---

## Part 9 — Packaging and Deploying

### 9.1 Build the Production Bundle

```bash
npm run build
# → Creates dist/ folder with your compiled app
```

### 9.2 Package as `.ehpk`

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

### 9.3 Validate the Package

```bash
npx evenhub validate myapp.ehpk
```

This checks package integrity, manifest validity, and entrypoint existence.

### 9.4 Inspect Package Metadata

```bash
npx evenhub info myapp.ehpk
```

### 9.5 Install on Device

Share the `.ehpk` file to the user's iPhone. When opened in the Even App, it installs the app on the glasses.

### 9.6 Submit to Even Hub

Upload your `.ehpk` to the Even Hub portal at [hub.evenrealities.com](https://hub.evenrealities.com). The review process and timeline are not yet publicly documented — ask in the Discord if you're ready to submit.

### 9.7 Bundle Size Optimization

For production, optimize your Vite build:

```typescript
// vite.config.ts
import { defineConfig } from 'vite'

export default defineConfig({
  server: {
    host: true,
    port: 5173,
  },
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
})
```

---

## Part 10 — Production-Ready App Skeleton

This is the recommended starting template for any real Even Hub app. It handles the bridge lifecycle, storage, events, and display updates correctly.

```typescript
import {
  waitForEvenAppBridge,
  EvenAppBridge,
  CreateStartUpPageContainer,
  TextContainerProperty,
  TextContainerUpgrade,
  OsEventTypeList,
} from '@evenrealities/even_hub_sdk'

// ─── State ────────────────────────────────────────────────────────────────────

let bridge: Awaited<ReturnType<typeof waitForEvenAppBridge>> | null = null
let isPageCreated = false
const storageCache = new Map<string, string>()

// ─── Storage (bridge-backed, not window.localStorage) ─────────────────────────

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
  if (bridge) void bridge.setLocalStorage(key, value).catch(() => {})
}

// ─── Display ──────────────────────────────────────────────────────────────────

async function createPage() {
  if (!bridge || isPageCreated) return
  const result = await bridge.createStartUpPageContainer(
    new CreateStartUpPageContainer({
      containerTotalNum: 1,
      textObject: [
        new TextContainerProperty({
          containerID: 1,
          containerName: 'main',
          xPosition: 0, yPosition: 0,
          width: 576, height: 288,
          borderWidth: 0, borderColor: 5,
          paddingLength: 8,
          content: 'Loading...',
          isEventCapture: 1,
        }),
      ],
    })
  )
  if (result === 0) isPageCreated = true
  else console.error('Page creation failed:', result)
}

async function updateDisplay(text: string) {
  if (!bridge || !isPageCreated) return
  await bridge.textContainerUpgrade(
    new TextContainerUpgrade({
      containerID: 1,
      containerName: 'main',
      content: text.slice(0, 2000),  // hard limit
    })
  )
}

// ─── Events ───────────────────────────────────────────────────────────────────

function setupEvents() {
  if (!bridge) return

  bridge.onEvenHubEvent((event) => {
    // Text container events (real hardware)
    if (event.textEvent) {
      handleInput(event.textEvent.eventType)
      return
    }

    // List container events (real hardware)
    if (event.listEvent) {
      // handle list events if you use list containers
      return
    }

    // System events (simulator + lifecycle on real hardware)
    if (event.sysEvent) {
      const { eventType } = event.sysEvent
      if (eventType === OsEventTypeList.FOREGROUND_EXIT_EVENT) {
        // App going to background — save state, stop timers
        return
      }
      if (eventType === OsEventTypeList.FOREGROUND_ENTER_EVENT) {
        // App returning to foreground — resume
        return
      }
      handleInput(eventType)
    }
  })
}

function handleInput(eventType: number | undefined) {
  // Click: eventType is 0 on simulator, undefined on real hardware
  if (eventType === 0 || eventType === undefined) {
    console.log('Tap detected')
  }
  // Add scroll, double-tap, long-press handlers as needed
}

// ─── Bootstrap ────────────────────────────────────────────────────────────────

async function main() {
  bridge = await waitForEvenAppBridge()

  bridge.onLaunchSource(async (source) => {
    if (source === 'glassesMenu') {
      await initStorage(['settings', 'lastState'])
      await createPage()
      setupEvents()
      await updateDisplay('Ready.')
    }
  })
}

main().catch(console.error)
```

---

## Part 11 — CI/CD with GitHub Actions

Automate your build and packaging:

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
      - run: npm run build

      - name: Package
        run: |
          npm install -g @evenrealities/evenhub-cli
          evenhub pack app.json dist -o app-${{ github.sha }}.ehpk
          evenhub validate app-${{ github.sha }}.ehpk

      - uses: actions/upload-artifact@v4
        with:
          name: even-hub-package
          path: '*.ehpk'
          retention-days: 30
```

---

## Part 12 — The Complete Workflow, End to End

Here's the full loop, from blank terminal to app running on glasses:

```
1.  npm create vite@latest my-app -- --template vanilla-ts
2.  cd my-app && npm install
3.  npm install @evenrealities/even_hub_sdk
4.  npm install -D @evenrealities/evenhub-cli
5.  Edit vite.config.ts (add host: true)
6.  Write your app in src/main.ts
7.  npm run dev                          ← start dev server
8.  npx @evenrealities/evenhub-simulator@latest http://localhost:5173
                                          ← test in simulator
9.  npm run qr                           ← generate QR for phone sideload
10. Scan QR with Even App on iPhone      ← app loads on glasses instantly
11. Iterate — Vite hot reloads on every save
12. Create app.json with your package ID, permissions, icon
13. npm run pack                         ← build + package into .ehpk
14. npx evenhub validate myapp.ehpk      ← verify package integrity
15. Upload .ehpk to hub.evenrealities.com ← submit for distribution
```

---

## Part 13 — Tips from the Trenches

**Start with `textContainerUpgrade` for everything.** Avoid `rebuildPageContainer` unless you're fundamentally changing the layout structure. `textContainerUpgrade` is flicker-free and handles up to 2,000 characters.

**Throttle display updates.** Buffer output on the phone side and render at 100–200ms intervals. Don't call SDK methods on every data event or you'll saturate the BLE link.

**Test on hardware early.** The simulator lies about fonts, timing, event sources, and image rendering. Don't build for months in the simulator.

**Design for ~400 characters visible.** Think of it as a 50-char × 12-line terminal. Paginate everything.

**Use Unicode creatively.** Box-drawing characters, block elements, and arrows are your only visual tools beyond text.

**Implement reconnection logic.** WebSocket and BLE connections will drop, especially when iOS backgrounds your app. Plan for it from day one.

**Don't store secrets on the phone.** `bridge.setLocalStorage` is the only persistence API, and its security properties are unknown. Keep API keys and credentials server-side behind a proxy.

**Join the Discord.** The community is small and responsive. Even Realities staff participate.

---

## Resources

| Resource | URL |
|----------|-----|
| Even Hub Portal | https://hub.evenrealities.com |
| Even Hub Developer Docs | https://hub.evenrealities.com/docs |
| SDK on npm | https://www.npmjs.com/package/@evenrealities/even_hub_sdk |
| CLI on npm | https://www.npmjs.com/package/@evenrealities/evenhub-cli |
| Simulator on npm | https://www.npmjs.com/package/@evenrealities/evenhub-simulator |
| Even Realities GitHub | https://github.com/even-realities |
| even-dev (community) | https://github.com/BxNxM/even-dev |
| Community SDK notes | https://github.com/nickustinov/even-g2-notes |
| BLE protocol (advanced) | https://github.com/i-soxi/even-g2-protocol |
