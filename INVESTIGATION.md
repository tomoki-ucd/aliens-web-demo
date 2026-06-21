# Investigation: ~30-Second Disconnect on Windows

## Symptom

On Windows with Chrome Web Bluetooth, the glasses (Ai Lens 071E) disconnect
after approximately **29–31 seconds** from when the `longPressConfirmResponse`
is received. The connection is otherwise healthy: translation frames are sent
and ACKed continuously until the moment of disconnect.

---

## What Was Ruled Out

### 1. Long-press UI timeout
**Theory:** `CMD_GLASS_LONG_PRESS` (CMD_BIND REQ) starts a 30-second timer
waiting for the user to physically press the glasses.

**Result:** Disconnect still occurs ~29 s after `longPressConfirmResponse`, not
from when `CMD_GLASS_LONG_PRESS` was sent. The timer starts from pairing
completion, not from the long-press trigger.

### 2. Missing CCCD / TX notifications not subscribed
**Theory:** `startNotifications()` fails on Windows before pairing
(`"GATT Error: invalid attribute length"`). Glasses might disconnect when the
CCCD is not set.

**Result:** Retrying `startNotifications()` after pairing works and ACKs arrive.
Disconnect still occurs.

### 3. `TX 45 4D 10 00` session keepalive
**Theory:** Glasses send `45 4D 10 00 02 00 04 00 19 00` at ~24 s expecting a
response, and disconnect if none comes.

**Result:** After replying with `CMD_BT_STATUS_POLL`, the `0x0010` message
stopped appearing in logs but the disconnect still happened. `0x0010` is a
side-effect, not the cause.

### 4. TX [0x01] not responded to
**Theory:** Glasses send TX `[0x01]` requesting long-press flow. Ignoring it
causes disconnect.

**Result:** Added handler responding with `CMD_GLASS_LONG_PRESS` +
`CMD_APP_SHOW_PAIR`. Disconnect still occurs.

### 5. Wrong pairing commands (CMD_DIRECT_PAIR / CMD_CONFIRMATION)
**Theory:** Sending `45 4D 0C 01 ...` and `45 4D 69 00 ...` was confusing the
glasses state machine.

**Result:** iOS SDK source confirmed these are defined but **never written to
the peripheral**. Removed them. Pairing flow now cleaner but disconnect persists.

### 6. BT status polling not started early enough
**Theory:** Polling only started from `userPressPairingCheck` handler, but on
Windows that frame never arrives.

**Result:** Added polling start from `longPressConfirmResponse`. Polling runs
continuously but `connectStatus` is always `false`. Disconnect persists.

### 7. CMD_BTCONNECT_BACK_ENABLE = 0 (disable BT auto-reconnect)
**Theory:** Protocol manual §4.57 — sending `CMD_BTCONNECT_BACK_ENABLE` (0x39)
with value=0 right after handshake might stop the firmware from requiring
Classic BT within 30 s.

**Result (2026-06-22 test):** No effect. Disconnect still at ~29 s after
`longPressConfirmResponse`.

---

## Confirmed Root Cause

### Two Bluetooth transports

The glasses use **two independent Bluetooth transports**:

| Transport | Protocol | What carries it | Windows |
|---|---|---|---|
| BLE (GATT) | Commands, display frames, status | Web Bluetooth API | ✓ Works |
| Classic BT (HFP/SPP) | Audio | Native Bluetooth stack | ✗ Not accessible from Chrome |

### The 30-second firmware timer

After `CMD_APP_SHOW_PAIR` (`CMD_SET_PHONE_TYPE` 0x67, §4.102) is received, the
glasses start a **~30-second timer** waiting for Classic BT to connect
(`connectStatus = 1` in BT status poll responses). If Classic BT does not
connect within that window, the glasses drop the BLE connection.

The timer starts from `longPressConfirmResponse` (CMD_BIND RSP), confirmed by
the 2026-06-22 log: longPressConfirmResponse at 02:02:43.8 → disconnect at
02:03:12.8 = exactly **29.0 seconds**.

### Why `45 4D 68 00 ... 01` never arrives on Windows

