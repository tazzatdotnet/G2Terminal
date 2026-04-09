# G2 SSH Terminal — Prototype Design Document

> **Purpose:** Single-source reference for Claude Code sessions. Contains every architectural decision, module boundary, data flow, SDK constraint, and implementation detail needed to build the prototype without re-deriving context.

---

## 1. System Architecture

```
G2 Glasses ◄──BLE 5.2──► iPhone 15 (iOS 26) ◄──WSS/Tailscale──► Proxmox VM (Gateway)
                          │                                        │
                     Even Hub WebView                         ssh2 + ws (Node.js)
                     (all app logic)                          SSH to any tailnet host
```

### Component Responsibilities

| Component | Owns | Does NOT own |
|-----------|------|-------------|
| **G2 Glasses** | 576×288 4-bit greyscale display, touchpad (swipe/tap) | No compute, no storage, no networking |
| **iPhone WebView** | App logic, keyboard capture, scrollback buffer, WebSocket client, SDK bridge calls, display rendering | No raw TCP, no persistent secure storage |
| **Gateway VM** | WebSocket server, SSH client (`ssh2`), session management, SSH key storage, reconnection grace period | No display logic, no user input processing |

### Why This Architecture

WebViews cannot open raw TCP sockets. SSH requires TCP on port 22. A WebSocket-to-SSH proxy is **architecturally mandatory**. Tailscale provides encrypted transport; the WKWebView resolves Tailscale IPs (100.x.x.x) natively when Tailscale VPN is active on iOS.

---

## 2. Project Structure

```
g2-ssh-terminal/
├── phone-app/                    # Even Hub WebView app (Vite + TypeScript)
│   ├── index.html                # Minimal HTML — hidden input + status div
│   ├── app.json                  # Even Hub manifest
│   ├── vite.config.ts
│   ├── tsconfig.json
│   ├── package.json
│   └── src/
│       ├── main.ts               # Entry: bridge init, lifecycle, orchestration
│       ├── display.ts            # Glasses rendering (containers, viewport)
│       ├── input.ts              # Keyboard capture, key bindings, command history
│       ├── connection.ts         # WebSocket client, reconnection logic
│       ├── buffer.ts             # Scrollback buffer, line processing, ANSI stripping
│       ├── ansi.ts               # ANSI escape sequence parser/stripper
│       └── types.ts              # Shared types and constants
│
└── gateway/                      # Node.js WebSocket-to-SSH proxy
    ├── package.json
    ├── tsconfig.json
    └── src/
        ├── server.ts             # ws server, connection routing
        ├── ssh-session.ts        # ssh2 client wrapper, session lifecycle
        └── types.ts              # Protocol message types
```

---

## 3. Even Hub Manifest (`app.json`)

```json
{
  "package_id": "com.terminal.g2ssh",
  "edition": "202604",
  "name": "SSH Terminal",
  "version": "0.1.0",
  "min_app_version": "0.1.0",
  "tagline": "SSH terminal for G2 glasses",
  "description": "Keyboard-driven SSH terminal. Connects to a gateway proxy over WebSocket.",
  "author": "developer",
  "entrypoint": "index.html",
  "permissions": {
    "network": [
      "ws://100.x.x.x:8022",
      "wss://gateway.tailnet-name.ts.net:8022"
    ]
  }
}
```

> **Open question:** Does the whitelist accept Tailscale IPs, MagicDNS hostnames, wildcards, or CIDR? Test with an exact IP first, fall back to MagicDNS hostname.

---

## 4. Display Module (`display.ts`)

### Container Layout

Two containers, created once at startup via `createStartUpPageContainer`:

| Container | ID | Position | Size | Purpose |
|-----------|----|----------|------|---------|
| Status bar | 1 | 0, 0 | 576×24 | `user@host:~/path` + scroll/connection indicator |
| Main output | 2 | 0, 24 | 576×264 | Terminal output viewport, `isEventCapture: 1` |

### Startup Page Creation

