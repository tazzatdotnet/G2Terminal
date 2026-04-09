# 06 — BLE Protocol (Low-Level / Native Path)

> **Source**: `even-realities/EvenDemoApp` (official), `i-soxi/even-g2-protocol` (community reverse engineering), `nickustinov/even-g2-notes` (architecture.md). All facts verified April 2026.
> **Cross-reference**: `01_PLATFORM_OVERVIEW.md` for Path A vs Path B distinction, `05_EVENTS_AND_INPUT.md` for SDK-level event handling.

---

## ⚠️ Critical Distinction: Two Development Paths

| | **Path A — Even Hub SDK** | **Path B — Native BLE** |
|--|--------------------------|------------------------|
| **What it is** | WebView app using `@evenrealities/even_hub_sdk` | Native iOS/Android app using raw BLE |
| **Language** | TypeScript / JavaScript | Swift, Kotlin, Python, etc. |
| **Distribution** | `.ehpk` package via Even Hub | Standalone app, sideloaded, or custom |
| **BLE access** | Abstracted — SDK handles all BLE | Direct BLE characteristic writes |
| **Complexity** | Low — SDK handles protocol | High — must implement full protocol |
| **Use case** | 99% of Even Hub apps | Custom tools, research, unrestricted access |
| **This file covers** | Background context only | Full protocol reference |

> ⚠️ **If you are building an Even Hub app using the SDK, you do NOT need this file.** The SDK abstracts all BLE communication. This file is for developers building native companion apps, research tools, or custom integrations that bypass the Even Hub entirely.

---

## Hardware Architecture

### Dual BLE Design

The G2 glasses have **two independent BLE connections** — one per arm (left and right). Each arm is a separate BLE peripheral with its own connection, characteristics, and sequence counters.

```
iPhone
├── BLE Connection 1 ──► Left Arm  (Even G2_XX_L_YYYYYY)
└── BLE Connection 2 ──► Right Arm (Even G2_XX_R_YYYYYY)
```

**Device naming convention:**

```
Even G2_XX_L_YYYYYY   ← Left arm
Even G2_XX_R_YYYYYY   ← Right arm

Where:
  XX     = Model variant (e.g. "A1", "B2")
  L/R    = Left or Right
  YYYYYY = Serial suffix (6 hex chars)
```

### Dual BLE Communication Rules

| Rule | Detail |
|------|--------|
| Default send order | **Left first**, then right after left ACK |
| Simultaneous send | Allowed for image data only (left and right can receive image packets independently at the same time) |
| Microphone | Right arm only — `0x0E` command sent to right BLE only |
| Display sync | Both arms share the same display content — synced via physical FPC cable (not wireless) |
| Sequence counters | Each arm maintains its own independent sequence counter |

> ⚠️ **Always send to left first, wait for ACK, then send to right** — unless the protocol explicitly specifies one side only (e.g. microphone = right only) or simultaneous send is allowed (e.g. image packets).

---

## BLE Services and UUIDs

### Base UUID Pattern

```
00002760-08c2-11e1-9073-0e8ac72e{XXXX}
```

### Service UUIDs

| UUID Suffix | Full UUID | Purpose |
|-------------|-----------|---------|
| `0000` | `00002760-08c2-11e1-9073-0e8ac72e0000` | Main Service |
| `5401` | `00002760-08c2-11e1-9073-0e8ac72e5401` | Write — Commands (Phone → Glasses) |
| `5402` | `00002760-08c2-11e1-9073-0e8ac72e5402` | Notify — Responses (Glasses → Phone) |
| `5450` | `00002760-08c2-11e1-9073-0e8ac72e5450` | Service Declaration |
| `6402` | `00002760-08c2-11e1-9073-0e8ac72e6402` | Display Rendering |

### ATT Handles

| Handle | UUID Suffix | Direction | Purpose |
|--------|-------------|-----------|---------|
| `0x0842` | 5401 | Write | Commands (Phone → Glasses) |
| `0x0844` | 5402 | Notify | Responses (Glasses → Phone) |
| `0x0864` | 6402 | Write | Display / Rendering commands |
| `0x0884` | ? | Notify | Secondary control (under research) |

### Characteristic Properties

| Characteristic | Properties | MTU | Notes |
|---------------|-----------|-----|-------|
| Write (0x5401) | Write Without Response | 512 bytes | All commands to glasses |
| Notify (0x5402) | Notify | — | All responses from glasses. Must enable CCCD (write `0x0100`) |
| Display (0x6402) | Write Without Response | 204 bytes | Binary display/rendering commands |

### Connection Parameters

