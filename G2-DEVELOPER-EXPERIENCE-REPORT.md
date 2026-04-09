# Even Realities G2 — Developer Experience & Community Report

*Compiled April 2026*

---

## 1. Platform Overview & Timeline

Even Hub launched publicly on **April 3, 2026** as the official developer platform and app store for the Even Realities G2 smart glasses. Prior to this date, the G2 operated as a closed ecosystem — users were restricted to Even Realities' built-in features (Conversate, Translate, Teleprompt, notifications, navigation). The SDK was available in a developer pilot program before public launch.

At launch, Even Hub debuted with approximately **50 apps and plug-ins** from a developer community Even Realities claims numbers over **2,000 developers**. The SDK, simulator, and CLI are all distributed via npm.

Even Realities has published **no install-base numbers** for the G2, which is a significant concern among developers evaluating whether to invest time in the platform. Glass Almanac specifically called this out: the company shipped features before publishing SDK documentation or install-base data, which slows developer trust.


## 2. What Even Hub Apps Actually Are

An Even Hub app is a **standard web app** (HTML, CSS, TypeScript/JavaScript) that runs inside a WKWebView on the user's iPhone. The phone communicates with the glasses over Bluetooth 5.2. There is no native code compilation, no Swift/Kotlin — everything is web tech.

The SDK (`@evenrealities/even_hub_sdk`) provides a JavaScript bridge that lets the web app control the glasses display and receive input events. The development loop is:

1. Write a standard Vite + TypeScript web app
2. Preview in the simulator (`evenhub-simulator`)
3. Sideload to device via QR code or private build upload
4. Package into `.ehpk` bundle via CLI (`evenhub pack`)
5. Submit to Even Hub for distribution

This is genuinely one of the lowest-friction development experiences in wearable tech. No Xcode, no Android Studio, no special hardware toolchain.


## 3. Community Projects & Examples

The community has been active. The `even-dev` multi-app development environment (by BxNxM) aggregates community apps and provides a unified launcher. As of April 2026, known community projects include:

| App | Author | Description |
|-----|--------|-------------|
| Chess | dmyster145 | Full chess game, R1 ring controls |
| EPUB Reader | chortya | Book reader optimized for HUD |
| Reddit Client | fuutott | Reddit browsing on glasses |
| Smart Cart | bryan-datastorm | Shopping list |
| Stars | thibautrey | Star map |
| Transit | langerhans | Public transport board |
| Weather | nickustinov | Weather display |
| Snake | nickustinov | Classic snake game |
| Pong | nickustinov | Pong game |
| Tetris | nickustinov | Tetris game |
| STT (Speech-to-Text) | nickustinov | Mic-based transcription |
| SubwayLens | (unknown) | NYC transit data |
| Display Plus | (unknown) | Spotify control |
| ER Market | (unknown) | Live stock tickers |
| Stillness | (unknown) | Guided breathing exercises |
| Tamagotchi | (unknown) | Virtual pet |
| Tesla Integration | (unknown) | Vehicle diagnostics |
| Workout guides | (various) | Fitness apps |

Additional community resources:

- **even-better-sdk** by @JappyJan — community wrapper around the official SDK with quality-of-life improvements
- **even-realities-ui** by @JappyJan — reusable UI components for settings pages and consistent styling
- **even-g2-notes** by nickustinov — community documentation including glyph tables and display behavior
- **even-toolkit** by fabioglimb — utility library
- **even-g2-protocol** by i-soxi — BLE protocol reverse engineering project for direct Bluetooth communication, bypassing the official SDK entirely


## 4. Common Developer Issues & Pain Points

### 4.1 No Monospace Font

**Severity: High. Universally cited as the biggest display limitation.**

The G2 uses a single LVGL firmware font that is **not monospaced**. There is no font selection, no size control, no bold/italic. This means any app that relies on column-aligned output (tables, code, terminal output, data grids) will have misaligned columns. Characters like `i`, `l`, `1` are narrower than `m`, `w`, `W`.

There is no workaround. Developers cannot load custom fonts. The firmware font is baked in.

**Impact on apps**: Chess boards need creative unicode solutions. Terminal emulators cannot render aligned columns. Any tabular data display requires either accepting misalignment or reformatting output into non-columnar layouts.