```typescript
await bridge.createStartUpPageContainer(
  new CreateStartUpPageContainer({
    containerTotalNum: 2,
    textObject: [
      new TextContainerProperty({
        containerID: 1, containerName: 'status',
        xPosition: 0, yPosition: 0, width: 576, height: 24,
        borderWidth: 0, borderColor: 0, paddingLength: 2,
        content: 'Connecting...', isEventCapture: 0,
      }),
      new TextContainerProperty({
        containerID: 2, containerName: 'term',
        xPosition: 0, yPosition: 24, width: 576, height: 264,
        borderWidth: 0, borderColor: 0, paddingLength: 4,
        content: '', isEventCapture: 1,
      }),
    ],
  })
);
```

> **Call `createStartUpPageContainer` exactly ONCE.** Never again. All subsequent updates use `textContainerUpgrade`.

### Update Strategy

```typescript
// Flicker-free content update (preferred — use for ALL output changes)
await bridge.textContainerUpgrade(
  new TextContainerUpgrade({
    containerID: 2, containerName: 'term',
    content: renderedPageText,  // max 2000 chars
  })
);

// Status bar update (same method, different container)
await bridge.textContainerUpgrade(
  new TextContainerUpgrade({
    containerID: 1, containerName: 'status',
    content: statusLine,
  })
);
```

### Display Constants

```typescript
const DISPLAY_WIDTH = 576;
const DISPLAY_HEIGHT = 288;
const STATUS_BAR_HEIGHT = 24;
const TERM_HEIGHT = 264;
const MAX_UPGRADE_CHARS = 2000;
const VISIBLE_CHARS = 500;        // empirical estimate of visible text
const EST_CHARS_PER_LINE = 50;    // varies — not monospaced
const EST_VISIBLE_LINES = 12;     // ~264px / ~22px per line
const UPDATE_THROTTLE_MS = 150;   // minimum interval between display updates
```

### Throttled Render Loop

```typescript
let renderPending = false;
let lastRenderTime = 0;

function scheduleRender(): void {
  if (renderPending) return;
  renderPending = true;
  const elapsed = Date.now() - lastRenderTime;
  const delay = Math.max(0, UPDATE_THROTTLE_MS - elapsed);
  setTimeout(() => {
    renderPending = false;
    lastRenderTime = Date.now();
    renderViewport();
  }, delay);
}
```

> **Never call `textContainerUpgrade` on every WebSocket data event.** Always buffer and throttle.

### `rebuildPageContainer` — When and Only When

Use ONLY for layout structure changes (e.g., showing/hiding status bar, switching to a menu screen). It causes a brief flicker on hardware. Never use for content updates.

---

## 5. Input Module (`input.ts`)

### Core Principle

**The phone is operated entirely by keyboard. The user never looks at the phone screen.** All interaction flows through a single `keydown` handler on `document`.

### Hidden Input Field

```html
<!-- index.html -->
<input id="kbd" type="text" autocomplete="off" autocorrect="off"
       autocapitalize="off" spellcheck="false"
       style="position:fixed;top:0;left:0;opacity:0.01;width:1px;height:1px;" />
```

### Auto-Focus Logic

```typescript
const input = document.getElementById('kbd') as HTMLInputElement;

function refocus(): void {
  setTimeout(() => input.focus(), 50);
}

input.addEventListener('blur', refocus);
document.addEventListener('click', refocus);
window.addEventListener('focus', refocus);
// Initial focus on load
input.focus();
```

### Key Bindings

All handled in a single `document.addEventListener('keydown', handler)`:

| Key | Action | Implementation |
|-----|--------|----------------|
| `Enter` | Send `input.value` as command + `\n` to WebSocket, clear input, reset history index | `ws.send(JSON.stringify({ type: 'input', data: input.value + '\n' }))` |
| `Ctrl+C` | Send SIGINT (`\x03`) to remote | `ws.send(JSON.stringify({ type: 'input', data: '\x03' }))` |
| `Ctrl+D` | Send EOF (`\x04`) | `ws.send(JSON.stringify({ type: 'input', data: '\x04' }))` |
| `Tab` | Send tab char (`\t`) for shell completion | `ws.send(JSON.stringify({ type: 'input', data: '\t' }))` |
| `ArrowUp` | Previous command in history | Populate input field, `e.preventDefault()` |
| `ArrowDown` | Next command in history | Populate input field, `e.preventDefault()` |
| `Ctrl+L` | Clear glasses display + send clear to remote | Clear buffer, render empty, send `\x0c` |
| `Escape` | Send escape char (`\x1b`) | `ws.send(JSON.stringify({ type: 'input', data: '\x1b' }))` |