```
Connection Interval: 7.5ms – 30ms (typical)
Slave Latency:       0
Supervision Timeout: 2000ms
MTU:                 512 bytes
```

### Pairing

The G2 uses **custom application-level authentication** — not standard BLE pairing/bonding:

- No PIN required
- No secure pairing handshake
- Session established via **7-packet handshake** (see Authentication section below)
- Timestamp + transaction ID exchange

---

## Packet Structure

### Transport Layer Format

Every packet follows this exact structure:

```
┌────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬─────────────┬────────┬────────┐
│ Magic  │ Type   │ Seq    │ Len    │ Pkt    │ Pkt    │ Svc    │ Svc    │ Payload     │ CRC    │ CRC    │
│ 0xAA   │        │ ID     │        │ Tot    │ Ser    │ Hi     │ Lo     │ ...         │ Lo     │ Hi     │
└────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴─────────────┴────────┴────────┘
  [0]      [1]      [2]      [3]      [4]      [5]      [6]      [7]      [8:N-2]       [N-1]    [N]
```

### Header Fields (8 bytes)

| Offset | Field | Description |
|--------|-------|-------------|
| 0 | Magic | Always `0xAA` |
| 1 | Type | `0x21` = Command (Phone→Glasses), `0x12` = Response (Glasses→Phone) |
| 2 | Sequence | Incrementing counter 0–255, wraps around |
| 3 | Length | Payload length + 2 (includes CRC bytes) |
| 4 | Packet Total | Total packets in this message (usually `0x01`) |
| 5 | Packet Serial | Current packet number (usually `0x01`, 1-indexed) |
| 6 | Service Hi | Service ID high byte |
| 7 | Service Lo | Service ID low byte |

### Payload

Variable length, **protobuf-encoded**. Structure depends on service ID.

### CRC (2 bytes, little-endian)

- **Algorithm**: CRC-16/CCITT
- **Init value**: `0xFFFF`
- **Polynomial**: `0x1021`
- **Input**: Payload bytes only — **skip the first 8 header bytes**
- **Output**: Little-endian (low byte first, high byte second)

### CRC Implementation

```python
def crc16_ccitt(data: bytes, init: int = 0xFFFF) -> int:
    crc = init
    for byte in data:
        crc ^= byte << 8
        for _ in range(8):
            if crc & 0x8000:
                crc = (crc << 1) ^ 0x1021
            else:
                crc <<= 1
        crc &= 0xFFFF
    return crc

def build_packet(
    seq: int,
    service_hi: int,
    service_lo: int,
    payload: bytes
) -> bytes:
    header = bytes([
        0xAA,           # Magic
        0x21,           # Type: Command
        seq & 0xFF,     # Sequence
        len(payload) + 2,  # Length (payload + CRC)
        0x01,           # Packet Total
        0x01,           # Packet Serial
        service_hi,     # Service Hi
        service_lo,     # Service Lo
    ])
    crc = crc16_ccitt(payload)
    return header + payload + bytes([crc & 0xFF, (crc >> 8) & 0xFF])
```

### Example Packets

**Command (Phone → Glasses):**
```
AA 21 01 0C 01 01 80 00 [payload] [crc_lo] [crc_hi]
↑  ↑  ↑  ↑  ↑  ↑  ↑  ↑
│  │  │  │  │  │  └──┴── Service ID = 0x8000
│  │  │  │  └──┴──────── Pkt 1 of 1
│  │  │  └────────────── Length = 12 (payload=10 + CRC=2)
│  │  └───────────────── Sequence = 1
│  └──────────────────── Type = Command
└─────────────────────── Magic
```

**Response (Glasses → Phone):**
```
AA 12 FD 08 01 01 80 00 [payload] [crc_lo] [crc_hi]
   ↑  ↑
   │  └── Glasses uses its own sequence counter
   └───── Type = Response
```

---

## Multi-Packet Messages

For payloads exceeding MTU (512 bytes), split into multiple packets:

```
Packet 1: AA 21 05 EC 05 01 [svc_hi] [svc_lo] [chunk1] [crc]
Packet 2: AA 21 05 EC 05 02 [svc_hi] [svc_lo] [chunk2] [crc]
Packet 3: AA 21 05 EC 05 03 [svc_hi] [svc_lo] [chunk3] [crc]
...
Packet 5: AA 21 05 EC 05 05 [svc_hi] [svc_lo] [chunk5] [crc]
```

- **Byte 4** (Packet Total): Total number of packets in this message
- **Byte 5** (Packet Serial): Current packet number (1-indexed, 1 to N)
- **Byte 2** (Sequence): Remains **constant** across all packets of the same message

