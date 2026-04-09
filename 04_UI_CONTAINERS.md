# 04 — UI Containers & Display System

> **Source**: `@evenrealities/even_hub_sdk` v0.0.9, `nickustinov/even-g2-notes` (display.md, ui-patterns.md, browser-ui.md), official demos. All facts verified April 2026.
> **Cross-reference**: `03_SDK_API_REFERENCE.md` for method signatures, `05_EVENTS_AND_INPUT.md` for event handling.

---

## The Canvas

The G2 display is a **576 × 288 pixel** green micro-LED panel.

| Property | Value |
|----------|-------|
| Resolution | 576 × 288 px |
| Colour | Green micro-LED — all colours converted to **4-bit greyscale** (16 levels of green) |
| Coordinate origin | Top-left `(0, 0)` |
| X axis | Increases rightward |
| Y axis | Increases downward |
| Colour depth | 4-bit (16 shades) |
| Background | Black (off) — no background colour property |

> ⚠️ There is **no CSS, no flexbox, no DOM** on the glasses display. You define containers with pixel coordinates and the glasses firmware renders them. All layout is absolute pixel positioning.

---

## Container Model

The UI is built from **containers** — rectangular regions positioned absolutely on the canvas. Three container types exist:

| Type | Class | Purpose |
|------|-------|---------|
| Text | `TextContainerProperty` | Renders plain text, supports internal scroll |
| List | `ListContainerProperty` | Native scrollable list widget |
| Image | `ImageContainerProperty` | Displays greyscale images |

### Global Container Rules

| Rule | Detail |
|------|--------|
| Max containers per page | **4** (mixed types allowed) |
| Max text containers per page | **8** |
| `containerTotalNum` | Must exactly equal the total number of containers passed across all types |
| `isEventCapture` | Exactly **one** container per page must be `1` |
| Overlap | Containers can overlap — later containers (higher `containerID`) draw on top |
| Z-index | No explicit z-index — draw order is declaration order |
| `containerID` | Must be unique per page |
| `containerName` | Must be unique per page, max **16 characters** |

---

## Shared Container Properties

All container types share these layout properties:

| Property | Type | Range | Notes |
|----------|------|-------|-------|
| `xPosition` | number | 0–576 | Left edge in pixels |
| `yPosition` | number | 0–288 | Top edge in pixels |
| `width` | number | 0–576 (20–200 for images) | Container width |
| `height` | number | 0–288 (20–100 for images) | Container height |
| `containerID` | number | any | Unique per page, used for updates |
| `containerName` | string | max 16 chars | Unique per page, used for updates |
| `isEventCapture` | number | 0 or 1 | Exactly one container must be `1` |

### Border and Decoration (Text and List only — not Image)

| Property | Type | Range | Notes |
|----------|------|-------|-------|
| `borderWidth` | number | 0–5 | `0` = no border |
| `borderColor` | number | 0–15 (list), 0–16 (text) | Greyscale level. `5` = subtle grey, `13` = bright |
| `borderRadius` | number | 0–10 | Rounded corners |
| `paddingLength` | number | 0–32 | Uniform padding on all sides |

> ⚠️ There is **no background colour property** and **no fill colour**. The only visual decoration is the border. The display background is always black (off).

---

## Text Containers (`TextContainerProperty`)

The workhorse container. Renders plain text, left-aligned, top-aligned.

```typescript
new TextContainerProperty({
  xPosition: 0,
  yPosition: 0,
  width: 576,
  height: 288,
  borderWidth: 0,
  borderColor: 5,
  borderRadius: 0,
  paddingLength: 8,
  containerID: 1,
  containerName: 'main-text',   // max 16 chars
  content: 'Hello from G2',
  isEventCapture: 1,
})
```

### Text Container Properties

| Property | Type | Range | Notes |
|----------|------|-------|-------|
| `xPosition` | number | 0–576 | Left edge |
| `yPosition` | number | 0–288 | Top edge |
| `width` | number | 0–576 | Container width |
| `height` | number | 0–288 | Container height |
| `borderWidth` | number | 0–5 | `0` = no border |
| `borderColor` | number | 0–16 | Greyscale level |
| `borderRadius` | number | 0–10 | Rounded corners |
| `paddingLength` | number | 0–32 | Uniform padding |
| `containerID` | number | any | Unique per page |
| `containerName` | string | max 16 chars | Unique per page |
| `content` | string | max 1000 chars (startup/rebuild), max 2000 (upgrade) | Text content |
| `isEventCapture` | number | 0 or 1 | Exactly one per page must be `1` |

### Content Limits

| Method | Max chars per text container |
|--------|------------------------------|
| `createStartUpPageContainer` | **1000** |
| `rebuildPageContainer` | **1000** |
| `textContainerUpgrade` | **2000** |

### Text Rendering Behaviour

- Text wraps at container width automatically
- If content overflows container height **and** `isEventCapture: 1`, the firmware scrolls internally — user can scroll with swipe/ring gestures
- `SCROLL_TOP_EVENT` and `SCROLL_BOTTOM_EVENT` fire when the user reaches the top or bottom boundary — they are **boundary events**, not raw gesture events
- Containers without `isEventCapture: 1` do **not** receive scroll input — overflow text is clipped
- `\n` works for line breaks
- Unicode characters work (see full glyph table below)
- Approximate capacity: **~400–500 characters** fill a full-screen (576×288) text container, depending on character width
- **No font selection, no font size** — the firmware uses a single fixed-width-ish LVGL font
- **No text alignment** — always left-aligned, top-aligned. To "centre" text, manually pad with spaces
- **No bold, italic, or underline**