### Command History

```typescript
const history: string[] = [];
let historyIndex = -1;
const MAX_HISTORY = 200;

// On Enter: push to history[0], reset index
// On ArrowUp: increment index (clamp to history.length - 1), populate input
// On ArrowDown: decrement index (clamp to -1 = empty), populate input
// Optionally persist via bridge.setLocalStorage('cmdHistory', JSON.stringify(history))
```

---

## 6. Buffer Module (`buffer.ts`)

### Scrollback Buffer

```typescript
interface ScrollbackBuffer {
  lines: string[];              // processed (ANSI-stripped) lines
  maxLines: number;             // 1000–10000, configurable
  viewportOffset: number;       // 0 = pinned to bottom (live mode)
  liveMode: boolean;            // true = auto-scroll with new output
}
```

### Data Flow

```
WebSocket data event
  → raw SSH output (string, may contain partial lines)
  → accumulate in lineBuffer until \n or \r\n
  → strip ANSI escapes (ansi.ts)
  → word-wrap to ~50 chars per line (EST_CHARS_PER_LINE)
  → push to scrollback buffer
  → if liveMode: scheduleRender()
```

### Viewport Extraction

```typescript
function getViewportLines(buf: ScrollbackBuffer): string[] {
  const pageSize = EST_VISIBLE_LINES;
  if (buf.liveMode) {
    // Show last N lines
    return buf.lines.slice(-pageSize);
  }
  const start = Math.max(0, buf.lines.length - buf.viewportOffset - pageSize);
  const end = start + pageSize;
  return buf.lines.slice(start, end);
}

function renderViewport(): void {
  const lines = getViewportLines(buffer);
  const text = lines.join('\n');
  // Truncate to MAX_UPGRADE_CHARS if needed
  updateTermContainer(text.slice(0, MAX_UPGRADE_CHARS));
}
```

### Touchpad Scroll (via `onEvenHubEvent`)

| Glasses Event | Action |
|---------------|--------|
| Swipe up (`SCROLL_FORWARD_EVENT`) | `viewportOffset += pageSize / 2; liveMode = false; scheduleRender()` |
| Swipe down (`SCROLL_BACK_EVENT`) | `viewportOffset -= pageSize / 2; if (viewportOffset <= 0) { liveMode = true; viewportOffset = 0; } scheduleRender()` |
| Single press (`CLICK_EVENT`) | Snap to bottom: `liveMode = true; viewportOffset = 0; scheduleRender()` |
| Double press (`DOUBLE_CLICK_EVENT`) | Toggle menu (future: connection settings, disconnect, clear) |

> **Scroll cooldown:** Enforce 300ms minimum between scroll events to prevent over-scrolling.

---

## 7. ANSI Module (`ansi.ts`)

### V1: Lightweight Regex Stripper

```typescript
// Strips CSI sequences, OSC sequences, and basic SGR codes
const ANSI_RE = /\x1b\[[0-9;]*[A-Za-z]|\x1b\][^\x07]*\x07|\x1b\(B|\x1b\[[\?]?[0-9;]*[hl]/g;

export function stripAnsi(raw: string): string {
  return raw.replace(ANSI_RE, '');
}
```

### V2 (Future): Headless xterm.js

If edge cases become painful (tab completion rendering, progress bars, `apt` output), upgrade to xterm.js running headlessly on the phone. Maintain a virtual terminal buffer, extract visible text for display. This is architecturally heavier but handles 100% of terminal output correctly.

### What V1 Cannot Handle

- Cursor addressing (arbitrary `\x1b[row;colH` positioning) — vim, nano, htop, tmux
- Partial-line overwrites (progress bars that use `\r` without `\n`)
- Alternate screen buffer switching (`\x1b[?1049h`)