On iOS/Android:
1. Host sends `CMD_APP_SHOW_PAIR` (set phone type)
2. Glasses advertise for Classic BT pairing (popup appears)
3. User accepts popup → Classic BT pairs
4. Glasses send `CMD_PAIR_STATE_SYNC` REQ (`45 4D 68 00 ... 01`, §4.103)
   as notification that pairing succeeded
5. Host sends `CMD_PAIR_STATE_SYNC` RSP (`CMD_CONNECT_SUCCEED`)
6. Host polls `CMD_REPORT_BT_STATUS` until `connectStatus=1`

On Windows with Chrome:
- Steps 2–3 never complete (Chrome has no Classic BT API)
- `45 4D 68 00 ... 01` is **never sent by the glasses**
- BT status poll always returns `connectStatus=false`
- 30-second timer fires → glasses disconnect BLE

### BT status poll response format (§4.58)

Response to `CMD_REPORT_BT_STATUS` (0x3A) REQ:
```
4F 42 3A 00 03 00 06 00 | 00 | 00 | 00
                           ^    ^    ^
                           |    |    connectStatus (1=Classic BT connected)
                           |    addressSaved (1=Classic BT address saved/bonded)
                           status (0=SUCCESS)
```

On Windows: always `addressSaved=false connectStatus=false`.
On iOS with Classic BT connected: `connectStatus=true` → host stops polling and
declares ready.

### Protocol manual findings (ThinkAR Glass Protocol Manual V1.0, 2023-10-07)

| Section | Command | Key finding |
|---|---|---|
| §4.102 | `CMD_SET_PHONE_TYPE` (0x67) | REQ payload: `uint8_t type` — Android=0x0A, iOS=0x01 |
| §4.103 | `CMD_PAIR_STATE_SYNC` (0x68) | Glasses→Host REQ payload: `status(1=Success, 0=Failed)` |
| §4.57 | `CMD_BTCONNECT_BACK_ENABLE` (0x39) | REQ payload: `0=off / 1=on` — disable BT auto-reconnect |
| §4.58 | `CMD_REPORT_BT_STATUS` (0x3A) | RSP payload: `status + addressSaved + connectStatus` |

No "BLE-only mode" or command to skip the Classic BT requirement was found in
the manual.

---

## Actual Pairing Flow on Windows (from logs)

What actually happens on Windows Chrome (from 2026-06-22 log):

```
02:02:35.4  Handshake OK
02:02:35.4  → CMD_BT_RECONNECT_OFF  (45 4D 39 00 01 00 02 00 00)
02:02:35.5  → CMD_GLASS_LONG_PRESS  (45 4D 00 ...)   [1st, from connect()]
02:02:40.8  User clicks "Continue" button
02:02:40.8  → CMD_APP_SHOW_PAIR     (45 4D 67 ... 01)  [1st]
02:02:41.0  TX notifications enabled (post-pairing)
02:02:41.0  ← TX [0x01]            glasses re-request long-press (missed earlier)
02:02:41.0  → CMD_GLASS_LONG_PRESS  (2nd, from TX [0x01] handler)
02:02:41.0  → CMD_APP_SHOW_PAIR     (2nd)
02:02:41.0  ← 4F 42 67 ...         ACK for CMD_APP_SHOW_PAIR
02:02:41.1  ← 45 4D CE 00 ... 03 00  state transition (03→00)
02:02:41.1  ← 45 4D 2F 00 ...       mic stop notification
02:02:43.8  ← 4F 42 00 00 ... 00   longPressConfirmResponse (CMD_BIND RSP, SUCCESS)
02:02:43.8  → CMD_APP_SHOW_PAIR     (3rd, from longPressConfirmResponse handler)
02:02:43.8  → CMD_PAIR_SYNC_SUCCESS (45 4D 68 00 01 00 02 00 01) [host-initiated]
02:02:43.8    BT status polling started
02:02:43.9  ← 4F 42 67 ...         ACK for CMD_APP_SHOW_PAIR
02:02:44.2  ← 45 4D C7 00 01 00 02 00 01  unknown (possibly "Classic BT preparing")
02:02:44.2  ← 45 4D CE 00 ... 03 01  state transition (03→01, change from earlier)
02:02:44.2  ← 45 4D 2F 00 ...       mic stop notification
02:02:45.0  ← 4F 42 3A 00 ... 00 00  BT status: addressSaved=false connectStatus=false
                                      [repeats every ~1 s]
02:03:12.8  ← gattserverdisconnected  (glasses drop BLE, 29 s after longPressConfirmResponse)

NEVER seen: 45 4D 68 00 xx xx xx xx 01  (CMD_PAIR_STATE_SYNC from glasses)
```