### 4.2 Extremely Limited Display Real Estate

The 576×288 pixel canvas with the firmware font yields approximately **400–500 characters** of visible text at once — roughly 12–15 lines of ~40–50 characters each. The exact count varies because the font is not monospaced.

Combined with character limits on SDK methods:

| Method | Max Characters |
|--------|---------------|
| `createStartUpPageContainer` | 1,000 |
| `textContainerUpgrade` | 2,000 |
| `rebuildPageContainer` | 1,000 |

Any app displaying substantial content must implement its own **scrollback buffer and pagination** on the phone side, sending only the visible "page" to the glasses via `textContainerUpgrade`.

### 4.3 No CSS/DOM on the Glasses

The glasses display is not a browser rendering surface. There is no CSS, no flexbox, no DOM. The UI is built entirely from SDK container objects (TextContainerProperty, ListContainerProperty, image containers) positioned with absolute pixel coordinates.

Developers coming from web backgrounds consistently find this jarring. You cannot use React components, HTML elements, or any web UI framework to render on the glasses. The phone-side WebView has a normal DOM (for app logic, settings UI, etc.), but everything displayed on the glasses goes through the SDK bridge as container objects.

### 4.4 Container Limits

A page can have at most **4 image containers** and **8 other containers**. Exactly one container must have `isEventCapture: 1`. There is no z-index control beyond declaration order.

This constrains UI complexity. You cannot build a complex multi-panel dashboard with many independent regions. Creative developers work around this by packing information into fewer text containers using Unicode box-drawing characters and line breaks.

### 4.5 Simulator vs. Hardware Divergence

The simulator (`@evenrealities/evenhub-simulator`) renders the glasses display on your computer screen. However, developers report that the simulator does not perfectly match hardware behavior:

- The simulator uses a different font rendering than the actual LVGL firmware font on the glasses
- Update rate limits and throttling behave differently
- Display ghosting/latency visible on hardware is not reproduced in the simulator
- The simulator doesn't replicate BLE connection drops or reconnection behavior
- A Japanese developer noted that `npm install` alone wasn't sufficient — additional `npm install @evenrealities/evenhub-simulator` and `npm rebuild` commands were needed beyond what the README documented

### 4.6 `rebuildPageContainer` Flicker

Using `rebuildPageContainer` causes a brief but visible **flicker on hardware** as the entire page is torn down and rebuilt. The recommended approach is to use `textContainerUpgrade` for content changes (which is flicker-free) and reserve `rebuildPageContainer` only for layout structure changes. This is documented but catches many new developers off guard.

### 4.7 Image Container Quirks

- Images **cannot be sent during `createStartUpPageContainer`** — you must create a placeholder container, then update via `updateImageRawData` after the page is created
- No concurrent image sends are allowed
- Image containers are limited to 20–200px wide and 20–100px tall
- The recommended pattern is to use a full-screen text container (with a space character as content) behind the image container for event capture

### 4.8 No Background Color or Fill

There is no background color or fill property on containers. The only visual decoration available is the border (`borderWidth`, `borderColor`, `borderRadius`). The display background is always transparent/off (black). This limits visual differentiation between UI regions.

### 4.9 List Container Inflexibility

`ListContainerProperty` provides native scrollable lists, but:

- Maximum 20 items
- Maximum 64 characters per item
- No custom styling per item
- No item height control
- No separator lines
- **Cannot be updated in-place** — must rebuild the entire page

Many developers avoid list containers entirely and implement their own list-like UIs using text containers with Unicode cursor indicators.

### 4.10 WebView Suspension on iOS

When the user switches to another iOS app or iOS reclaims memory, the Even Realities host app's WebView may be suspended. This kills any WebSocket connections, stops timers, and halts all JavaScript execution. The Screen Wake Lock API (`navigator.wakeLock`) may or may not be available in the WKWebView — this remains untested/unconfirmed.

Apps that maintain persistent connections (WebSocket, polling) must implement reconnection logic.

### 4.11 No Raw TCP Sockets

The WebView sandbox means apps are limited to HTTP, HTTPS, and WebSocket protocols. Raw TCP connections (SSH, MQTT over TCP, custom binary protocols) are impossible. Any app needing raw TCP must use an external proxy/gateway server.