---

## Varint Encoding

Payload fields use **protobuf-style varint encoding**:

| Value | Encoding |
|-------|----------|
| 0–127 | Single byte |
| 128–16383 | Two bytes (MSB has bit 7 set) |
| 16384+ | Three+ bytes |

```python
def encode_varint(value: int) -> bytes:
    result = []
    while value > 0x7F:
        result.append((value & 0x7F) | 0x80)
        value >>= 7
    result.append(value & 0x7F)
    return bytes(result)

# Examples:
# 10  → 0x0A
# 127 → 0x7F
# 128 → 0x80 0x01
# 255 → 0xFF 0x01
# 300 → 0xAC 0x02
```

---

## Service ID Reference

Service IDs are 2 bytes in the packet header (bytes 6–7). The high byte encodes the service category; the low byte encodes the sub-service or mode.

### Low Byte Conventions

| Low byte | Meaning |
|----------|---------|
| `0x00` | Control / query |
| `0x01` | Response |
| `0x20` | Data / payload |

### Core Services

| Service ID | Name | Description |
|------------|------|-------------|
| `0x80-00` | Auth Control | Session management, time sync |
| `0x80-20` | Auth Data | Authentication with payload |
| `0x80-01` | Auth Response | Glasses auth acknowledgment |

### Feature Services

| Service ID | Name | Description |
|------------|------|-------------|
| `0x04-20` | Display Wake | Activate display |
| `0x06-20` | Teleprompter | Text display, scripts, scrolling |
| `0x07-20` | Dashboard | Widget data (calendar, weather, etc.) |
| `0x09-00` | Device Info | Version, firmware info |
| `0x0B-20` | Conversate | Speech transcription |
| `0x0C-20` | Tasks | Todo list items |
| `0x0D-00` | Configuration | Device settings |
| `0x0E-20` | Display Config | Display parameters |
| `0x11-20` | Conversate (alt) | Alternative conversate service ID |
| `0x20-20` | Commit | Confirm / commit changes |
| `0x81-20` | Display Trigger | Wake / activate display |

---

## Authentication: 7-Packet Handshake

Every session begins with a 7-packet authentication sequence. This must complete before any feature commands are sent.

### Handshake Overview

```
Phone                          Glasses
  │                               │
  │── Packet 1: Capability Query ─►│  (0x80-00, type=0x04)
  │◄─ Packet 2: Capability Resp  ──│  (0x80-01, type=0x05)
  │── Packet 3: Time Sync ────────►│  (0x80-00, type=0x80, with timestamp + txn_id)
  │◄─ Packet 4: Time Sync ACK ────│  (0x80-01)
  │── Packet 5: Auth Data ────────►│  (0x80-20)
  │◄─ Packet 6: Auth ACK ─────────│  (0x80-01)
  │── Packet 7: Session Start ────►│  (0x80-00)
  │◄─ Session Established ─────────│
```

### Auth Packet Types

| Type | Direction | Meaning |
|------|-----------|---------|
| `0x04` | Phone → Glasses | Capability query |
| `0x05` | Glasses → Phone | Capability response |
| `0x80` | Phone → Glasses | Time sync with transaction ID |

### Python Auth Implementation

```python
import time
import struct

def send_auth_sequence(left_ble, right_ble):
    """Send 7-packet auth handshake to both arms."""
    timestamp = int(time.time())
    txn_id = 0x01  # arbitrary transaction ID

    # Packet 1: Capability query (0x80-00, type=0x04)
    pkt1 = build_packet(seq=1, service_hi=0x80, service_lo=0x00,
                        payload=bytes([0x04]))
    left_ble.write(pkt1)
    right_ble.write(pkt1)

    # Wait for capability response (0x80-01, type=0x05)
    resp = left_ble.read_notify()
    # ... parse response ...

    # Packet 3: Time sync (0x80-00, type=0x80)
    ts_bytes = struct.pack('>I', timestamp)  # big-endian timestamp
    payload = bytes([0x80]) + ts_bytes + bytes([txn_id])
    pkt3 = build_packet(seq=3, service_hi=0x80, service_lo=0x00, payload=payload)
    left_ble.write(pkt3)
    right_ble.write(pkt3)

    # ... continue handshake ...
```