**Accept that TUI apps are unsupported in V1.** Document this clearly.

---

## 8. Connection Module (`connection.ts`)

### WebSocket Client

```typescript
interface ConnectionConfig {
  gatewayUrl: string;       // "wss://100.x.x.x:8022" or MagicDNS hostname
  targetHost: string;       // SSH target hostname/IP on tailnet
  targetPort: number;       // default 22
  username: string;         // SSH username on target
  authToken?: string;       // gateway authentication token
}

interface WSMessage {
  type: 'input' | 'resize' | 'auth';
  data: string;
  cols?: number;
  rows?: number;
}

interface GatewayMessage {
  type: 'output' | 'status' | 'error' | 'disconnect';
  data: string;
}
```

### Connection Lifecycle

```
1. User enters config (or loaded from bridge.getLocalStorage)
2. Open WebSocket to gateway
3. Send auth message: { type: 'auth', data: token, targetHost, targetPort, username }
4. Gateway opens SSH session to target
5. Gateway streams SSH stdout/stderr as { type: 'output', data: '...' }
6. Phone sends keystrokes as { type: 'input', data: '...' }
7. On close/error → enter reconnection loop
```

### Reconnection Logic

```typescript
const RECONNECT_DELAYS = [1000, 2000, 4000, 8000, 15000]; // backoff
let reconnectAttempt = 0;

function reconnect(): void {
  const delay = RECONNECT_DELAYS[Math.min(reconnectAttempt, RECONNECT_DELAYS.length - 1)];
  updateStatus('Reconnecting...');
  setTimeout(() => {
    reconnectAttempt++;
    connect(currentConfig);
  }, delay);
}

// On successful open:
// reconnectAttempt = 0;
```

### Persistence (Non-Sensitive Only)

```typescript
// Save connection preferences (NOT keys, NOT tokens)
bridge.setLocalStorage('sshConfig', JSON.stringify({
  gatewayUrl: config.gatewayUrl,
  targetHost: config.targetHost,
  targetPort: config.targetPort,
  username: config.username,
}));
```

---

## 9. Gateway Server (`gateway/`)

### Stack

- **Runtime:** Node.js 20+
- **Dependencies:** `ws` (WebSocket server), `ssh2` (SSH client)
- **Transport:** WSS over Tailscale (TLS even within the tunnel)
- **Port:** 8022

### Protocol (WebSocket JSON messages)

**Phone → Gateway:**

| `type` | Fields | Purpose |
|--------|--------|---------|
| `auth` | `token`, `targetHost`, `targetPort`, `username` | Initiate SSH session |
| `input` | `data` (string) | Keystrokes → SSH stdin |
| `resize` | `cols`, `rows` | Terminal resize (SIGWINCH) |

**Gateway → Phone:**

| `type` | Fields | Purpose |
|--------|--------|---------|
| `output` | `data` (string) | SSH stdout/stderr → phone |
| `status` | `data` (string) | Connection status messages |
| `error` | `data` (string) | Error messages |
| `disconnect` | `data` (string, reason) | Session ended |

### Session Management

```typescript
// Per WebSocket connection:
interface Session {
  ws: WebSocket;
  ssh: ssh2.Client;
  stream: ssh2.ClientChannel | null;
  targetHost: string;
  lastActivity: number;
  gracePeriodTimer: NodeJS.Timeout | null;  // for reconnection
}
```

### Reconnection Grace Period

When a WebSocket drops (iOS backgrounding), the gateway keeps the SSH session alive for 60 seconds. If the phone reconnects and sends a valid `auth` with the same target, the gateway reattaches to the existing SSH session instead of opening a new one. After 60 seconds with no reconnect, the SSH session is closed.

### SSH Key Management

- Private keys stored on the gateway VM filesystem only, never transmitted to the phone
- Gateway uses `ssh2` `privateKey` option when connecting to targets
- Gateway authenticates the phone via a simple bearer token (configured at gateway startup)

### Minimal Server Skeleton