### 4.12 Network Permission Whitelist Ambiguity

The `app.json` manifest requires a `whitelist` array for network permissions. It is unclear whether this supports:

- Wildcards
- CIDR notation
- IP addresses (particularly Tailscale 100.x.x.x range)
- Only exact URLs

Developers working with dynamic IPs or private networks (Tailscale, WireGuard) have to experiment to find what works.

### 4.13 `textContainerUpgrade` Rate Limiting (Unknown)

There is no documented rate limit for `textContainerUpgrade`. If an app streams rapidly changing content (live data, terminal output, game frames), calling this method many times per second may cause the firmware to queue, drop, or mishandle updates. Community consensus is to throttle to ~100–200ms intervals, but the actual firmware behavior is undocumented.

### 4.14 ANSI Escape Code Handling

Apps that display terminal or command-line output must strip ANSI escape sequences. Raw SSH output contains color codes, cursor movement sequences, screen-clearing commands, and window title changes. A naive regex strip handles simple cases but breaks on interactive output (tab completion rendering, progress bars, `apt` install output).

The community has not settled on a standard approach. Options range from lightweight regex stripping to running xterm.js headlessly on the phone.


## 5. Developer Experience: What Works Well

### 5.1 Low Barrier to Entry

The web-app model means any JavaScript/TypeScript developer can start building immediately. No native toolchain, no special IDE, no hardware SDK compilation. `npm install`, write code, `evenhub-simulator`, done.

### 5.2 The `even-dev` Environment

BxNxM's `even-dev` multi-app environment is genuinely excellent community infrastructure. It provides:

- A unified launcher for all community apps
- Auto-discovery and caching of git-based apps
- Vite plugin system for shared functionality
- Visual UI editor (misc/editor) for rapid prototyping
- One-command setup: `./start-even.sh`

### 5.3 QR Sideloading

The ability to generate a QR code (`evenhub qr --url`) and sideload apps directly to the glasses with hot reload is a fast iteration cycle that most wearable platforms lack.

### 5.4 `textContainerUpgrade` for Live Content

The flicker-free in-place text update (up to 2,000 chars) is the workhorse API that makes live-data apps viable. Most successful apps use `createStartUpPageContainer` once at startup and then exclusively use `textContainerUpgrade` for all subsequent changes.

### 5.5 Privacy-First Design

The deliberate absence of a camera means developers don't need to worry about privacy permissions, user consent for recording, or the social stigma that plagued Google Glass. This is a feature, not a limitation — it makes the glasses socially acceptable for all-day wear.

### 5.6 Community & Discord

The Discord community is active and responsive. Even Realities staff participate. Developers share real-time feedback, demos, and workarounds. The community has produced several third-party libraries (even-better-sdk, even-realities-ui) that fill gaps in the official SDK.


## 6. Developer Sentiment (Aggregated)

### Positive Themes

- "Creative constraint" — many developers enjoy the challenge of building useful apps within tight display limits
- Fast iteration cycle from code to glasses
- Genuine excitement about building for a novel form factor
- Privacy-first approach is appreciated
- The Vite + TypeScript stack feels modern and familiar

### Negative Themes

