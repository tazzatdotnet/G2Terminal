# G2 SSH Terminal — Architecture & Limitations

## Project overview

An SSH terminal emulator for the Even Realities G2 smart glasses. The user types commands on a Bluetooth keyboard paired to their iPhone; terminal output is displayed on the glasses. The phone screen does not need to be looked at.

## Architecture

```
                    Tailscale VPN (private network)
                              │
G2 Glasses ◄──BT 5.2──► iPhone 15 (iOS 26)  ◄──WebSocket/TLS──► Proxmox VM (SSH Gateway)
                         │                                         │
                    Even Hub WebView                          SSH to any
                    (app logic, input,                        tailnet device
                     display rendering)                       (no agent needed
                                                              on targets)
```

### Component responsibilities

| Component | Role |
|-----------|------|
| G2 glasses | Display terminal output (576×288, 4-bit greyscale). Touchpad input for scrolling/menu. |
| iPhone + BT keyboard | Runs Even Hub WebView app. Captures all keyboard input. Manages WebSocket to gateway. Renders output to glasses via SDK. |
| Proxmox VM (gateway) | Runs a WebSocket-to-SSH proxy. Accepts WebSocket connections from the phone, opens real TCP/SSH sessions to target machines. Holds SSH keys. Reachable via Tailscale IP. |
| Target machines | Standard sshd, no modifications, no agents. The gateway connects to them like any SSH client. |

### Why a gateway is required

Even Hub apps run in a WKWebView. WebViews cannot open raw TCP sockets — they are limited to HTTP, HTTPS, and WebSocket. SSH is a TCP protocol on port 22. Native iOS terminal apps (Termius, Blink Shell) work because they are compiled Swift/ObjC with access to Apple's Network framework for raw TCP. There is no workaround for this in the web platform. A WebSocket-to-SSH proxy is architecturally mandatory.

The Even Hub SDK does not expose any raw socket or native networking bridge.

### Tailscale routing

Tailscale on iOS operates as a system-wide VPN. All traffic from the iPhone — including WKWebView network requests — routes through the Tailscale tunnel when connected. This has been confirmed through direct testing: Safari, WebViews, and local web apps all resolve Tailscale IPs (100.x.x.x) correctly. No special configuration is needed beyond having Tailscale connected.

## Gateway proxy options

Any of these can run on the Proxmox VM:

| Option | Language | Notes |
|--------|----------|-------|
| Custom `ssh2` + `ws` | Node.js | Full control, simple to build, ~100 lines |
| Sshwifty | Go | Single binary, web UI included (ignore UI, use WebSocket API) |
| webssh2 | Node.js | Mature, WebSocket-native |
| ttyd | C | Lightweight, shares a terminal over HTTP/WebSocket |
| Apache Guacamole | Java | Heavy, enterprise-grade, supports RDP/VNC too |

Recommended: start with a custom Node.js proxy using `ssh2` and `ws` for maximum control over the protocol, then evaluate off-the-shelf options if maintenance becomes a burden.

## Phone-side input design

### Core principle

The phone is operated entirely by keyboard. The user never needs to look at the phone screen.

### Input field behavior

- A single `<input>` element is auto-focused on app load
- On `blur`, the input refocuses after a 50ms delay
- `document.addEventListener('click')` and `window.addEventListener('focus')` also trigger refocus
- All interaction is handled via a `keydown` event listener on the document

### Key bindings

| Key | Action |
|-----|--------|
| Enter | Send current input as command |
| Ctrl+C | Send SIGINT to remote process |
| Ctrl+D | Send EOF (close stdin / logout) |
| Tab | Send tab character (trigger remote shell completion) |
| Arrow Up | Navigate command history (previous) |
| Arrow Down | Navigate command history (next) |
| Ctrl+L | Clear glasses display |
| Escape | Send escape character to remote |

### Command history

- Maintained as an array on the phone, most recent first
- Arrow Up/Down cycles through history
- History index resets on Enter
- History is stored in memory; optionally persist via `bridge.setLocalStorage`

### Focus mode (screen-off alternative)

Since iOS suspends WebViews when the screen turns off, true screen-off is not viable. Instead:

- App requests a Wake Lock (`navigator.wakeLock.request('screen')`)
- Page background set to `#000000` (zero power draw on OLED)
- All visible UI hidden except a minimal status indicator
- Phone brightness set to minimum by the user manually
- Phone can be placed face-down or pocketed (with keyboard on desk)

## Display constraints & limitations

### Hardware facts

| Constraint | Value | Impact |
|------------|-------|--------|
| Resolution | 576 × 288 px per eye | ~50 characters per line (varies by character) |
| Color depth | 4-bit greyscale (16 shades of green) | No color-coded terminal output |
| Font | Single LVGL firmware font, **not monospaced** | Column-aligned output (docker ps, ls -la) will not align properly |
| Font control | None (no size, weight, or family selection) | Cannot use a monospace font even if one existed |
| Text alignment | Left-aligned, top-aligned only | Cannot center or right-align text |
| Background | None (no fill, no background color) | Cannot simulate a traditional terminal background |
| Max containers | 4 image + 8 other per page | Limits UI complexity |
| Line breaks | `\n` supported | Only way to control vertical layout within a container |