### Observable state transitions

`45 4D CE 00` appears twice with different last bytes:
- Before longPressConfirmResponse: `... 03 00` (state=0)
- After second CMD_APP_SHOW_PAIR: `... 03 01` (state=1)

`45 4D C7 00 01 00 02 00 01` appears once after longPressConfirmResponse. Both
`C7` and `CE` are in the protocol's 0x81–0xCE "transparent instruction" range
and are not documented in the manual. They likely represent internal state
machine transitions in the glasses firmware.

---

## Commands Tried to Bypass the Classic BT Requirement

| Commit | Command | Result |
|---|---|---|
| `aeb3aad` | Respond to `0x0010` session-renewal | No effect on disconnect |
| `5276b61` | BT status polling after `userPressPairingCheck` | `45 4D 68 00 ... 01` never arrives on Windows |
| `9df52d8` | Respond to TX `[0x01]` with full pairing sequence | Correct but disconnect persists |
| `011def4` | Start polling from `longPressConfirmResponse` | Polls run, `connectStatus` always false |
| `b998794` | `CMD_BTCONNECT_BACK_ENABLE = 0` (disable BT auto-reconnect) | No effect |
| `05efa72` | Host-initiated `CMD_PAIR_SYNC_SUCCESS` (`45 4D 68 00 01 00 02 00 01`) | Pending test |

---

## Complete TX Message Reference

| Bytes | Name / Meaning | Action |
|---|---|---|
| `01` (1 byte) | Glasses request long-press flow | Send `CMD_GLASS_LONG_PRESS` + `CMD_APP_SHOW_PAIR` + `CMD_PAIR_SYNC_SUCCESS` |
| `4F 42 00 00 01 00 02 00 00` | `longPressConfirmResponse` — CMD_BIND RSP, status=SUCCESS | Send `CMD_APP_SHOW_PAIR` + `CMD_PAIR_SYNC_SUCCESS` + start BT polling |
| `4F 42 00 00 01 00 02 00 1C` | `longPressTimeoutResponse` — CMD_BIND RSP, status=ERR_TIME_OUT | Log error |
| `4F 42 67 00 01 00 02 00 00` | ACK for `CMD_APP_SHOW_PAIR` (`CMD_SET_PHONE_TYPE`) | `notifyAck()` |
| `4F 42 3A 00 03 00 06 00 ss aa cc` | BT status RSP — ss=status, aa=addressSaved, cc=connectStatus | Stop polling if `connectStatus=1` |
| `4F 42 1A 00 ...` | ACK for binary packet (translation frame) | `notifyAck()` |
| `4F 42 22 00 ...` | Brightness notification (spontaneous) | Ignored |
| `45 4D 68 00 xx xx xx xx 01` | `CMD_PAIR_STATE_SYNC` REQ, status=Success — only after Classic BT pairs | Send `CMD_CONNECT_SUCCEED` + `CMD_APP_SHOW_PAIR` + start BT polling |
| `45 4D 68 00 xx xx xx xx 00` | `CMD_PAIR_STATE_SYNC` REQ, status=Failed | Log error |
| `45 4D 10 00 ...` | Session renewal request | Reply with `CMD_BT_STATUS_POLL` |
| `45 4D 15 00 ...` | IMU data | Ignored |
| `45 4D C7 00 01 00 02 00 01` | Unknown — appears after `longPressConfirmResponse` | Logged only |
| `45 4D CE 00 02 00 04 00 03 00/01` | Unknown state transition — value changes 0→1 during pairing | Logged only |
| `45 4D 2F 00 01 00 02 00 00` | Mic stop notification | Logged only |

---