### Partial Updates via `textContainerUpgrade`

Updates text content in-place without a full page rebuild. Faster and flicker-free on real hardware.

```typescript
await bridge.textContainerUpgrade(new TextContainerUpgrade({
  containerID: 1,
  containerName: 'main-text',
  contentOffset: 0,       // start position in existing content string
  contentLength: 100,     // length of content to replace
  content: 'Updated text here',
}))
```

> ⚠️ On the simulator, this still causes a visual redraw. On real hardware, it's smooth and flicker-free.

---

## Font and Unicode Support

The glasses firmware uses a **single LVGL font** baked into the firmware. There is no font selection, no font size control. The font is **not monospaced** — different characters have different widths. Characters outside the font are **silently skipped** (no placeholder glyph).

> Tested on Even Hub Simulator v0.0.7 (Feb 2025). Real hardware may differ slightly.

### ✅ Available Glyphs

**ASCII and Latin (U+0020–U+00FF)** — nearly complete (all printable except 5):

All printable ASCII `U+0021–U+007E` and Latin-1 Supplement `U+00A1–U+00FF` (accented letters, symbols, etc.)

**Arrows (12 characters):**

| Char | Code | | Char | Code | | Char | Code |
|------|------|-|------|------|-|------|------|
| ← | U+2190 | | ↑ | U+2191 | | → | U+2192 |
| ↓ | U+2193 | | ↔ | U+2194 | | ↕ | U+2195 |
| ↖ | U+2196 | | ↗ | U+2197 | | ↘ | U+2198 |
| ↙ | U+2199 | | ⇒ | U+21D2 | | ⇔ | U+21D4 |

**Box Drawing (U+2500–U+2573)** — single-line and light/heavy characters:

| Char | Code | | Char | Code | | Char | Code | | Char | Code |
|------|------|-|------|------|-|------|------|-|------|------|
| ─ | U+2500 | | ━ | U+2501 | | │ | U+2502 | | ┃ | U+2503 |
| ┌ | U+250C | | ┍ | U+250D | | ┎ | U+250E | | ┏ | U+250F |
| ┐ | U+2510 | | ┑ | U+2511 | | ┒ | U+2512 | | ┓ | U+2513 |
| └ | U+2514 | | ┕ | U+2515 | | ┖ | U+2516 | | ┗ | U+2517 |
| ┘ | U+2518 | | ┙ | U+2519 | | ┚ | U+251A | | ┛ | U+251B |
| ├ | U+251C | | ┝ | U+251D | | ┞ | U+251E | | ┟ | U+251F |
| ┠ | U+2520 | | ┡ | U+2521 | | ┢ | U+2522 | | ┣ | U+2523 |
| ┤ | U+2524 | | ┥ | U+2525 | | ┦ | U+2526 | | ┧ | U+2527 |
| ┨ | U+2528 | | ┩ | U+2529 | | ┪ | U+252A | | ┫ | U+252B |
| ┬ | U+252C | | ┭ | U+252D | | ┮ | U+252E | | ┯ | U+252F |
| ┰ | U+2530 | | ┱ | U+2531 | | ┲ | U+2532 | | ┳ | U+2533 |
| ┴ | U+2534 | | ┵ | U+2535 | | ┶ | U+2536 | | ┷ | U+2537 |
| ┸ | U+2538 | | ┹ | U+2539 | | ┺ | U+253A | | ┻ | U+253B |
| ┼ | U+253C | | ┽ | U+253D | | ┾ | U+253E | | ┿ | U+253F |
| ╀ | U+2540 | | ╁ | U+2541 | | ╂ | U+2542 | | ╃ | U+2543 |
| ╄ | U+2544 | | ╅ | U+2545 | | ╆ | U+2546 | | ╇ | U+2547 |
| ╈ | U+2548 | | ╉ | U+2549 | | ╊ | U+254A | | ╋ | U+254B |
| ═ | U+2550 | | ╞ | U+255E | | ╡ | U+2561 | | ╪ | U+256A |
| ╭ | U+256D | | ╮ | U+256E | | ╯ | U+256F | | ╰ | U+2570 |
| ╱ | U+2571 | | ╲ | U+2572 | | ╳ | U+2573 | | | |

**Block Elements (U+2580–U+2595):**

| Char | Code | Name |
|------|------|------|
| ▁ | U+2581 | Lower one eighth block |
| ▂ | U+2582 | Lower one quarter block |
| ▃ | U+2583 | Lower three eighths block |
| ▄ | U+2584 | Lower half block |
| ▅ | U+2585 | Lower five eighths block |
| ▆ | U+2586 | Lower three quarters block |
| ▇ | U+2587 | Lower seven eighths block |
| █ | U+2588 | Full block |
| ▉ | U+2589 | Left seven eighths block |
| ▊ | U+258A | Left three quarters block |
| ▋ | U+258B | Left five eighths block |
| ▌ | U+258C | Left half block |
| ▍ | U+258D | Left three eighths block |
| ▎ | U+258E | Left one quarter block |
| ▏ | U+258F | Left one eighth block |
| ▒ | U+2592 | Medium shade |
| ▔ | U+2594 | Upper one eighth block |
| ▕ | U+2595 | Right one eighth block |

