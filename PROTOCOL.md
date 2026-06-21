# AiLens BLE Protocol Reference

This document explains the full BLE GATT profile and command protocol used in the
`ailens-web-demo` project. It is derived from the iOS SDK source code
(`AiLensBLE`, releases 20250912 and 20260510).

---

## 1. GATT Profile

### 1.1 Device identification

The glasses advertise a BLE device name that contains the string `"Ai Lens"`
(e.g. `"Ai Lens 071E"`). Web Bluetooth scans for all devices and filters by
this substring after the user selects a device.

### 1.2 Services and characteristics

All integers in UUIDs are in little-endian order on the wire.

| Role | UUID | Direction | Properties |
|---|---|---|---|
| **Primary Service** | `00010000-0000-1000-8000-00805f9b5a6b` | — | — |
| **Write characteristic** | `00010001-0000-1000-8000-00805f9b5a6b` | host → device | Write Without Response |
| **TX characteristic** | `00010002-0000-1000-8000-00805f9b5a6b` | device → host | Notify |
| **Handshake characteristic** | `00010003-0000-1000-8000-00805f9b5a6b` | host → device, device → host | Write Without Response, Read |
| **Streaming characteristic** | `00030002-0000-1000-8000-00805f9b5a6b` | device → host | Notify |

**Write characteristic** is the single entry point for every command sent to the
glasses (canvas drawing, translation, pairing, disconnect, etc.).

**TX characteristic** carries device-to-host notifications: ACK responses,
pairing state machine events, and IMU sensor data.

**Handshake characteristic** is used only during initial connection to exchange
the authentication token.

**Streaming characteristic** carries raw microphone audio data (not used in this
demo).

---

## 2. Connection and Pairing Sequence

Connection is a three-phase protocol. It must complete successfully before any
command can be sent.

### Phase 1 — Token handshake (Handshake characteristic `00010003`)

1. Write the 27-byte static token to the Handshake characteristic
   (Write Without Response or Write With Response — try with response first):

   ```
   A4 6A D6 E0 35 82 00 00
   10 01 C0 EF 1A 03 2B A8
   40 14 87 30 1C B8 A5 61
   EC 77 8D
   ```

2. Wait ~250 ms.
3. Read the Handshake characteristic value.
4. Verify the response: must be exactly **18 bytes** and `response[0]` must be
   `0x64`, `0x65`, or `0x66`. Any other value means failure (retry up to 3×).

This step authenticates the host application to the glasses firmware.

### Phase 2 — Pairing (Write characteristic `00010001`, TX characteristic `00010002`)

The glasses **require a physical long-press of the touch bar** to complete
pairing. The firmware shows a pairing UI after receiving `glassShowLongPressCommand`
and waits up to ~30 seconds for the long-press; if no press occurs it disconnects.

On iOS the pairing state machine is fully event-driven via TX notifications:

1. Glasses → TX `[0x01]` → host sends `glassShowLongPressCommand`
2. User long-presses touch bar
3. Glasses → TX `longPressConfirmResponse` (`4F 42 00 00 01 00 02 00 00`) → host sends `appShowPairCommand` + `directPairCommand` + `confirmationCommand`
4. Glasses → TX `45 4D 68 00` → host declares ready

On **Windows**, `startNotifications()` fails so TX events are never received.
The demo works around this by showing a **"Continue (long-press done)"** button
in the UI. The user long-presses the glasses, then clicks the button, and only
then does the demo send the remaining pairing commands.

Commands sent to the Write characteristic and their timing:

| Step | Trigger | Bytes (hex) | Purpose |
|---|---|---|---|
| `glassShowLongPressCommand` | Immediately after handshake | `45 4D 00 00 00 00 00 00` | Tell glasses to show pairing UI |
| `appShowPairCommand` | After user clicks "Continue" | `45 4D 67 00 01 00 02 00 01` | App confirms it wants to pair |
| `directPairCommand` | 200 ms after above | `45 4D 0C 01 00 00 00 00` | Request direct (non-interactive) pairing |
| `confirmationCommand` | 200 ms after above | `45 4D 69 00 01 00 02 00 01` | Confirm pairing completion |

After this sequence the glasses are ready to receive commands.