- **"Walled garden" complaints** were loud before Even Hub launched (November 2025 Hacker News: "complete walled garden with very sparse functionality," "no open App Store is a non starter," "Why do I want this if I can't run code on it?"). Even Hub has addressed this, but the perception lingered for months.
- **Install base opacity** — developers want to know how many G2 units are in the wild before committing serious effort. Even Realities has not disclosed this.
- **SDK documentation gaps** — Glass Almanac noted "SDK documentation and install-base figures remain undisclosed, slowing developer trust." The official docs are improving but were initially sparse.
- **Fragmented community** — before Even Hub, development efforts were scattered across Discord, GitHub, and individual repos. An early GitHub issue (#11 on EvenDemoApp) explicitly asked for open-sourcing the base application to centralize community effort, comparing to Flipper Zero's community app store model.
- **Hard-coded API key in demo app** — a Hacker News commenter discovered Even Realities had shipped a hard-coded DeepSeek API key in the EvenDemoApp source code, which "doesn't exactly leave great confidence in their engineering."
- **Software bugginess** — Engadget's review noted "touch controls feel imprecise and occasionally erratic," "many fitness metrics aren't being properly recorded," and "both devices have had a difficult time staying paired to the app." Auto brightness didn't work. These are consumer-facing issues, but they reflect on the platform's maturity.
- **BLE connection reliability** — multiple users and developers report frequent Bluetooth disconnections. Even Realities' own support page has extensive troubleshooting for connection failures, including factory reset procedures.


## 7. BLE Protocol Reverse Engineering

A parallel community effort (even-g2-protocol by i-soxi) is reverse-engineering the G2's BLE protocol independently of the official SDK. This project maps BLE services, UUIDs, and the custom packet protocol, enabling direct Bluetooth communication from Python or any BLE-capable platform.

This effort exists because some developers want to:

- Build native iOS/Android apps (not WebView-constrained)
- Control the glasses from desktop computers, Raspberry Pi, etc.
- Bypass Even Hub entirely for custom deployments
- Access lower-level features not exposed by the SDK

The project includes a working Python teleprompter example that sends text directly to the glasses over BLE.


## 8. Comparison to Other Smart Glasses Developer Platforms

| Aspect | Even G2 / Even Hub | Meta Ray-Ban | Google Glass (historical) |
|--------|-------------------|--------------|---------------------------|
| Developer access | Open SDK + app store (April 2026) | No third-party app SDK | GDK available at launch |
| App model | Web apps in WebView | None (Meta AI only) | Native Android APKs |
| Display | 576×288, 4-bit greyscale | No display | 640×360, color |
| Camera | None | Yes (12MP) | Yes (5MP) |
| Programming language | TypeScript/JavaScript | N/A | Java/Kotlin |
| Distribution | Even Hub + sideloading | N/A | MyGlass store |
| Iteration speed | Fast (hot reload via QR) | N/A | Moderate (ADB sideload) |
| Install base transparency | Unknown | ~10-20M estimated | Unknown at launch |


## 9. Recommendations for New Developers

1. **Start with `even-dev`** — don't set up from scratch. Clone BxNxM's environment and study the chess and reddit apps as reference implementations.

2. **Use `textContainerUpgrade` for everything** — avoid `rebuildPageContainer` unless you're changing the fundamental layout structure.

3. **Throttle display updates** — buffer output on the phone side and render at 100–200ms intervals. Don't call SDK methods on every data event.

4. **Test on hardware early** — the simulator diverges from hardware behavior in font rendering, update latency, and connection behavior. Don't build for months in the simulator only.

5. **Design for ~400 characters visible** — treat it like a 50-char × 12-line terminal. Paginate everything.

6. **Use Unicode creatively** — box-drawing characters, block elements, and arrows are your only visual tools beyond text.

7. **Implement reconnection logic** — WebSocket and BLE connections will drop. Plan for it from day one.

8. **Don't store secrets on the phone** — `bridge.setLocalStorage` is the only persistence API, and its security properties are unknown. Keep sensitive data server-side.

9. **Join the Discord** — it's the fastest path to answers. The community is small and responsive.

10. **Accept the font** — you cannot change it. Design your UI around proportional text, not grid alignment.


## 10. Open Questions for the Platform

1. Will Even Realities publish install-base numbers?
2. Will dashboard widgets and AI skills/integrations launch as documented in the roadmap?
3. Will the SDK expose additional hardware access (raw BLE, programmatic brightness, etc.)?
4. What is the actual firmware rate limit for `textContainerUpgrade`?
5. Will monospace font support ever be added to the firmware?
6. Will the `app.json` network whitelist support wildcards or CIDR notation?
7. Is `navigator.wakeLock` functional in the Even Hub WKWebView on iOS?
8. Will Even Realities open-source more of the platform, or will the community BLE reverse engineering effort remain the only path to low-level access?
9. What is the app review/approval process and timeline for Even Hub submissions?
10. Will G1 glasses receive Even Hub support, or is it G2 only?

---

*Sources: Even Hub official documentation, GitHub repositories (even-dev, EvenDemoApp, even-g2-protocol, even-g2-notes), Hacker News discussions, Engadget review, Glass Almanac analysis, Tom's Guide, Android Authority, Digital Trends, 9to5Google, community Discord feedback (aggregated), and direct SDK/simulator testing.*