### Character limits

| SDK method | Max characters | Use case |
|------------|---------------|----------|
| `createStartUpPageContainer` | 1,000 | Initial page setup |
| `textContainerUpgrade` | 2,000 | Live output updates (preferred) |
| `rebuildPageContainer` | 1,000 | Full layout changes |

### What this means in practice

- A full screen of text is approximately 400–500 characters
- `textContainerUpgrade` allows up to 2,000 chars but only ~500 are visible at once
- A single verbose command (large `ls -la`, build log) can easily produce 50,000+ characters
- All output must be buffered on the phone and paginated for display
- The glasses show the current "page"; the phone holds the full scrollback

## Known limitations (hard)

These cannot be worked around within the current platform.

### 1. No monospace font
**Impact**: High for readability.
Terminal output assumes monospace for columnar alignment. Output from `docker ps`, `ps aux`, `ls -la`, `df -h`, `top`, and most CLI tools will have misaligned columns on the G2. The output is still readable as text, but tabular structure is lost.
**Mitigation**: Accept as a cosmetic limitation. Consider post-processing output to replace column alignment with delimiter-separated or reformatted output where possible.

### 2. No full-screen TUI apps
**Impact**: vim, nano, tmux, htop, less, man pages — anything that uses cursor addressing (ANSI escape sequences to position the cursor at arbitrary screen coordinates) will not render correctly.
**Mitigation**: Treat these as unsupported. Use non-interactive alternatives where possible (`cat` instead of `less`, `grep` instead of interactive search). Document this clearly for users.

### 3. No raw TCP from WebView
**Impact**: Architectural. Requires a gateway proxy.
**Mitigation**: Proxmox VM on Tailscale (see Architecture section).

### 4. WebSocket disconnects on app background
**Impact**: When the user switches to another iOS app or iOS decides to reclaim memory, the Even Realities host app's WebView may be suspended. This kills the WebSocket connection.
**Mitigation**: Implement automatic reconnection logic on the phone. The gateway should keep SSH sessions alive for a configurable timeout (e.g., 60 seconds) waiting for WebSocket reconnection. Use SSH keepalives on the gateway side.

### 5. 2,000 character update ceiling
**Impact**: Cannot send more than 2,000 characters in a single `textContainerUpgrade` call.
**Mitigation**: Phone-side scrollback buffer. Only the visible page (~400–500 chars) is sent to the glasses. Swipe up/down on the G2 touchpad to navigate the buffer.

### 6. No persistent secure storage
**Impact**: `bridge.setLocalStorage` is the only persistence API. It is unknown whether this is encrypted at rest on the device.
**Mitigation**: Do not store SSH private keys on the phone. Store keys on the gateway VM. The phone authenticates to the gateway using a session token, password, or a less-sensitive credential. The gateway holds the real SSH keys and manages connections to targets.

## Known limitations (soft)

These may be workable with testing and tuning.

### 7. textContainerUpgrade rate limiting (untested)
**Impact**: Unknown. If SSH output streams rapidly (build logs, `cat` on a large file, `yes` command), the phone may call `textContainerUpgrade` many times per second. The glasses firmware may queue, drop, or mishandle rapid updates.
**Mitigation**: Throttle updates on the phone to a fixed interval (start with 150ms, tune based on testing). Buffer incoming output and render the latest state on each tick. Test on both simulator and hardware.

### 8. ANSI escape code handling
**Impact**: Raw SSH output contains ANSI escape sequences for colors, cursor movement, screen clearing, and window titles. A naive regex strip handles simple cases but mangles interactive output (tab completion rendering, progress bars, `apt` install output).
**Mitigation**: Use a proper terminal state parser. Options:
- **xterm.js** running headless on the phone — maintain a virtual terminal buffer, extract visible text for display on glasses. This is the most robust approach.
- **Lightweight ANSI stripper** — regex-based, handles 90% of cases, breaks on cursor-addressed output.
Recommend starting with the lightweight stripper and upgrading to xterm.js if edge cases become painful.

### 9. Even Hub network permission whitelist format (untested)
**Impact**: The `app.json` whitelist field expects URL strings. Tailscale IPs are in the `100.64.0.0/10` range. It is unknown whether the whitelist supports wildcards, CIDR notation, or only exact URLs.
**Mitigation**: Test with an exact Tailscale IP first (`ws://100.x.x.x:8022`). If wildcards are needed, check Even Hub docs or ask in the Discord. Worst case, use a stable Tailscale hostname via MagicDNS (e.g., `ws://gateway.tailnet-name.ts.net:8022`).