### Phase 3 — Ready signal (TX characteristic, iOS only)

On iOS, the glasses send a TX notification starting with `45 4D 68 00` to signal
readiness. On Windows this notification is not received because TX notifications
fail, so the demo declares ready 500 ms after `confirmationCommand` instead.

### Disconnect

Send to Write characteristic:

```
45 4D 73 00 00 00 00 00
```

### Windows-specific notes

- Do **not** pair the glasses at the Windows OS level (Settings → Bluetooth).
  OS-level pairing blocks Web Bluetooth from accessing GATT characteristics.
  If a Windows Bluetooth pairing popup appears during the glasses long-press,
  dismiss/cancel it — do not complete it.
- `startNotifications()` on the TX characteristic fails with
  `GATT Error: invalid attribute length` on Windows. The demo catches and ignores
  this error. ACK waiting falls back to a 1-second timeout, and the long-press
  confirmation is handled by a UI button instead (see Phase 2 above).
- `device.gatt.connect()` may time out on the first attempt. The demo retries up
  to 5 times with a 1-second delay between attempts.
- A 500 ms delay is inserted after `gatt.connect()` before service discovery to
  give the Windows Bluetooth stack time to settle.
- Sending `glassShowLongPressCommand` starts a ~30-second firmware timer on the
  glasses. If the user does not long-press within that window, the glasses
  disconnect. Always long-press promptly after the "Continue" button appears.

---

## 3. Command Frame Formats

All multi-byte integers are **little-endian** unless noted otherwise.

There are two frame types used in this demo.

### 3.1 Simple fixed-length command

Used for: pairing commands, disconnect, canvas open/close.

```
Byte 0–1:   45 4D          Magic "EM"
Byte 2–3:   [cmdType LE]   Command type (2 bytes)
Byte 4–5:   [len LE]       Payload length (2 bytes)
Byte 6–7:   [len2 LE]      Payload length repeated (2 bytes)
Byte 8+:    [payload]      Command-specific payload bytes
```

Examples:

| Command | Bytes |
|---|---|
| Open Canvas | `45 4D 79 00 00 00 00 00` |
| Close Canvas | `45 4D 7A 00 00 00 00 00` |
| Disconnect | `45 4D 73 00 00 00 00 00` |
| IMU subscribe | `45 4D 12 00 01 00 02 00 01` |
| IMU unsubscribe | `45 4D 12 00 01 00 02 00 02` |
| Mic start | `45 4D 2F 00 01 00 02 00 01` |
| Mic stop | `45 4D 2F 00 01 00 02 00 00` |

### 3.2 Binary packet (`CMD_BINARY_PACKET`, cmdType `0x001A`)

Used for: canvas drawing (text, rect, line, clear) and simultaneous translation.

This format wraps a larger payload across one or more BLE frames, with CRC-32
checksums for integrity.

#### Overall structure of one BLE frame

```
┌─────────────────────────────────────────────────────┐
│ CMD Header       8 bytes                            │
│   45 4D                  magic                      │
│   1A 00                  cmdType = 0x001A           │
│   [valueLen LE 2]        size of (PacketHead+Data)  │
│   [valueLen×2 LE 2]      same value doubled         │
├─────────────────────────────────────────────────────┤
│ Packet Head      10 bytes                           │
│   [binType 1]            0x50=canvas, 0x67=transl.  │
│   [sliceType 1]          0x00=begin 0x01=mid 0x02=end│
│   [CRC32(chunk) 4]       CRC of this frame's data   │
│   [frameIndex 4]         0-based frame counter      │
├─────────────────────────────────────────────────────┤
│ Data chunk       up to (MTU − 18) bytes             │
│   slice of (CommData Head + Payload)                │
└─────────────────────────────────────────────────────┘
```

- `sliceType = 0x00` (begin) for the first frame, `0x01` (middle) for
  intermediate frames, `0x02` (end) for the last frame. Single-frame messages
  use `0x00` only.
- `MTU` is the maximum total frame size. iOS negotiates ~185 bytes. The demo
  tries 185 first and retries with 23 on write failure.

#### CommData Head (25 bytes, sits at the front of the data portion)