```typescript
import { WebSocketServer } from 'ws';
import { Client as SSHClient } from 'ssh2';

const wss = new WebSocketServer({ port: 8022 });

wss.on('connection', (ws) => {
  let sshClient: SSHClient | null = null;

  ws.on('message', (raw) => {
    const msg = JSON.parse(raw.toString());
    switch (msg.type) {
      case 'auth':
        sshClient = new SSHClient();
        sshClient.on('ready', () => {
          sshClient!.shell({ term: 'xterm', cols: 50, rows: 12 }, (err, stream) => {
            if (err) { ws.send(JSON.stringify({ type: 'error', data: err.message })); return; }
            stream.on('data', (data: Buffer) => {
              ws.send(JSON.stringify({ type: 'output', data: data.toString() }));
            });
            stream.on('close', () => {
              ws.send(JSON.stringify({ type: 'disconnect', data: 'shell closed' }));
            });
            // Store stream reference for input forwarding
          });
        });
        sshClient.connect({
          host: msg.targetHost,
          port: msg.targetPort || 22,
          username: msg.username,
          privateKey: readFileSync('/path/to/key'),
        });
        break;
      case 'input':
        // Forward to SSH stream stdin
        break;
      case 'resize':
        // stream.setWindow(msg.rows, msg.cols, 0, 0);
        break;
    }
  });

  ws.on('close', () => {
    // Start grace period timer instead of immediate SSH close
  });
});
```

---

## 10. App Lifecycle (`main.ts`)

### Initialization Sequence

```
1. Import bridge from SDK
2. Wait for bridge.init(onEvenHubEvent)
3. Check launchSource (appMenu vs glassesMenu)
4. Create startup page (2 containers)
5. Load saved config from bridge.getLocalStorage
6. Connect WebSocket to gateway
7. Begin render loop
```

### Event Handler

```typescript
function onEvenHubEvent(event: string): void {
  const parsed = JSON.parse(event);

  // Lifecycle events
  if (parsed.sysEvent?.eventType === OsEventTypeList.FOREGROUND_ENTER_EVENT) {
    // Reconnect WebSocket if disconnected, resume render loop
  }
  if (parsed.sysEvent?.eventType === OsEventTypeList.FOREGROUND_EXIT_EVENT) {
    // Pause render loop, optionally close WebSocket cleanly
  }

  // Touchpad events (from term container, isEventCapture: 1)
  if (parsed.textEvent) {
    switch (parsed.textEvent.eventType) {
      case OsEventTypeList.SCROLL_FORWARD_EVENT: scrollUp(); break;
      case OsEventTypeList.SCROLL_BACK_EVENT: scrollDown(); break;
      case OsEventTypeList.CLICK_EVENT: snapToLive(); break;
      case OsEventTypeList.DOUBLE_CLICK_EVENT: toggleMenu(); break;
    }
  }

  // Fallback: simulator sends sysEvent for scroll/tap
  if (parsed.sysEvent) {
    // Handle same scroll/click events via sysEvent path
  }
}
```

### Focus Mode (Screen-Off Alternative)

True screen-off kills the WebView. Instead:

```typescript
// Request wake lock (may not work in WKWebView — test on device)
try {
  await navigator.wakeLock.request('screen');
} catch { /* fallback: silent audio loop or setInterval keepalive */ }
```

- Page background: `#000000` (zero power on OLED)
- All visible UI hidden
- Phone brightness manually set to minimum by user
- Phone placed face-down, keyboard on desk

---

## 11. Hard Constraints Checklist

These **cannot** be worked around. Every module must respect them.