**Geometric Shapes (selective):**

| Char | Code | | Char | Code | | Char | Code | | Char | Code |
|------|------|-|------|------|-|------|------|-|------|------|
| ■ | U+25A0 | | □ | U+25A1 | | ▲ | U+25B2 | | △ | U+25B3 |
| ▶ | U+25B6 | | ▷ | U+25B7 | | ▼ | U+25BC | | ▽ | U+25BD |
| ◀ | U+25C0 | | ◁ | U+25C1 | | ◆ | U+25C6 | | ◇ | U+25C7 |
| ◈ | U+25C8 | | ◊ | U+25CA | | ○ | U+25CB | | ● | U+25CF |
| ◐ | U+25D0 | | ◑ | U+25D1 | | ◢ | U+25E2 | | ◣ | U+25E3 |
| ◤ | U+25E4 | | ◥ | U+25E5 | | ◯ | U+25EF | | | |

**Misc Symbols (13 characters):**

| Char | Code | | Char | Code | | Char | Code |
|------|------|-|------|------|-|------|------|
| ★ | U+2605 | | ☆ | U+2606 | | ☉ | U+2609 |
| ☎ | U+260E | | ☏ | U+260F | | ☜ | U+261C |
| ☞ | U+261E | | ♠ | U+2660 | | ♡ | U+2661 |
| ♣ | U+2663 | | ♤ | U+2664 | | ♥ | U+2665 |
| ♧ | U+2667 | | | | | | |

**Typographic symbols:**

| Char | Code | | Char | Code | | Char | Code |
|------|------|-|------|------|-|------|------|
| © | U+00A9 | | ® | U+00AE | | ™ | U+2122 |
| † | U+2020 | | ※ | U+203B | | ° | U+00B0 |
| ∞ | U+221E | | | | | | |

**Superscripts and subscripts:**

| Char | Code | | Char | Code | | Char | Code | | Char | Code |
|------|------|-|------|------|-|------|------|-|------|------|
| ⁰ | U+2070 | | ¹ | U+00B9 | | ² | U+00B2 | | ³ | U+00B3 |
| ⁴ | U+2074 | | ⁵ | U+2075 | | ⁶ | U+2076 | | ⁷ | U+2077 |
| ⁸ | U+2078 | | ⁹ | U+2079 | | | | | | |
| ₀ | U+2080 | | ₁ | U+2081 | | ₂ | U+2082 | | ₃ | U+2083 |
| ₄ | U+2084 | | ₅ | U+2085 | | ₆ | U+2086 | | ₇ | U+2087 |
| ₈ | U+2088 | | ₉ | U+2089 | | | | | | |

**Fractions:** ¼ (U+00BC), ½ (U+00BD), ⅛ (U+215B)

---

### ❌ Missing Glyphs

**ASCII/Latin — 5 missing:**

| Char | Code | Name |
|------|------|------|
| ¨ | U+00A8 | Diaeresis |
| ¯ | U+00AF | Macron |
| ´ | U+00B4 | Acute accent |
| µ | U+00B5 | Micro sign |
| ¸ | U+00B8 | Cedilla |

**Arrows** — everything outside the 12 listed above (no double arrows, dashed arrows, wavy arrows)

**Box Drawing missing subsets:**
- U+2504–U+250B — dashed/dotted lines (┄ ┅ ┆ ┇ ┈ ┉ ┊ ┋)
- U+254C–U+254F — dash variants (╌ ╍ ╎ ╏)
- U+2551–U+255D (except U+2550) — most double-line characters
- U+255F–U+2560, U+2562–U+2569, U+256B–U+256C — double-line intersections
- U+2574–U+257F — half-line segments (╴ ╵ ╶ ╷ ╸ ╹ ╺ ╻ ╼ ╽ ╾ ╿)

**Block Elements missing:**
- ▀ (U+2580) Upper half block
- ▐ (U+2590) Right half block
- ░ (U+2591) Light shade
- ▓ (U+2593) Dark shade
- ▖–▟ (U+2596–U+259F) All quadrant characters

**Geometric Shapes missing:** small squares (U+25AA–U+25AB), rectangles, small triangles, circle segments, everything from U+25F0 onward

**Misc Symbols missing:** ☀ ☁ ☂ ☃ (weather), ☠ ☢ ☣ (hazard), ☮ ☯ (peace/yin-yang), ☺ ☻ (faces), most other symbols in U+2600–U+263F

**Entirely absent ranges:**
- Dingbats (U+2700–U+273F)
- Emoji (U+1F300+)
- Misc weather symbols (U+26C4–U+26C8 — ⛄ ⛅ etc.)
- Snowflake (U+2744 ❄), droplet (U+1F4A7 💧)

---

### Useful Unicode Combinations for Apps

**Progress bars:**
```typescript
// 8-level block bar (full → empty)
const filled = '█'.repeat(n)
const empty  = '░'.repeat(total - n)  // ░ missing — use ─ instead
const bar = filled + empty

// Thick/thin line bar
const filled = '━'.repeat(n)
const empty  = '─'.repeat(total - n)
const bar = `[${filled}${empty}]`

// Left-fractional block bar (7 levels of precision)
// ▉▊▋▌▍▎▏ — use for sub-step precision
```