> ⚠️ The exact byte layout of each auth packet is still partially under community research. The sequence above is confirmed working from the `i-soxi/even-g2-protocol` project. See the [EvenRealities Discord](https://discord.gg/arDkX3pr) RE channel for the latest captures.

---

## Command Reference

### TouchBar Events (Glasses → Phone)

These are **incoming** events from the glasses. Your app receives them on the notify characteristic (0x5402).

| Bytes | Event | Notes |
|-------|-------|-------|
| `0xF5 0x01` | Single tap | Right TouchBar: next page. Left TouchBar: read notification detail |
| `0xF5 0x00` | Double tap | Close feature / turn off display |
| `0xF5 0x04` | Triple tap | Toggle Silent Mode (left arm) |
| `0xF5 0x05` | Triple tap | Toggle Silent Mode (right arm) |
| `0xF5 0x17` | Long press | Start Even AI (user held left TouchBar) |

### Even AI Sub-Commands (Glasses → Phone, via `0xF5`)

| Sub-cmd | Hex | Description |
|---------|-----|-------------|
| 0 | `0xF5 0x00` | Exit to dashboard manually |
| 1 | `0xF5 0x01` | Page up (left BLE) / page down (right BLE) in manual mode |
| 23 | `0xF5 0x17` | Start Even AI — notify phone to activate |
| 24 | `0xF5 0x18` | Stop Even AI recording — recording ended |

### Microphone Control (Phone → Glasses)

Command: `0x0E`

| Field | Value | Meaning |
|-------|-------|---------|
| Command | `0x0E` | Microphone control |
| enable | `0x00` | Disable MIC |
| enable | `0x01` | Enable MIC |

> ⚠️ Send microphone commands to the **right arm only**.

**Enable microphone:**
```
0x0E 0x01
```

**Disable microphone:**
```
0x0E 0x00
```

**Glasses response:**
```
0x0E [rsp_status] [enable]

rsp_status:
  0xC9 = Success
  0xCA = Failure

enable:
  0x00 = MIC disabled
  0x01 = MIC enabled
```

### Receive Microphone Audio (Glasses → Phone)

Command: `0xF1`

| Field | Range | Description |
|-------|-------|-------------|
| Command | `0xF1` | Microphone audio data |
| seq | 0–255 | Sequence number of this audio chunk |
| data | variable | Raw audio data (LC3 format) |

> ⚠️ **Audio format on native BLE path is LC3** (not PCM). This differs from the SDK path which delivers PCM S16LE at 16kHz. LC3 is a compressed codec — you must decode it before processing.

```python
# Receive audio frame
def on_notify(data: bytes):
    if data[0] == 0xF1:
        seq = data[1]
        audio_lc3 = data[2:]
        # Decode LC3 → PCM before STT processing
        pcm = lc3_decode(audio_lc3)
        process_audio(seq, pcm)
```

### Send AI Result to Glasses (Phone → Glasses)

Command: `0x4E` — used for both AI results and plain text display.

| Field | Range | Description |
|-------|-------|-------------|
| Command | `0x4E` | AI result / text display |
| seq | 0–255 | Sequence number of this package |
| total_package_num | 1–255 | Total packages in this transmission |
| current_package_num | 0–255 | Current package number (0-indexed) |
| newscreen | byte | Screen status (see below) |
| new_char_pos0 | byte | Higher 8 bits of new character position |
| new_char_pos1 | byte | Lower 8 bits of new character position |
| current_page_num | 0–255 | Current page number |
| max_page_num | 1–255 | Total number of pages |
| data | variable | Text content for this package |

### `newscreen` Byte Encoding

The `newscreen` byte encodes two nibbles:

**Lower 4 bits — Screen Action:**

| Value | Meaning |
|-------|---------|
| `0x01` | Display new content |

**Upper 4 bits — Status:**

| Value | Meaning |
|-------|---------|
| `0x30` | Even AI displaying (automatic mode, default) |
| `0x40` | Even AI display complete (last page of automatic mode) |
| `0x50` | Even AI manual mode |
| `0x60` | Even AI network error |
| `0x70` | Text Show (plain text, not AI) |

**Combined examples:**

| Combined | Meaning |
|----------|---------|
| `0x31` | New content + Even AI displaying |
| `0x41` | New content + Even AI complete (last page) |
| `0x51` | New content + Even AI manual mode |
| `0x61` | New content + Even AI network error |
| `0x71` | New content + Text Show |

### Text Display (Plain Text, Phone → Glasses)

Uses the same `0x4E` command as AI results, but with `newscreen` upper nibble = `0x70`.

**Text layout constraints (native BLE path):**

| Property | Value |
|----------|-------|
| Display width | **488 pixels** (not 576 — narrower than SDK path) |
| Font size | 21 (customizable) |
| Lines per screen | 5 |
| Packets per screen | 2 (first 3 lines = packet 1, last 2 lines = packet 2) |

> ⚠️ The native BLE text display uses **488px width** (not 576px). This is the Even AI display width. The SDK path uses the full 576px canvas.

**Text sending algorithm:**

```python
def send_text(left_ble, right_ble, text: str, display_width: int = 488, font_size: int = 21):
    """
    Send plain text to glasses via native BLE protocol.
    """
    # Step 1: Word-wrap text to display width
    lines = word_wrap(text, display_width, font_size)

    # Step 2: Group into screens (5 lines per screen)
    LINES_PER_SCREEN = 5
    screens = [lines[i:i+LINES_PER_SCREEN] for i in range(0, len(lines), LINES_PER_SCREEN)]
    max_page = len(screens)

    seq = 0
    for page_num, screen_lines in enumerate(screens):
        # Step 3: Split screen into 2 packets (3 lines + 2 lines)
        packet_1_lines = screen_lines[:3]
        packet_2_lines = screen_lines[3:]

        for pkt_idx, pkt_lines in enumerate([packet_1_lines, packet_2_lines]):
            if not pkt_lines:
                continue

            newscreen = 0x71  # New content + Text Show
            payload = bytes([
                0x4E,                    # Command
                seq & 0xFF,              # seq
                2,                       # total_package_num (2 per screen)
                pkt_idx,                 # current_package_num (0-indexed)
                newscreen,               # newscreen
                0x00,                    # new_char_pos0
                0x00,                    # new_char_pos1
                page_num & 0xFF,         # current_page_num
                max_page & 0xFF,         # max_page_num
            ]) + '\n'.join(pkt_lines).encode('utf-8')

            packet = build_packet(seq, 0x06, 0x20, payload)

            # Send left first, wait for ACK, then right
            left_ble.write(packet)
            wait_for_ack(left_ble)
            right_ble.write(packet)
            wait_for_ack(right_ble)

            seq = (seq + 1) & 0xFF
```

---

## Image Transmission Protocol

### Image Specifications (Native BLE Path)

| Property | Value |
|----------|-------|
| Format | **1-bit BMP** (black and white, not greyscale) |
| Dimensions | **576 × 136 pixels** |
| Packet size | **194 bytes** per data packet |
| Command | `0x15` |
| End command | `0x20 0x0D 0x0E` |
| CRC | CRC32/XZ big-endian (not CRC-16/CCITT) |
| Storage address | `0x00 0x1C 0x00 0x00` (prepended to first packet only) |

> ⚠️ **Native BLE images are 1-bit BMP at 576×136px** — completely different from the SDK path which uses 4-bit greyscale PNG at up to 200×100px. These are two entirely separate image systems.

### Image Transmission Sequence

```
1. Divide BMP data into 194-byte chunks
2. For each chunk:
   a. Prepend [0x15, seq & 0xFF]
   b. If first chunk: also prepend [0x00, 0x1C, 0x00, 0x00] (storage address)
   c. Send to left BLE
   d. Send to right BLE (simultaneously — image is exception to left-first rule)
3. After last chunk: send end command [0x20, 0x0D, 0x0E] to both arms
4. After end command ACK: send CRC check via 0x16 command
```

### Image Packet Format

**First packet:**
```
[0x15] [seq] [0x00] [0x1C] [0x00] [0x00] [data0...data193]
  ↑      ↑    └──────────────────────────┘  └──────────────┘
Command  Seq   Storage address (first only)   194 bytes BMP data
```

**Subsequent packets:**
```
[0x15] [seq] [data0...data193]
  ↑      ↑    └──────────────┘
Command  Seq   194 bytes BMP data
```

**End command:**
```
[0x20] [0x0D] [0x0E]
```

**CRC check command:**
```
Command: 0x16
Field: crc (CRC32/XZ big-endian, calculated over storage address + all BMP data)
```

### Python Image Transmission

```python
import struct
import zlib

def crc32_xz(data: bytes) -> int:
    """CRC32/XZ (same as Python's zlib.crc32 with standard init)."""
    return zlib.crc32(data) & 0xFFFFFFFF

def send_bmp_image(left_ble, right_ble, bmp_data: bytes):
    """
    Send a 1-bit 576x136 BMP image to both arms.
    Left and right can receive image packets simultaneously.
    """
    STORAGE_ADDRESS = bytes([0x00, 0x1C, 0x00, 0x00])
    PACKET_SIZE = 194
    seq = 0

    # Split BMP data into 194-byte chunks
    chunks = [bmp_data[i:i+PACKET_SIZE] for i in range(0, len(bmp_data), PACKET_SIZE)]

    for i, chunk in enumerate(chunks):
        if i == 0:
            # First packet: prepend storage address
            packet_data = bytes([0x15, seq & 0xFF]) + STORAGE_ADDRESS + chunk
        else:
            packet_data = bytes([0x15, seq & 0xFF]) + chunk

        # Image packets can be sent to both arms simultaneously
        left_ble.write(packet_data)
        right_ble.write(packet_data)

        seq = (seq + 1) & 0xFF

    # Send end command to both arms
    end_cmd = bytes([0x20, 0x0D, 0x0E])
    left_ble.write(end_cmd)
    right_ble.write(end_cmd)

    # Wait for end command ACK
    wait_for_ack(left_ble)
    wait_for_ack(right_ble)

    # Calculate CRC over storage address + full BMP data
    crc_input = STORAGE_ADDRESS + bmp_data
    crc_value = crc32_xz(crc_input)
    crc_bytes = struct.pack('>I', crc_value)  # big-endian

    # Send CRC check command
    crc_cmd = bytes([0x16]) + crc_bytes
    left_ble.write(crc_cmd)
    right_ble.write(crc_cmd)
```

---

## Teleprompter Protocol (Service 0x06-20)

The teleprompter service displays scrollable text on the G2 glasses. It supports manual scroll mode (swipe to scroll) and AI auto-scroll mode (voice-triggered).

### Teleprompter Message Sequence

```
1. Auth Packets (7 packets)          ← Establish session
2. Display Config (0x0E-20, type=2)  ← Configure display parameters
3. Teleprompter Init (0x06-20, type=1) ← Select script, set scroll mode
4. Content Pages 0–9 (0x06-20, type=3) ← First batch
5. Mid-Stream Marker (0x06-20, type=255) ← Required marker
6. Content Pages 10–11 (0x06-20, type=3) ← Second batch
7. Sync Trigger (0x80-00, type=14)   ← Trigger rendering
8. Content Pages 12+ (0x06-20, type=3) ← Final pages
```

### Message Types

#### Type 1: Init / Select Script

```
08-01       Type = 1 (init)
10-XX       msg_id (varint)
1A-XX       Field 3, length (settings block)
  08-XX     Script index (0 or 1)
  12-XX     Display settings sub-block
    08-01   Sub-field 1 = 1
    10-00   Sub-field 2 = 0
    18-00   Sub-field 3 = 0
    20-8B-02  Sub-field 4 = 267 (display width)
    28-XX-XX  Sub-field 5 = content height (varint)
    30-E6-01  Sub-field 6 = 230 (line height)
    38-XX-XX  Sub-field 7 = viewport height (varint)
    40-05   Sub-field 8 = 5 (font size)
    48-XX   Sub-field 9 = scroll mode
```

**Scroll Mode (field 9):**

| Value | Mode | Indicator |
|-------|------|-----------|
| `0x00` | Manual | Shows "M" indicator on glasses |
| `0x01` | AI auto-scroll | Shows animation |

#### Type 2: Script List

```
08-02       Type = 2 (list)
10-XX       msg_id
22-XX       Field 4, length (list content)
  0A-XX     Script entry
    0A-XX   Script ID (string)
    12-XX   Script title (string)
```

#### Type 3: Content Page

```
08-03       Type = 3 (content)
10-XX       msg_id (varint)
2A-XX       Field 5, length (content block)
  08-XX     Page number (0-indexed, varint)
  10-0A     Line count = 10
  1A-XX     Field 3, length (text)
  [TEXT]    UTF-8 text with \n separators
```

**Text format rules:**
- Each page has exactly **10 lines**
- Lines separated by `\n` (0x0A)
- Text starts with `\n` (leading newline)
- Text ends with ` \n` (space + newline)
- Lines wrap at ~**25 characters**

#### Type 4: Content Complete

```
08-04       Type = 4 (complete)
10-XX       msg_id
32-XX       Field 6, length (completion info)
  08-00     Start page (0)
  10-XX     Total pages (varint)
  18-XX     Total lines (varint)
```

#### Type 255 (0xFF): Mid-Stream Marker

Required marker sent during content streaming. Must be sent between pages 9 and 10.

```
08-FF-01    Type = 255 (varint encoding)
10-XX       msg_id
6A-04       Field 13, length 4
  08-00     Sub-field 1 = 0
  10-06     Sub-field 2 = 6
```

### Teleprompter Display Characteristics

| Property | Value |
|----------|-------|
| Characters per line | ~25 |
| Lines per page | 10 |
| Visible lines at once | ~7 |
| Minimum content | ~10 pages (less may not render) |
| Script index | Must reference existing script (0 or 1) |

### Scroll Bar Sizing

The scroll bar size is controlled by init packet fields:

```
Scroll bar ratio = viewport_height / content_height

Example (Bee Movie script, 140 lines):
  content_height = 2665
  viewport_height = 1294
  ratio = ~49% (scroll bar takes half the track)
```

### Python Teleprompter Example

```python
def send_teleprompter(left_ble, right_ble, text: str):
    """Display scrollable text on G2 glasses via teleprompter protocol."""

    # Step 1: Auth
    send_auth_sequence(left_ble, right_ble)

    # Step 2: Display config
    send_display_config(left_ble, right_ble, seq=8, msg_id=20)

    # Step 3: Init teleprompter (manual mode, script index 1)
    send_teleprompter_init(left_ble, right_ble, seq=9, msg_id=21,
                           script_index=1, mode=0)

    # Step 4: Paginate text (10 lines per page, ~25 chars per line)
    pages = paginate_text(text, chars_per_line=25, lines_per_page=10)

    # Step 5: Send pages 0–9
    for page_num in range(min(10, len(pages))):
        send_content_page(left_ble, right_ble,
                          seq=10+page_num, msg_id=22+page_num,
                          page_num=page_num, text=pages[page_num])

    # Step 6: Mid-stream marker (required)
    send_type_255(left_ble, right_ble, seq=20, msg_id=32)

    # Step 7: Send pages 10–11
    for page_num in range(10, min(12, len(pages))):
        send_content_page(left_ble, right_ble,
                          seq=21+(page_num-10), msg_id=33+(page_num-10),
                          page_num=page_num, text=pages[page_num])

    # Step 8: Sync trigger
    send_sync(left_ble, right_ble, seq=23, msg_id=35)

    # Step 9: Send remaining pages
    for page_num in range(12, len(pages)):
        send_content_page(left_ble, right_ble,
                          seq=24+(page_num-12), msg_id=36+(page_num-12),
                          page_num=page_num, text=pages[page_num])

    # Step 10: Content complete
    send_content_complete(left_ble, right_ble,
                          total_pages=len(pages),
                          total_lines=len(pages) * 10)
```

---

## Even AI Flow (Native BLE Path)

The full Even AI voice assistant flow over native BLE:

```
User                    Glasses                 Phone App               LLM API
 │                         │                       │                       │
 │── Long press left ──────►│                       │                       │
 │   TouchBar               │── 0xF5 0x17 ─────────►│                       │
 │                         │                       │── 0x0E 0x01 ──────────►│ (enable right mic)
 │                         │◄── 0x0E 0xC9 0x01 ────│                       │
 │                         │── 0xF1 [seq] [lc3] ───►│                       │
 │                         │── 0xF1 [seq] [lc3] ───►│                       │
 │   (speaking...)         │── ... ─────────────────►│                       │
 │── Release TouchBar ─────►│                       │                       │
 │                         │── 0xF5 0x18 ───────────►│                       │
 │                         │                       │── 0x0E 0x00 ──────────►│ (disable mic)
 │                         │                       │── LC3→PCM→STT ─────────►│
 │                         │                       │◄── transcript ──────────│
 │                         │                       │── LLM prompt ───────────►│
 │                         │                       │◄── LLM response ─────────│
 │                         │◄── 0x4E [AI result] ──│                       │
 │◄── Display result ──────│                       │                       │
```

### Even AI State Machine

```
IDLE
  │
  │ 0xF5 0x17 (long press)
  ▼
RECORDING
  │ Send 0x0E 0x01 to right arm
  │ Receive 0xF1 audio frames (LC3)
  │
  │ 0xF5 0x18 (release) or 30s timeout
  ▼
PROCESSING
  │ Send 0x0E 0x00 (disable mic)
  │ Decode LC3 → PCM
  │ STT → transcript
  │ LLM → response
  │
  │ Response ready
  ▼
DISPLAYING (automatic mode)
  │ Send 0x4E packets (newscreen=0x31)
  │ Last page: newscreen=0x41
  │
  │ 0xF5 0x01 (single tap during display)
  ▼
DISPLAYING (manual mode)
  │ newscreen=0x51
  │ Left tap = page up, Right tap = page down
  │
  │ 0xF5 0x00 (double tap)
  ▼
IDLE
```

---

## Display Architecture: Dual Channel

The G2 uses a **dual-channel display architecture**:

| Channel | UUID Suffix | Handle | Purpose |
|---------|-------------|--------|---------|
| Content Channel | `0x5401` | `0x0842` | What to display (text, data payload) |
| Rendering Channel | `0x6402` | `0x0864` | How to display (positioning, styling, 204-byte binary commands) |

The rendering channel (`0x6402`) sends **204-byte binary rendering packets** that control display positioning and styling. The exact format of these packets is still under community research.

```
Content Channel (0x5401):
  ← Text content, AI results, widget data

Rendering Channel (0x6402):
  ← 204-byte binary display commands
  ← Positioning, layout, styling
  ← Under active reverse engineering
```

---

## Community Protocol Status

| Feature | Status | Notes |
|---------|--------|-------|
| BLE Connection | ✅ Working | Standard BLE, no special pairing |
| Authentication | ✅ Working | 7-packet handshake confirmed |
| Teleprompter | ✅ Working | Custom text display confirmed |
| Calendar Widget | ✅ Working | Display events on glasses |
| Notifications | ⚠️ Partial | Metadata only (app name + count) |
| Even AI | 🔬 Research | Protocol identified, not fully documented |
| Navigation | 🔬 Research | High display traffic observed |
| Rendering Channel (0x6402) | 🔬 Research | 204-byte binary format under investigation |
| Image display (BMP) | ✅ Working | 1-bit 576×136px confirmed |
| Microphone (LC3) | ✅ Working | Right arm only, LC3 codec |

---

## Community Resources

| Resource | URL | Notes |
|----------|-----|-------|
| Protocol repo | [i-soxi/even-g2-protocol](https://github.com/i-soxi/even-g2-protocol) | BLE UUIDs, packet structure, teleprompter |
| Official demo | [even-realities/EvenDemoApp](https://github.com/even-realities/EvenDemoApp) | Official BLE command reference |
| Community notes | [nickustinov/even-g2-notes](https://github.com/nickustinov/even-g2-notes) | Architecture, display, input |
| Discord RE channel | [EvenRealities Discord](https://discord.gg/arDkX3pr) | Live reverse engineering discussion |
| Protobuf defs | [i-soxi/even-g2-protocol/proto/](https://github.com/i-soxi/even-g2-protocol/tree/main/proto) | Protobuf definitions for payload encoding |
| Teleprompter example | [i-soxi/even-g2-protocol/examples/teleprompter/](https://github.com/i-soxi/even-g2-protocol/tree/main/examples/teleprompter) | Working Python teleprompter |

---

## SDK vs Native BLE: Side-by-Side Comparison

| Aspect | SDK Path (Even Hub) | Native BLE Path |
|--------|--------------------|-----------------| 
| Image format | 4-bit greyscale PNG, max 200×100px | 1-bit BMP, 576×136px |
| Image CRC | None (SDK handles) | CRC32/XZ big-endian |
| Audio format | PCM S16LE, 16kHz | LC3 compressed |
| Text width | 576px canvas | 488px display width |
| Text font | LVGL firmware font | Firmware font (same) |
| Scroll | SDK container model | Teleprompter service |
| Auth | SDK handles | 7-packet handshake |
| Dual BLE | SDK handles | Manual left/right management |
| Sequence counter | SDK handles | Manual per-arm counter |
| CRC | SDK handles | Manual CRC-16/CCITT per packet |
| Protobuf | SDK handles | Manual encoding |

---

## Critical Gotchas Summary

| Gotcha | Detail |
|--------|--------|
| Always left first | Send to left arm, wait for ACK, then right — except image packets |
| Image exception | Image packets can be sent to both arms simultaneously |
| Microphone = right only | `0x0E` command goes to right arm only |
| Audio is LC3, not PCM | Native BLE delivers LC3 compressed audio — must decode before STT |
| Image is 1-bit BMP | Not greyscale PNG — completely different from SDK path |
| Image dimensions | 576×136px (not 576×288 or 200×100) |
| Image CRC is CRC32/XZ | Not CRC-16/CCITT — different algorithm from packet CRC |
| Storage address in first packet only | `0x00 0x1C 0x00 0x00` prepended to first image packet only |
| Teleprompter needs mid-stream marker | Type 255 packet required between pages 9 and 10 |
| Minimum teleprompter content | Less than ~10 pages may not render |
| Script index must exist | Must reference existing script (0 or 1) |
| Rendering channel format unknown | 0x6402 is 204-byte binary — under research |
| Auth must complete first | No feature commands before 7-packet handshake |
| Sequence counter per arm | Each arm has its own independent counter |
| CRC scope | CRC-16/CCITT calculated over payload only — skip 8-byte header |

---