| # | Constraint | Impact |
|---|-----------|--------|
| 1 | No monospace font | Column-aligned output (ls -la, docker ps, ps aux) will misalign. Accept it. |
| 2 | No TUI apps | vim, nano, tmux, htop, less, man — all unsupported. Use cat, grep, non-interactive alternatives. |
| 3 | No raw TCP from WebView | Gateway proxy is mandatory. |
| 4 | 2000 char upgrade limit | Only ~500 visible; buffer everything on phone, paginate for display. |
| 5 | 1000 char startup/rebuild limit | Initial page is minimal; content populated via upgrade. |
| 6 | No background colors/images at page creation | Terminal "background" is just empty display (black = off on micro-LED). |
| 7 | No font control (size, weight, family) | Single LVGL firmware font. Cannot load monospace. |
| 8 | WebSocket dies on iOS background | Must implement reconnection + gateway grace period. |
| 9 | `createStartUpPageContainer` called once only | All subsequent updates via `textContainerUpgrade` or `rebuildPageContainer`. |
| 10 | Exactly one `isEventCapture: 1` per page | Assigned to the main terminal container (ID 2). |
| 11 | Max 8 text containers per page | Using 2. Well within limit. |
| 12 | `bridge.setLocalStorage` only — no `window.localStorage` | Survives app restart; `window.localStorage` does not in `.ehpk` WebView. |

---

## 12. Implementation Phases

### Phase 1: Skeleton (proof of life)

**Goal:** Keyboard input on phone → text appears on glasses.

- [ ] Scaffold Vite + TS project with SDK dependency
- [ ] `index.html` with hidden input field
- [ ] `main.ts`: bridge init, create 2-container startup page
- [ ] `input.ts`: auto-focus, keydown handler, Enter sends text
- [ ] `display.ts`: `textContainerUpgrade` wrapper with throttle
- [ ] Hardcoded echo mode (type → see on glasses, no network)
- [ ] Test in simulator

### Phase 2: Gateway + Live SSH

**Goal:** Full SSH session from glasses through gateway.

- [ ] `gateway/server.ts`: ws + ssh2 proxy, JSON protocol
- [ ] `connection.ts`: WebSocket client, auth handshake
- [ ] Wire input.ts keystrokes → WebSocket → gateway → SSH stdin
- [ ] Wire SSH stdout → WebSocket → buffer → display
- [ ] `ansi.ts`: V1 regex stripper
- [ ] `buffer.ts`: scrollback buffer, line processing, word wrap
- [ ] Status bar showing `user@host:path`
- [ ] Test on simulator with real SSH session through gateway

### Phase 3: Robustness

**Goal:** Handle real-world conditions.

- [ ] Reconnection logic with exponential backoff
- [ ] Gateway grace period (60s session keepalive on WS drop)
- [ ] Lifecycle events: pause/resume on FOREGROUND_ENTER/EXIT
- [ ] Command history (ArrowUp/Down)
- [ ] Ctrl+C, Ctrl+D, Tab, Escape key bindings
- [ ] Scroll via touchpad (swipe up/down, tap to snap)
- [ ] Scroll cooldown (300ms)
- [ ] Config persistence via `bridge.setLocalStorage`
- [ ] Wake lock attempt + black background focus mode

### Phase 4: Polish & Hardware Testing

**Goal:** Validate on real G2 hardware. Tune empirical values.

- [ ] Calibrate EST_CHARS_PER_LINE and EST_VISIBLE_LINES on hardware
- [ ] Tune UPDATE_THROTTLE_MS (start 150ms, test lower)
- [ ] Test `textContainerUpgrade` rate limits (what happens at 10+/sec?)
- [ ] Check display ghosting on rapid updates
- [ ] Verify Tailscale IP in `app.json` whitelist works
- [ ] Test wake lock in WKWebView
- [ ] Test BLE stability during sustained terminal sessions
- [ ] Package as `.ehpk`, test sideload via QR

---

## 13. Key SDK Imports

```typescript
import {
  EvenAppBridge,
  CreateStartUpPageContainer,
  RebuildPageContainer,
  TextContainerProperty,
  TextContainerUpgrade,
  OsEventTypeList,
  LAUNCH_SOURCE_APP_MENU,
  LAUNCH_SOURCE_GLASSES_MENU,
} from '@evenrealities/even_hub_sdk';
```

---

## 14. Unicode UI Elements

No images or background colors at page creation. Use Unicode box drawing and block elements for visual structure:

```
Status bar separator:  ─────────────────────
Scroll indicators:     ▲ ▼ ● ○
Connection status:     ● Connected  ○ Disconnected  ◌ Connecting
Prompt symbol:         $ or ❯
Menu borders:          ┌─┐ │ │ └─┘
Progress:              ████░░░░ 50%
```