**Games and diagrams:**
```
●○   — filled/empty circles
■□   — filled/empty squares
▲△▶▷▼▽◀◁  — directional triangles
◢◣◤◥  — corner triangles
─│┌┐└┘├┤┬┴┼  — box drawing
╭╮╯╰  — rounded corners
★☆   — stars
♠♤ ♥♡ ♣♧  — card suits (filled/empty)
```

---

## List Containers (`ListContainerProperty`)

Native scrollable list widget rendered by the glasses firmware. The firmware handles scroll highlighting natively — no app intervention needed for scrolling through items.

```typescript
new ListContainerProperty({
  xPosition: 0,
  yPosition: 0,
  width: 576,
  height: 288,
  borderWidth: 1,
  borderColor: 13,
  borderRadius: 6,
  paddingLength: 5,
  containerID: 1,
  containerName: 'my-list',
  isEventCapture: 1,
  itemContainer: new ListItemContainerProperty({
    itemCount: 5,
    itemWidth: 560,          // containerWidth - 2*padding
    isItemSelectBorderEn: 1, // show selection highlight
    itemName: ['Item 1', 'Item 2', 'Item 3', 'Item 4', 'Item 5'],
  }),
})
```

### `ListItemContainerProperty` Fields

| Property | Type | Range | Notes |
|----------|------|-------|-------|
| `itemCount` | number | 1–20 | Must match `itemName.length` |
| `itemWidth` | number | pixels | `0` = auto-fill (firmware calculates). Other = fixed px. Usually `containerWidth - 2*padding` |
| `isItemSelectBorderEn` | number | 0 or 1 | Show selection highlight border on current item |
| `itemName` | string[] | max 64 chars each, max 20 items | Text label for each item |

### List Behaviour

- Firmware handles scrolling natively — scroll gestures move the selection highlight without any app code
- `SCROLL_TOP_EVENT` and `SCROLL_BOTTOM_EVENT` fire only when the user reaches the **top or bottom boundary** of the list (boundary events, not every scroll gesture)
- Click events report `currentSelectItemIndex` and `currentSelectItemName` via `listEvent`
- No custom styling per item — no per-item colours, icons, or secondary text
- No item height control — firmware calculates item height from `containerHeight / itemCount`
- Items are plain text only, single line per item
- Cannot update list items in-place — must `rebuildPageContainer` to change items
- No separator lines between items (border is around the whole list, not between items)

> ⚠️ **List containers take over scroll handling.** If a list is on screen, scroll events arrive as `listEvent` (not `textEvent`/`sysEvent`), and the device moves the selection highlight automatically. You only need to respond to click/double-click.

> ⚠️ **`currentSelectItemIndex` may be missing for index 0.** The SDK's `fromJson` normalises `0` to `undefined`. Always fall back to tracking selection in app state:
> ```typescript
> const index = event.listEvent.currentSelectItemIndex ?? appState.selectedIndex
> ```

---

## Image Containers (`ImageContainerProperty`)

Displays greyscale images on the glasses. The host app converts all image data to 4-bit greyscale (16 shades of green) before sending to the device.

```typescript
new ImageContainerProperty({
  xPosition: 188,   // (576 - 200) / 2 = centred
  yPosition: 94,    // (288 - 100) / 2 = centred
  width: 100,
  height: 50,
  containerID: 3,
  containerName: 'logo',
  // No isEventCapture — image containers cannot receive events
})
```

### Image Container Constraints

| Constraint | Value |
|-----------|-------|
| Min width | 20 px |
| Max width | **200 px** (hardware enforced) |
| Min height | 20 px |
| Max height | **100 px** (hardware enforced) |
| Colour | Converted to 4-bit greyscale by host |
| Concurrent sends | **Not allowed** — queue sequentially |
| Startup data | Cannot send image data during `createStartUpPageContainer` — create empty placeholder, then call `updateImageRawData` |
| Tiling | If image data is smaller than container dimensions, hardware **tiles (repeats)** it — always match image size to container size |
| `isEventCapture` | **Not available** on image containers |

> ⚠️ Image containers **cannot cover the full 576×288 canvas** due to the 200×100 max. To centre a max-size image: `xPosition = (576−200)/2 = 188`, `yPosition = (288−100)/2 = 94`.

### Sending Image Data (`updateImageRawData`)

Image containers are **empty placeholders** until `updateImageRawData` is called. Always call this immediately after page creation.

```typescript
// Step 1: Create page with empty image placeholder
await bridge.createStartUpPageContainer(new CreateStartUpPageContainer({
  containerTotalNum: 2,
  textObject: [
    new TextContainerProperty({
      containerID: 1, containerName: 'evt',
      content: ' ', xPosition: 0, yPosition: 0,
      width: 576, height: 288, isEventCapture: 1, paddingLength: 0,
    }),
  ],
  imageObject: [
    new ImageContainerProperty({
      containerID: 2, containerName: 'logo',
      xPosition: 188, yPosition: 94, width: 200, height: 100,
    }),
  ],
}))

// Step 2: Send image data (AFTER page creation)
const pngBytes = Array.from(new Uint8Array(await pngBlob.arrayBuffer()))
await bridge.updateImageRawData(new ImageRawDataUpdate({
  containerID: 2,
  containerName: 'logo',
  imageData: pngBytes,
}))
```

### `imageData` Accepted Formats