```
Byte 0:     [binType]           same as Packet Head binType
Byte 1–4:   [payloadLen LE 4]   byte length of the Payload section
Byte 5–8:   [CRC32(payload) 4]  CRC of the full Payload section
Byte 9–24:  [reserved 16]       canvas: 0x00×16 / translation: 0xFF×16
```

The full data stream split across frames is: `CommData Head || Payload`.

---

## 4. Canvas Drawing Commands

Canvas commands use `binType = 0x50` and `reserved = 0x00 × 16`.

They require an explicit Open Canvas / draw / Close Canvas sequence.

### Canvas session

```
beginCanvas  → send Open Canvas command  (45 4D 79 00 00 00 00 00)
  writeText / writeLine / writeRect / clearCanvas  (binary packet frames)
endCanvas    → send Close Canvas command (45 4D 7A 00 00 00 00 00)
```

### Canvas payload format

The Payload section begins with a `0x60` wrapper:

```
60                      PL_ALGO_DRAW_CANVAS marker
[innerLen LE 2]         byte length of the object bytes below
[object bytes]          see object types below
```

#### Object types

**Text (type `0x01`)**
```
01
[textSize LE 2]
[x LE 2]
[y LE 2]
[width LE 2]
[height LE 2]
[UTF-8 text bytes]
```

**Rectangle (type `0x02`)**
```
02
[lineWidth LE 2]
[x LE 2]
[y LE 2]
[width LE 2]
[height LE 2]
[fill: 01=filled, 00=outline]
```

**Line (type `0x03`)**
```
03
[lineWidth LE 2]
[x1 LE 2]
[y1 LE 2]
[x2 LE 2]
[y2 LE 2]
```

**Clear (type `0x05`)**
```
05
[x LE 2]        always 0x0000
[y LE 2]        always 0x0000
[width LE 2]    always 0x0280 (640)
[height LE 2]   always 0x01E0 (480)
```

Canvas dimensions are **640 × 480** pixels.

---

## 5. Simultaneous Translation Command

Translation uses `binType = 0x67` and `reserved = 0xFF × 16`.
No canvas open/close is needed — the command is self-contained.

### Payload / inner content structure

```
[sessionId 1]           0–255, incremented on each call
[TLV for original]      type=0x89, len_lo, len_hi, UTF-8 bytes
[TLV for translation]   type=0x8A, len_lo, len_hi, UTF-8 bytes
```

Each TLV:
```
[type 1]        0x89 = original, 0x8A = translation
[length LE 2]   byte length of the value
[value]         UTF-8 encoded string
```

### Key differences vs canvas commands

| Property | Canvas | Simultaneous Translation |
|---|---|---|
| `binType` | `0x50` | `0x67` |
| CommData Head reserved bytes | `0x00 × 16` | `0xFF × 16` |
| CRC field in CommData Head | CRC32 of wrapped payload (`0x60` + inner) | CRC32 of raw inner bytes |
| Requires canvas open/close | Yes | No |
| Multi-frame splitting | Rare (short payloads) | Common (large text) |

### Session ID

`sessionId` is a 1-byte counter (1–255) that identifies a translation update.
The demo increments it after each successful send. The glasses use it to detect
duplicate or out-of-order updates.

---

## 6. CRC-32 Algorithm

The checksum function is `metaCRC32` from `CRC.swift`. It is a custom CRC-32
variant with:

- Initial value: `0x00000000` (not the standard `0xFFFFFFFF`)
- No final XOR
- Big-endian shift direction (`crc << 8`, index from `crc >> 24`)
- A custom 256-entry lookup table (embedded in the source)

All CRC-32 values in packet headers are stored **little-endian** (4 bytes).

JavaScript implementation:

```js
function metaCRC32(bytes) {
  let crc = 0;
  for (const b of bytes) {
    crc = ((crc << 8) ^ CRC32_TABLE[((b ^ (crc >>> 24)) & 0xFF)]) >>> 0;
  }
  return crc;
}
```

---

## 7. ACK / Response Protocol

After sending a command to the Write characteristic, the glasses reply on the
**TX characteristic** with a notification that begins with `4F 42` ("OB").

The demo waits up to 1000 ms for this ACK before proceeding. If TX notifications
are unavailable (Windows), the 1000 ms timeout acts as the pacing mechanism.