### 10. Display ghosting / update latency on hardware (untested)
**Impact**: The G2 display technology may have visible ghosting or latency when text content changes rapidly. This would affect the feel of interactive terminal usage.
**Mitigation**: Test on hardware. If ghosting occurs, increase the update throttle interval or implement a "settled" delay that waits for output to pause before updating the display.

### 11. iOS Wake Lock support in WKWebView (untested)
**Impact**: The Screen Wake Lock API (`navigator.wakeLock`) may not be available in WKWebView on iOS 26. Safari gained support in iOS 16.4, but WKWebView feature parity is not guaranteed.
**Mitigation**: Test on device. Fallback: use a looping silent audio track or a `setInterval` keep-alive as a secondary anti-suspension mechanism. Neither is guaranteed.

## Glasses-side UI layout

### Container layout

```
┌──────────────────────────────────────────────────────┐
│ status bar (containerID: 1, height: ~24px)            │
│ user@host:~/path              ▲▼ scroll  ● menu      │
├──────────────────────────────────────────────────────┤
│ main output (containerID: 2, height: ~264px)          │
│                                                       │
│ $ ls -la                                              │
│ total 48                                              │
│ drwxr-xr-x  5 user user 4096 Apr 9  .                │
│ drwxr-xr-x  3 user user 4096 Apr 8  ..               │
│ -rw-r--r--  1 user user  220 Apr 7  .bashrc           │
│ drwxr-xr-x  2 user user 4096 Apr 9  src/              │
│ -rw-r--r--  1 user user 1247 Apr 9  package.json      │
│                                                       │
│ $ _                                                   │
│                                                       │
└──────────────────────────────────────────────────────┘
```

### Touchpad controls

| Input | Action |
|-------|--------|
| Swipe up | Scroll up through output buffer |
| Swipe down | Scroll down through output buffer |
| Press | Snap to latest output (live mode) |
| Double press | Open menu (connection settings, disconnect, clear) |

### Update strategy

- Use `createStartUpPageContainer` once at startup to create the status bar and main output containers
- Use `textContainerUpgrade` for all subsequent output updates (flicker-free, 2,000 char limit)
- Use `rebuildPageContainer` only for layout changes (e.g., showing/hiding the status bar)
- Never use `rebuildPageContainer` for content updates — it causes a brief flicker on hardware

## Scrollback buffer design

```
Phone memory:
┌─────────────────────────────┐
│ line 0 (oldest)              │
│ line 1                       │
│ ...                          │
│ line N-K ◄── scroll position │  ← glasses show lines N-K to N-K+page_size
│ ...                          │
│ line N (newest)              │
└─────────────────────────────┘
```

- Buffer is a ring buffer or array of processed (ANSI-stripped) lines
- A "viewport" pointer tracks which slice of the buffer is currently displayed on glasses
- In "live mode" (default), viewport is pinned to the bottom — new output appears immediately
- Swipe up enters "scroll mode" — viewport moves up, new output is buffered but not displayed
- Press to snap back to live mode
- Page size is calibrated empirically (likely 12–15 lines depending on content)
- Maximum buffer size TBD (1,000–10,000 lines; constrained by phone memory, not glasses)

## Security considerations

- SSH private keys stored only on the gateway VM, never on the phone
- Gateway WebSocket should use WSS (TLS) even over Tailscale
- Gateway should authenticate phone connections (token, password, or client certificate)
- Consider a session timeout on the gateway — auto-disconnect after inactivity
- Tailscale already provides encrypted, authenticated network transport
- `bridge.setLocalStorage` used only for non-sensitive preferences (connection hostname, username, UI settings)

## Future expansion (out of scope for v1)

These ideas build on the terminal foundation but are separate projects:

- **AI agent**: Same WebSocket to gateway, additional message type for natural language commands. Gateway runs an LLM (local Gemma4 on 2x 3060, or cloud via AbacusAI/OpenClaw) that translates requests to shell commands and executes them. "JARVIS mode."
- **Multi-session**: Gateway manages multiple SSH sessions, phone UI for switching between them
- **File transfer**: Upload/download files through the gateway
- **Notifications**: Gateway monitors services and pushes alerts to glasses
- **Voice input**: G2 mic array → speech-to-text → command execution

## Open questions (need testing)

1. Does `textContainerUpgrade` have a practical rate limit? What happens at 10+ calls/second?
2. Does the Even Hub network whitelist support Tailscale IPs or MagicDNS hostnames?
3. Is `navigator.wakeLock` available in the Even Hub WKWebView on iOS 26?
4. How does the G2 display handle rapid text changes — any ghosting?
5. Does the Even Realities host app maintain Bluetooth connection when backgrounded on iOS?
6. What is the actual character capacity per line and per screen on hardware (empirical measurement needed)?
7. Would Even Realities consider exposing a raw socket API in the bridge for future SDK versions?