| Format | Notes |
|--------|-------|
| `number[]` | **Recommended** — PNG file bytes or raw greyscale pixel values |
| `string` | Base64-encoded PNG (strip `data:image/png;base64,` prefix) |
| `Uint8Array` | Auto-converted to `number[]` by SDK |
| `ArrayBuffer` | Auto-converted to `number[]` by SDK |

### Image Processing Best Practices

```typescript
// Recommended pipeline for best quality:
// 1. Resize to fit container (preserve aspect ratio)
// 2. Centre on a black canvas (black = off on micro-LED)
// 3. Convert to greyscale (BT.601 luminance: 0.299R + 0.587G + 0.114B)
// 4. Encode as PNG
// 5. Send as number[] or base64

async function prepareImage(
  imageUrl: string,
  containerWidth: number,
  containerHeight: number
): Promise<number[]> {
  const canvas = document.createElement('canvas')
  canvas.width = containerWidth
  canvas.height = containerHeight
  const ctx = canvas.getContext('2d')!

  // Black background (off on micro-LED)
  ctx.fillStyle = 'black'
  ctx.fillRect(0, 0, containerWidth, containerHeight)

  // Load and draw image centred
  const img = new Image()
  img.src = imageUrl
  await new Promise(resolve => { img.onload = resolve })

  const scale = Math.min(containerWidth / img.width, containerHeight / img.height)
  const w = img.width * scale
  const h = img.height * scale
  const x = (containerWidth - w) / 2
  const y = (containerHeight - h) / 2
  ctx.drawImage(img, x, y, w, h)

  // Convert to greyscale (BT.601)
  const imageData = ctx.getImageData(0, 0, containerWidth, containerHeight)
  for (let i = 0; i < imageData.data.length; i += 4) {
    const grey = Math.round(
      0.299 * imageData.data[i] +
      0.587 * imageData.data[i + 1] +
      0.114 * imageData.data[i + 2]
    )
    imageData.data[i] = imageData.data[i + 1] = imageData.data[i + 2] = grey
  }
  ctx.putImageData(imageData, 0, 0)

  // Encode as PNG bytes
  const blob = await new Promise<Blob>(resolve =>
    canvas.toBlob(b => resolve(b!), 'image/png')
  )
  return Array.from(new Uint8Array(await blob.arrayBuffer()))
}
```

> ⚠️ **Do NOT perform 1-bit dithering** in your app. The host does better 4-bit downsampling. Manual Floyd-Steinberg dithering creates noisy green dots on the display.

> ⚠️ **Never send images concurrently.** Always `await` one `updateImageRawData` call before starting the next. Use a queue:

```typescript
// Sequential image update queue
const imageQueue: Array<() => Promise<void>> = []
let imageQueueRunning = false

async function queueImageUpdate(containerID: number, containerName: string, data: number[]) {
  imageQueue.push(async () => {
    await bridge!.updateImageRawData(new ImageRawDataUpdate({
      containerID,
      containerName,
      imageData: data,
    }))
  })
  if (!imageQueueRunning) {
    imageQueueRunning = true
    while (imageQueue.length > 0) {
      await imageQueue.shift()!()
    }
    imageQueueRunning = false
  }
}
```

---

## Page Lifecycle: Three Methods

| Method | When to use | Full redraw? | Content limit |
|--------|-------------|-------------|---------------|
| `createStartUpPageContainer` | **Once** — initial page | Yes | 1000 chars |
| `rebuildPageContainer` | All subsequent page changes | Yes (brief flicker) | 1000 chars |
| `textContainerUpgrade` | Fast text update in existing container | No (smooth) | 2000 chars |

```typescript
// First call — initialize
await bridge.createStartUpPageContainer(new CreateStartUpPageContainer({ ... }))

// All subsequent page changes
await bridge.rebuildPageContainer(new RebuildPageContainer({ ... }))

// Fast text update (no full rebuild, no flicker)
await bridge.textContainerUpgrade(new TextContainerUpgrade({
  containerID: 1,
  containerName: 'main',
  contentOffset: 0,
  contentLength: 100,
  content: 'Updated text',
}))
```

---

## UI Patterns from Real Apps

### Pattern 1: Fake "Buttons" with Text Cursor

Since there are no button widgets, simulate interactive items by appending action labels to text content and tracking a cursor position with `>` prefix:

```typescript
let cursorPos = 0
const items = ['Return', 'Delete note', 'Settings']

function renderMenu() {
  const content = items.map((item, i) =>
    `${i === cursorPos ? '>' : ' '} ${item}`
  ).join('\n')

  bridge.textContainerUpgrade(new TextContainerUpgrade({
    containerID: 1,
    containerName: 'menu',
    content,
  }))
}

bridge.onEvenHubEvent((event) => {
  const { textEvent } = event
  if (!textEvent) return

  const isClick = textEvent.eventType === OsEventTypeList.CLICK_EVENT || textEvent.eventType === undefined
  if (isClick) {
    handleAction(items[cursorPos])
  } else if (textEvent.eventType === OsEventTypeList.SCROLL_BOTTOM_EVENT) {
    cursorPos = Math.min(cursorPos + 1, items.length - 1)
    renderMenu()
  } else if (textEvent.eventType === OsEventTypeList.SCROLL_TOP_EVENT) {
    cursorPos = Math.max(cursorPos - 1, 0)
    renderMenu()
  }
})
```