## Fundamental Platform Limitation

**Chrome Web Bluetooth is BLE-only.** It has no API for Classic Bluetooth
(BR/EDR, HFP, SPP). The Web Bluetooth specification explicitly excludes
Classic BT by design.

The glasses firmware requires `connectStatus=1` (Classic BT connected) to clear
the 30-second pairing timer. This condition cannot be satisfied from a Chrome
web page.

### Why native mobile apps work

An iOS or Android app has access to the device's native Bluetooth stack, which
handles BLE and Classic BT simultaneously. When the user accepts the pairing
popup, both transports connect at once and the glasses' timer clears.

---

## Options to Resolve

---

## All BLE-Only Approaches Exhausted (2026-06-22)

### Definitive proof: the "pi-pon" sound

When the glasses connect to an iPhone or Android phone they play a **"pi-pon"
connection sound** through their own speaker. This sound is transmitted via
the **HFP (Hands-Free Profile) Classic BT audio channel** — it is not possible
over BLE. The glasses play this sound at the exact moment Classic BT audio
connects, which is the same moment they send `45 4D 68 00 ... 01` and clear
the 30-second timer.

On Windows there is no "pi-pon". Classic BT audio never connects. The 30-second
timer always fires.

### Complete causal chain

```
Phone (iOS/Android)          Glasses firmware
─────────────────────────────────────────────────────────
CMD_APP_SHOW_PAIR ──────────► advertise for Classic BT
                              Classic BT audio (HFP) connects
                              ♪ pi-pon ♪  (via HFP audio channel)
                              connectStatus ← 1
                 ◄─────────── 45 4D 68 00 ... 01  (CMD_PAIR_STATE_SYNC REQ)
CMD_CONNECT_SUCCEED ─────────►
BT status poll ─────────────► connectStatus = 1
                              30-second timer cleared
```

Chrome Web Bluetooth cannot trigger any step on the Glasses column above.

### Host-initiated CMD_PAIR_SYNC_SUCCESS: did not work

Commit `05efa72` sent `45 4D 68 00 01 00 02 00 01` from the host.
Glasses ignored it — still disconnected at ~29 s. The firmware only accepts
`CMD_PAIR_STATE_SYNC` REQ as a notification it sends to the host, not the
reverse.

### Pre-pairing Classic BT via Windows Settings: causes cycling

Attempting to pair the glasses as a Classic BT device via Windows Settings
while a BLE session is active causes the glasses to alternate between the
"pairing page" and the "BLE disconnected page" in a loop.

**Why**: the glasses firmware expects BLE and Classic BT to pair simultaneously
(coordinated by a phone's Bluetooth stack in one combined popup). When they
arrive as separate events from two independent Windows subsystems, the firmware
state machine enters an inconsistent state and resets the BLE connection.

---

## Conclusion: Fundamental Platform Limitation

The web demo **cannot maintain a connection on Windows Chrome** without
architectural changes. This is not a protocol bug — it is a hard platform
constraint:

| Requirement | Web Bluetooth | Chrome on Windows |
|---|---|---|
| BLE GATT (commands, display) | ✓ Supported | ✓ Works |
| Classic BT audio (HFP, "pi-pon") | ✗ Not in spec | ✗ Impossible |

---

## Remaining Options

### Option 1: Electron app (recommended for Windows)

Replace the Chrome web demo with an Electron app. Node.js native Bluetooth
bindings (`@abandonware/noble` for BLE + WinRT or `node-bluetooth` for Classic
BT HFP) can replicate the full phone Bluetooth stack, triggering the "pi-pon"
and clearing the timer.

### Option 2: Firmware BLE-only mode (requires ThinkAR)

Ask ThinkAR to add a firmware flag or command that makes the glasses skip the
Classic BT audio requirement. With this, `connectStatus` would not be needed
and the 30-second timer would not apply. This is the cleanest fix but requires
firmware changes from the vendor.

### Option 3: Web demo only for iOS/Android

Accept the Windows limitation. Add a clear message in the web demo UI when
running on Windows (detected via `navigator.userAgent`) explaining that a
native app is required. The demo continues to work correctly on iOS and Android.