```
TX notification starting with 4F 42 → ACK, proceed to next command
TX notification starting with 45 4D 15 00 → IMU data, ignore in this demo
TX notification starting with 45 4D 68 00 → glasses ready (iOS pairing flow)
TX single byte 01 → long-press prompt (iOS pairing flow)
```

---

## 8. Complete Connection Flow (as implemented in index.html)

### 8.1 Initial connection

```
navigator.bluetooth.requestDevice()
  └─ filter: acceptAllDevices, optionalServices: [SERVICE_UUID]
  └─ validate: device.name includes "Ai Lens"

device.gatt.connect()   ← retry up to 5× with 5 s timeout each
  └─ wait 500 ms

server.getPrimaryService(SERVICE_UUID)
  └─ getCharacteristic(WRITE_UUID)    → writeChar
  └─ getCharacteristic(TX_UUID)       → txChar
  └─ getCharacteristic(HANDSHAKE_UUID)→ hsChar

txChar.startNotifications()           ← best-effort, ignored if fails on Windows
txChar.addEventListener('characteristicvaluechanged', onTX)

hsChar.writeValueWithResponse(HANDSHAKE_TOKEN)   ← 27 bytes
  └─ fallback: writeValueWithoutResponse
wait 250 ms
hsChar.readValue()
  └─ verify: length=18, byte[0] in {0x64, 0x65, 0x66}

writeChar.writeValueWithoutResponse(CMD_GLASS_LONG_PRESS)
  └─ UI shows "Continue (long-press done)" button
  └─ user long-presses touch bar on glasses
  └─ user clicks button in browser   ← replaces TX longPressConfirmResponse on Windows
writeChar.writeValueWithoutResponse(CMD_APP_SHOW_PAIR)
wait 200 ms
writeChar.writeValueWithoutResponse(CMD_DIRECT_PAIR)
wait 200 ms
writeChar.writeValueWithoutResponse(CMD_CONFIRMATION)
wait 500 ms

→ Glasses ready. Translation commands can now be sent.
```

### 8.2 Disconnect behavior

When the GATT connection drops (either user-initiated or unexpected), the demo:

```
gattserverdisconnected event fires
  └─ auto-send stopped, writeChar set to null
  └─ "Continue" button hidden (in case disconnect happened mid-pairing)
  └─ Connect button re-enabled — user must reconnect manually
```

The ~30-second firmware timeout (glasses disconnect if no long-press received)
is avoided by having the user actually long-press before the remaining pairing
commands are sent. See Windows-specific notes in Section 2.

---

## 9. File Structure

```
ailens-web-demo/
└── index.html      Single-file web app. Contains all HTML, CSS, and JavaScript.
```

All protocol logic lives in `index.html`:

| Function | Purpose |
|---|---|
| `metaCRC32(bytes)` | CRC-32 checksum (custom variant from CRC.swift) |
| `u16le(v)` / `u32le(v)` | Little-endian byte encoding helpers |
| `buildTranslationFrames(sessionId, original, translation, mtu)` | Builds one or more BLE frames for the simultaneous translation command |
| `connect()` | Full connection + pairing sequence |
| `waitForUserLongPress()` | Returns a Promise that resolves when the user clicks "Continue (long-press done)"; shows/hides the button |
| `onTX(event)` | TX notification handler; logs all TX bytes including ACKs |
| `sendTranslation()` | Sends original + translation text with incrementing `sendCount` appended |
| `startAuto()` | Starts 2-second auto-send interval |
| `stopAuto()` | Stops auto-send interval |
| `onDisconnected()` | Handles `gattserverdisconnected` event; stops auto-send, hides Continue button, resets UI |
| `disconnect()` | User-initiated disconnect: sends disconnect command and cleans up state |

Key state variables:

| Variable | Purpose |
|---|---|
| `writeChar` | Reference to the Write characteristic; set to `null` on any disconnect |
| `sessionId` | 1-byte translation session counter, wraps 1–255 |
| `sendCount` | Display counter appended to each sent string; reset by Reset Count button |
| `autoTimer` | `setInterval` handle; `null` when auto-send is not running |
| `longPressResolve` | Resolve function for the `waitForUserLongPress()` promise; `null` when not waiting |