### Pattern 2: Selection Highlight with Text Borders

Without lists, simulate selection by toggling `borderWidth` on individual text containers, each representing a "row":

```typescript
// 3 containers at height: 96 each = 288px total
// Selected container gets borderWidth: 2, others get borderWidth: 0
async function renderRows(selectedIndex: number) {
  const rows = ['Option A', 'Option B', 'Option C']
  await bridge.rebuildPageContainer(new RebuildPageContainer({
    containerTotalNum: 3,
    textObject: rows.map((text, i) => new TextContainerProperty({
      containerID: i + 1,
      containerName: `row${i}`,
      xPosition: 0,
      yPosition: i * 96,
      width: 576,
      height: 96,
      borderWidth: i === selectedIndex ? 2 : 0,
      borderColor: 13,
      paddingLength: 8,
      content: text,
      isEventCapture: i === 0 ? 1 : 0,  // only one can be 1
    })),
  }))
}
```

### Pattern 3: Multi-Slot Text Layout

Multiple text containers act as rows (e.g. 3 containers at `height: 96` each = 288px total). Each container shows one "item" with its own border state. This simulates a list without the native list widget's scroll-takeover behaviour.

```typescript
// Useful when you need scroll events on individual rows
// (native list takes over scroll — this pattern keeps scroll in textEvent)
const ROW_HEIGHT = 96
const ROW_COUNT = 3

function buildRows(items: string[], selectedIndex: number) {
  return items.slice(0, ROW_COUNT).map((item, i) =>
    new TextContainerProperty({
      containerID: i + 1,
      containerName: `row${i}`,
      xPosition: 0,
      yPosition: i * ROW_HEIGHT,
      width: 576,
      height: ROW_HEIGHT,
      borderWidth: i === selectedIndex ? 2 : 0,
      borderColor: 13,
      paddingLength: 8,
      content: item,
      isEventCapture: i === 0 ? 1 : 0,
    })
  )
}
```

### Pattern 4: Unicode Progress Bars

```typescript
function progressBar(value: number, max: number, width: number = 20): string {
  const filled = Math.round((value / max) * width)
  const empty = width - filled
  return '━'.repeat(filled) + '─'.repeat(empty)
}

// Usage:
const bar = progressBar(7, 10, 20)  // '━━━━━━━━━━━━━━──────'
const content = `Progress\n[${bar}] ${value}/${max}`

// Block bar (8 levels):
function blockBar(value: number, max: number, width: number = 20): string {
  const filled = Math.round((value / max) * width)
  return '█'.repeat(filled) + '▒'.repeat(width - filled)
  // Note: ▒ (U+2592) is available; ░ (U+2591) is NOT
}
```

### Pattern 5: Event Capture for Image-Based Apps

Image containers have **no `isEventCapture` property**. To receive events when displaying images, add a full-screen text container with `isEventCapture: 1` **behind** the image container.

> ⚠️ **Use a text container, not a list.** A hidden 1×1 list with 1 item does NOT generate scroll events — there's nothing to scroll, so `SCROLL_TOP_EVENT` and `SCROLL_BOTTOM_EVENT` never fire. Only click/double-click events come through.

```typescript
// Full-screen text container (invisible) behind image container
const config = {
  containerTotalNum: 2,
  textObject: [
    new TextContainerProperty({
      containerID: 1,
      containerName: 'evt',
      content: ' ',           // single space — invisible but valid
      xPosition: 0,
      yPosition: 0,
      width: 576,
      height: 288,
      isEventCapture: 1,      // receives all events
      paddingLength: 0,
      borderWidth: 0,
    }),
  ],
  imageObject: [
    new ImageContainerProperty({
      containerID: 2,         // higher ID = drawn on top
      containerName: 'screen',
      xPosition: 188,         // (576 - 200) / 2
      yPosition: 94,          // (288 - 100) / 2
      width: 200,
      height: 100,
    }),
  ],
}
```

Events arrive as `textEvent` (not `listEvent`) — adjust your event handler accordingly. This gives you click, double-click, scroll-top, and scroll-bottom events while the image is visible.

### Pattern 6: Page Flipping for Long Text

Although text containers scroll internally for overflow content, many apps pre-paginate text into ~400–500 char pages at word boundaries for better UX:

```typescript
function paginateText(text: string, charsPerPage: number = 450): string[] {
  const pages: string[] = []
  let remaining = text

  while (remaining.length > 0) {
    if (remaining.length <= charsPerPage) {
      pages.push(remaining)
      break
    }
    // Find last word boundary before limit
    let cutAt = charsPerPage
    while (cutAt > 0 && remaining[cutAt] !== ' ' && remaining[cutAt] !== '\n') {
      cutAt--
    }
    if (cutAt === 0) cutAt = charsPerPage  // no word boundary found, hard cut
    pages.push(remaining.slice(0, cutAt).trimEnd())
    remaining = remaining.slice(cutAt).trimStart()
  }

  return pages
}

// Usage with page indicator
let pageIndex = 0
const pages = paginateText(longText)

async function showPage(index: number) {
  const indicator = `${index + 1}/${pages.length}`
  const content = `${indicator}\n${'─'.repeat(20)}\n${pages[index]}`
  await bridge.rebuildPageContainer(new RebuildPageContainer({
    containerTotalNum: 1,
    textObject: [new TextContainerProperty({
      containerID: 1, containerName: 'main',
      xPosition: 0, yPosition: 0, width: 576, height: 288,
      paddingLength: 8, isEventCapture: 1, content,
    })],
  }))
}

bridge.onEvenHubEvent((event) => {
  const { textEvent } = event
  if (!textEvent) return
  if (textEvent.eventType === OsEventTypeList.SCROLL_BOTTOM_EVENT) {
    if (pageIndex < pages.length - 1) showPage(++pageIndex)
  } else if (textEvent.eventType === OsEventTypeList.SCROLL_TOP_EVENT) {
    if (pageIndex > 0) showPage(--pageIndex)
  }
})
```