---

## 15. File-by-File Implementation Notes

### `src/main.ts`
- Entry point. Imports all modules.
- Initializes bridge: `const bridge = new EvenAppBridge()` → `bridge.init(onEvenHubEvent)`
- Checks launch source, calls `display.createInitialPage(bridge)`
- Loads config, calls `connection.connect(config)`
- Exports nothing. Side-effect module.

### `src/display.ts`
- Exports: `createInitialPage(bridge)`, `updateTerm(text)`, `updateStatus(text)`, `scheduleRender()`
- Owns the throttle timer and render state
- Calls `buffer.getViewportLines()` inside `renderViewport()`
- Holds container IDs/names as constants

### `src/input.ts`
- Exports: `initInput(sendFn)` where `sendFn` sends to WebSocket
- Sets up the hidden input, auto-focus, and keydown handler
- Manages command history array and index
- Calls `sendFn` for all key actions
- Calls `display.scheduleRender()` after Ctrl+L (clear)
- Calls `buffer.scrollUp()`, `buffer.scrollDown()`, `buffer.snapToLive()` — but these are triggered from main.ts via glasses events, not keyboard

### `src/buffer.ts`
- Exports: `appendOutput(raw)`, `getViewportLines()`, `scrollUp()`, `scrollDown()`, `snapToLive()`, `clear()`
- Owns the `ScrollbackBuffer` state
- Calls `ansi.stripAnsi()` on incoming data
- Handles partial-line accumulation (data may arrive mid-line)
- Word-wraps long lines to EST_CHARS_PER_LINE

### `src/ansi.ts`
- Exports: `stripAnsi(raw: string): string`
- Pure function. No state. Single regex replace.

### `src/connection.ts`
- Exports: `connect(config)`, `disconnect()`, `send(msg)`, `isConnected()`
- Manages WebSocket lifecycle and reconnection
- Calls `buffer.appendOutput()` on incoming `output` messages
- Calls `display.updateStatus()` on status/error/disconnect messages
- Calls `display.scheduleRender()` after appending output

### `src/types.ts`
- Shared interfaces: `ConnectionConfig`, `WSMessage`, `GatewayMessage`, `ScrollbackBuffer`
- All display/buffer constants

---

## 16. Testing Strategy

| Level | Tool | What |
|-------|------|------|
| Unit | Vitest | `ansi.ts` (strip various escape sequences), `buffer.ts` (append, scroll, wrap, viewport extraction) |
| Integration | evenhub-simulator | Full app in simulator — keyboard input, display rendering, mock WebSocket |
| E2E | Real hardware | G2 glasses + iPhone + gateway VM on Tailscale. The only way to validate display behavior, font metrics, update latency, BLE stability. |

---

## 17. Dependencies

### Phone App

```json
{
  "dependencies": {
    "@evenrealities/even_hub_sdk": "^0.0.9"
  },
  "devDependencies": {
    "@evenrealities/evenhub-cli": "latest",
    "typescript": "^5.9",
    "vite": "^7.3",
    "vitest": "latest"
  }
}
```

### Gateway

```json
{
  "dependencies": {
    "ws": "^8.0",
    "ssh2": "^1.15"
  },
  "devDependencies": {
    "@types/ws": "^8.0",
    "@types/ssh2": "^1.15",
    "typescript": "^5.9",
    "tsx": "latest"
  }
}
```

---

## 18. Security Model

| Concern | Mitigation |
|---------|-----------|
| SSH private keys | Stored on gateway VM only. Never on phone. |
| Gateway auth | Phone sends bearer token on `auth` message. Token set at gateway startup. |
| Transport encryption | WSS (TLS) over Tailscale (WireGuard). Double encrypted. |
| Phone-side storage | `bridge.setLocalStorage` for non-sensitive prefs only. No keys, no tokens in persisted storage. |
| Session timeout | Gateway auto-closes SSH sessions after configurable inactivity (default 30 min). |
| Tailscale ACLs | Gateway VM only accepts connections from phone's Tailscale IP. |