### Pattern 7: Header + Content Layout

Two-container layout with a fixed header and scrollable content area:

```typescript
const HEADER_HEIGHT = 40
const CONTENT_HEIGHT = 288 - HEADER_HEIGHT  // 248

await bridge.createStartUpPageContainer(new CreateStartUpPageContainer({
  containerTotalNum: 2,
  textObject: [
    // Header (no event capture, no scroll)
    new TextContainerProperty({
      containerID: 1,
      containerName: 'header',
      xPosition: 0,
      yPosition: 0,
      width: 576,
      height: HEADER_HEIGHT,
      borderWidth: 0,
      paddingLength: 4,
      content: 'My App  ●  12:34',
      isEventCapture: 0,
    }),
    // Content (event capture, scrollable)
    new TextContainerProperty({
      containerID: 2,
      containerName: 'content',
      xPosition: 0,
      yPosition: HEADER_HEIGHT,
      width: 576,
      height: CONTENT_HEIGHT,
      borderWidth: 0,
      paddingLength: 8,
      content: mainContent,
      isEventCapture: 1,
    }),
  ],
}))

// Update header only (fast, no flicker)
async function updateHeader(text: string) {
  await bridge.textContainerUpgrade(new TextContainerUpgrade({
    containerID: 1,
    containerName: 'header',
    content: text,
  }))
}
```

### Pattern 8: Canvas Renderer for Games/Graphics

For games or graphical apps, render to an HTML5 Canvas and send as image data:

```typescript
const canvas = document.createElement('canvas')
canvas.width = 200   // max image container width
canvas.height = 100  // max image container height
const ctx = canvas.getContext('2d')!

async function renderFrame(gameState: GameState) {
  // Clear to black (off on micro-LED)
  ctx.fillStyle = 'black'
  ctx.fillRect(0, 0, 200, 100)

  // Draw game elements in white/grey
  ctx.fillStyle = 'white'
  ctx.fillRect(gameState.playerX, gameState.playerY, 8, 8)

  // Encode and send
  const blob = await new Promise<Blob>(resolve =>
    canvas.toBlob(b => resolve(b!), 'image/png')
  )
  const bytes = Array.from(new Uint8Array(await blob.arrayBuffer()))

  await bridge!.updateImageRawData(new ImageRawDataUpdate({
    containerID: 2,
    containerName: 'screen',
    imageData: bytes,
  }))
}

// Game loop — throttle to ~10fps to avoid BLE congestion
let lastFrame = 0
function gameLoop(timestamp: number) {
  if (timestamp - lastFrame > 100) {  // 100ms = 10fps
    lastFrame = timestamp
    renderFrame(gameState)
  }
  requestAnimationFrame(gameLoop)
}
requestAnimationFrame(gameLoop)
```

---

## Phone-Side UI: even-toolkit

The companion settings/config page that runs in the WebView (not the glasses display) can use **[even-toolkit](https://github.com/fabioglimb/even-toolkit)** — a design system and component library built specifically for Even Realities G2 apps.

```bash
npm install even-toolkit
```

```css
/* Import in your app entry point */
@import "even-toolkit/web/theme-light.css";
@import "even-toolkit/web/typography.css";
@import "even-toolkit/web/utilities.css";
```

### Web Components (55+)

```typescript
import { Button, Card, NavBar, ListItem, Toggle, AppShell } from 'even-toolkit/web'
```

**Primitives:** `Button`, `Card`, `Badge`, `Input`, `Textarea`, `Select`, `Checkbox`, `RadioGroup`, `Slider`, `InputGroup`, `Skeleton`, `Progress`, `StatusDot`, `Pill`, `Toggle`, `SegmentedControl`, `Table`, `Kbd`, `Divider`

**Layout:** `AppShell`, `Page`, `NavBar`, `NavHeader`, `ScreenHeader`, `SectionHeader`, `SettingsGroup`, `CategoryFilter`, `ListItem` (swipe-to-delete), `SearchBar`, `Tag`, `TagCarousel`, `TagCard`, `SliderIndicator`, `PageIndicator`, `StepIndicator`, `Timeline`, `StatGrid`, `StatusProgress`

**Feedback:** `TimerRing`, `Dialog`, `ConfirmDialog`, `Toast`, `EmptyState`, `Loading`, `BottomSheet`, `CTAGroup`, `ScrollPicker`, `DatePicker`, `TimePicker`, `SelectionPicker`

**Charts:** `Sparkline`, `LineChart`, `BarChart`, `PieChart`, `StatCard` (via recharts)

**Media:** `ChatContainer`, `ChatBubble`, `ChatInput`, `ChatThinking`, `ChatCodeBlock`, `ChatDiff`, `ChatToolCall`, `ChatCommand`, `ChatError`, `Calendar`, `FileUpload`, `VoiceInput`, `WaveformVisualizer`, `ImageGrid`, `ImageViewer`, `AudioPlayer`

### Icons (191 pixel-art icons)

```typescript
import { IcChevronBack, IcTrash, IcSettings } from 'even-toolkit/web/icons/svg-icons'
```

| Category | Count |
|----------|-------|
| Edit & Settings | 32 |
| Features | 40 |
| Guide | 20 |
| Health | 12 |
| Menu | 8 |
| Navigation | 23 |
| Status | 56 |

Convenience aliases: `IcTrash`, `IcChevronBack`, `IcSettings`, `IcSearch`, `IcPlus`, `IcCross`, `IcEdit`, `IcShare`, `IcCopy`, `IcCheck`, `IcMore`

### Glasses SDK Bridge Helpers

```typescript
import { useGlasses } from 'even-toolkit/useGlasses'
import { line, separator } from 'even-toolkit/types'
import { buildActionBar } from 'even-toolkit/action-bar'
import { wordWrap, paginateText, pageIndicator } from 'even-toolkit/paginate-text'
import { cleanForG2, normalizeWhitespace } from 'even-toolkit/text-clean'
import { DISPLAY_W, DISPLAY_H } from 'even-toolkit/layout'  // 576, 288
import { getCanvas, drawToCanvas, renderToImage } from 'even-toolkit/canvas-renderer'
import { composeStartupPage, composeRebuildPage } from 'even-toolkit/composer'
import { encodeTilesBatch, canvasToPngBytes } from 'even-toolkit/png-utils'
```

### Design Tokens (CSS Custom Properties)

| Token | Light value | Usage |
|-------|-------------|-------|
| `--color-text` | #232323 | Primary text |
| `--color-text-dim` | #7B7B7B | Secondary text |
| `--color-bg` | #EEEEEE | Page background |
| `--color-surface` | #FFFFFF | Card/container surface |
| `--color-border` | #E4E4E4 | Primary border |
| `--color-accent` | #232323 | Primary accent |
| `--color-positive` | #4BB956 | Success/connected |
| `--color-negative` | #FF453A | Error/warning |
| `--radius-default` | 6px | Border radius |
| `--spacing-margin` | 12px | Outer margin |
| `--font-display` | FK Grotesk Neue, -apple-system | Display/body font |
| `--font-mono` | SF Mono, Cascadia Code | Monospace font |

### Typography Classes

| Class | Size | Weight |
|-------|------|--------|
| `.text-vlarge-title` | 24px | 400 |
| `.text-large-title` | 20px | 400 |
| `.text-medium-title` | 17px | 400 |
| `.text-medium-body` | 17px | 300 |
| `.text-normal-title` | 15px | 400 |
| `.text-normal-body` | 15px | 300 |
| `.text-subtitle` | 13px | 400 |
| `.text-detail` | 11px | 400 |

---

## Container Layout Quick Reference

### Full-Screen Single Container (most common)

```typescript
new TextContainerProperty({
  containerID: 1, containerName: 'main',
  xPosition: 0, yPosition: 0, width: 576, height: 288,
  borderWidth: 0, paddingLength: 8, isEventCapture: 1,
  content: 'Your content here',
})
```

### Header + Content (two containers)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Header (576×40, y=0, isEventCapture: 0)                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│ Content (576×248, y=40, isEventCapture: 1)                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Three-Row Layout (three containers)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Row 0 (576×96, y=0, isEventCapture: 1)                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│ Row 1 (576×96, y=96, isEventCapture: 0)                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│ Row 2 (576×96, y=192, isEventCapture: 0)                                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Image + Event Capture (two containers)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Text container (576×288, isEventCapture: 1, content: ' ') — invisible       │
│                    ┌──────────────────────────────┐                         │
│                    │ Image (200×100, centred)      │                         │
│                    │ xPos=188, yPos=94             │                         │
│                    └──────────────────────────────┘                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Critical Gotchas Summary

| Gotcha | Detail |
|--------|--------|
| Max 4 containers per page | Mixed types allowed |
| Max 8 text containers | Hard limit |
| `isEventCapture` exactly one | Exactly one container per page must be `1` |
| Image containers: no `isEventCapture` | Use a text container behind the image |
| Image max size: 200×100 px | Cannot cover full canvas |
| Image containers are empty placeholders | Must call `updateImageRawData` after page creation |
| Never send images concurrently | Always `await` before next send |
| No 1-bit dithering | Host does better 4-bit downsampling |
| Image tiling | If image smaller than container, hardware tiles it |
| `containerName` max 16 chars | Silently truncated or causes errors |
| `containerTotalNum` must match | Must equal actual total containers passed |
| No background colour | Display background is always black |
| No font control | Single LVGL font, not monospaced |
| No text alignment | Always left-aligned, top-aligned |
| Missing glyphs silently skipped | No placeholder glyph shown |
| List takes over scroll | Scroll events arrive as `listEvent`, not `textEvent` |
| `currentSelectItemIndex` missing for 0 | Fall back to app state tracking |
| `textContainerUpgrade` needs matching IDs | `containerID` and `containerName` must match existing container |
| `rebuildPageContainer` loses scroll state | All internal scroll positions reset on rebuild |

---