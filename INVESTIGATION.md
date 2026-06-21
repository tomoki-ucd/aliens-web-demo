# Investigation: ~30-Second Disconnect on Windows

## Symptom

On Windows with Chrome Web Bluetooth, the glasses (Ai Lens 071E) disconnect
after almost exactly **30.8 seconds** from the moment pairing completes
("Glasses ready!"). The connection is otherwise healthy: data frames are sent
and ACKs are received up until the moment of disconnect. The timing is
completely consistent across every test run.

---

## What Was Ruled Out

### 1. Long-press UI timeout
**Theory:** `CMD_GLASS_LONG_PRESS` starts a 30-second timer waiting for the user
to physically press the glasses.

**Result:** Disconnect still occurred at 30.8 s from "Glasses ready!", not from
when `CMD_GLASS_LONG_PRESS` was sent. Timer clearly starts from pairing
completion, not from the long-press trigger.

**Note:** A "Continue (long-press done)" UI button was still added and kept,
because the physical long-press IS required — the remaining pairing commands
must be sent after it, not with a blind 500 ms delay.

### 2. Missing CCCD / TX notifications not subscribed
**Theory:** `startNotifications()` fails on Windows before pairing
(`"GATT Error: invalid attribute length"`). Glasses might disconnect when they
see the CCCD is not set.

**Result:** Retrying `startNotifications()` after pairing succeeded. ACKs
(`4F 42 …`) then arrived for every translation frame. Disconnect still occurred
at 30.8 s.

### 3. `TX 45 4D 10 00` keepalive request (initially suspected root cause)
**Theory:** Glasses send `45 4D 10 00 02 00 04 00 19 00` at ~24 s, expect a
response within 7 s, and disconnect if none comes.

**Result:** After replying with `CMD_CONFIRMATION`, the `0x0010` message stopped
appearing in logs — but the disconnect still happened at 30.8 s. So `0x0010`
is a side-effect, not the primary cause. It is still handled with a
`CMD_CONFIRMATION` reply as a precaution.

### 4. TX [0x01] not responded to
**Theory:** Glasses send `TX [0x01]` right after "Glasses ready!" expecting
`CMD_GLASS_LONG_PRESS` back. Ignoring it for 30 s causes disconnect.

**Result:** Added handler: TX [0x01] → send `CMD_GLASS_LONG_PRESS` + full
pairing sequence. The GATT error "operation already in progress" fired because
TX [0x01] was arriving twice simultaneously. Fixed with a 200 ms dedup guard.
Disconnect still occurred at 30.8 s.

---

## Duplicate TX Event Bug (fixed)

Every BLE notification fired twice. Root causes investigated:
- `startNotifications()` called twice (pre- and post-pairing). Even the failing
  pre-pairing call partially registered an internal Chrome listener.
- Fixed by tracking `txNotificationsWorking` and adding the
  `characteristicvaluechanged` listener only once, inside whichever
  `startNotifications()` call succeeds.
- A global 200 ms dedup guard (`_lastTxHex` / `_lastTxMs`) was also added as a
  belt-and-suspenders measure, since some events still fire twice.

---

## Current Understanding of the Pairing State Machine

From analysis of the iOS SDK (`MetaGlasses.swift`) and the TX event stream
observed in logs, the correct event-driven pairing flow is:

```
[After handshake + TX subscription]

Glasses → TX [0x01]
Host    → CMD_GLASS_LONG_PRESS
                    (user physically long-presses touch bar)
Glasses → TX longPressConfirmResponse  (4F 42 00 00 01 00 02 00 00)
Host    → CMD_APP_SHOW_PAIR + CMD_DIRECT_PAIR + CMD_CONFIRMATION
Glasses → TX 45 4D 68 00              (true "ready" signal)
Host    → declares connected
```

### Why this was broken on Windows

TX notifications fail **before** pairing on Windows (CCCD write rejected). So:

1. Glasses send TX `[0x01]` → we can't receive it (not subscribed)
2. We send `CMD_GLASS_LONG_PRESS` proactively and wait for user button
3. User long-presses → Glasses send `longPressConfirmResponse` → we can't receive it
4. We send `CMD_APP_SHOW_PAIR + DIRECT_PAIR + CONFIRMATION` (triggered by button)
5. TX notifications succeed **post-pairing**
6. We declare "Glasses ready!" **prematurely** — the glasses' state machine
   hasn't completed its expected event-driven cycle
7. Glasses re-send TX `[0x01]` (because the previous one was missed)
8. 30.8 s after step 6, glasses disconnect

### Key TX messages observed

| Bytes | Meaning | Handled |
|---|---|---|
| `4F 42 …` (generic) | ACK for a write command | `notifyAck()` |
| `4F 42 00 00 01 00 02 00 00` | `longPressConfirmResponse` — pairing state machine trigger | Send `APP_SHOW_PAIR + DIRECT_PAIR + CONFIRMATION` |
| `4F 42 67 00 …` | ACK for `CMD_APP_SHOW_PAIR` | `notifyAck()` |
| `4F 42 0C 01 26 00 4C 00 …` | ACK for `CMD_DIRECT_PAIR`, includes device serial in payload (`MGG09B1224809005 4`) | `notifyAck()` |
| `4F 42 69 00 …` | ACK for `CMD_CONFIRMATION` | `notifyAck()` |
| `4F 42 1A 00 …` | ACK for binary packet (translation frame) | `notifyAck()` |
| `45 4D 68 00` | Glasses "truly ready" signal — expected after full pairing cycle | Log only (not yet observed in tests) |
| `45 4D 10 00 02 00 04 00 19 00` | Unknown; appears ~24 s in, 7 s before disconnect | Reply `CMD_CONFIRMATION` |
| `45 4D 15 00 …` | IMU data | Ignored |
| `45 4D CE 00 02 00 04 00 03 00/01` | Unknown; appears after pairing commands | Logged only |
| `45 4D C7 00 01 00 02 00 01` | Unknown; appears after `longPressConfirmResponse` | Logged only |
| `45 4D 2F 00 01 00 02 00 00` | Mic stop (`0x002F`) | Logged only |
| `01` (single byte) | Glasses requesting long-press flow | Send `CMD_GLASS_LONG_PRESS` |

---

## Root Cause (confirmed from iOS SDK source, 2026-06-22)

Analysis of `MetaGlasses.swift` and `GlassClient.swift` from the
`ailensble_20260610` snapshot revealed the following:

### iOS SDK never uses `CMD_DIRECT_PAIR` or `CMD_CONFIRMATION`

Both `directPairCommand` and `confirmationCommand` are defined as constants in
`MetaGlasses.swift` but are **never written to the peripheral** anywhere in
the codebase. The web demo was sending them incorrectly — this was confusing
the glasses' state machine and preventing it from sending the
`45 4D 68 00 xx ... 01` frame that the host must acknowledge.

### The correct full pairing sequence (from iOS SDK)

```
Glasses → TX [0x01]
Host    → CMD_GLASS_LONG_PRESS (45 4D 00 00 00 00 00 00)
                    (user physically long-presses touch bar)
Glasses → TX longPressConfirmResponse (4F 42 00 00 01 00 02 00 00)
Host    → CMD_APP_SHOW_PAIR (45 4D 67 00 01 00 02 00 01)  [only this — not DIRECT_PAIR/CONFIRMATION]
Glasses → TX userPressPairingCheck (45 4D 68 00 xx xx xx xx 01)  [9 bytes, last=0x01]
Host    → CMD_CONNECT_SUCCEED (4F 42 68 00 00 00 00 00)
Host    → CMD_APP_SHOW_PAIR (again, per beginClassicPairingConfirmation)
Host    → CMD_BT_STATUS_POLL (45 4D 3A 00 00 00 00 00) every 1 s
Glasses → TX 4F 42 3A 00 ... (BT status: result, bonded, classicConnected)
          When classicConnected=1 → iOS declares ready and stops polling
```

### Why 30 seconds?

The glasses firmware has a 30-second timer starting from when `CMD_GLASS_LONG_PRESS`
is sent. If the full sequence above (including `45 4D 68 00 ... 01` + polling)
does not complete within that window, the glasses drop the BLE connection.

The periodic `CMD_BT_STATUS_POLL` (`45 4D 3A 00`) every 1 second is the keepalive
that prevents the timeout. On iOS this also waits for Classic BT/HFP to connect;
on Windows (no Classic BT) the glasses may still respond, and we poll indefinitely.

### TX messages now understood

| Bytes | Meaning |
|---|---|
| `4F 42 00 00 01 00 02 00 00` | `longPressConfirmResponse` — auto-sent in direct-pair mode |
| `4F 42 00 00 01 00 02 00 1C` | `longPressTimeoutResponse` — user didn't long-press in time |
| `45 4D 68 00 xx xx xx xx 01` | `userPressPairingCheck` success — send `CMD_CONNECT_SUCCEED` + start polling |
| `45 4D 68 00 xx xx xx xx 00` | `userPressPairingCheck` failure — glasses reject, disconnect |
| `4F 42 3A 00 ...` (≥11 bytes) | BT status response to `CMD_BT_STATUS_POLL` |
| `4F 42 22 00 ...` | Brightness notification (spontaneous, no action needed) |
| `45 4D C7 00 ...` | Unknown; appears after pairing ACKs, benign |
| `45 4D CE 00 ...` | Unknown status transition indicator, benign |
| `45 4D 2F 00 01 00 02 00 00` | Mic stop notification, benign |

## Current Fix (status: implemented 2026-06-22, awaiting test)

Changes made to `index.html`:

1. **Removed `CMD_DIRECT_PAIR` and `CMD_CONFIRMATION`** from both the initial
   button-press sequence and the `longPressConfirmResponse` handler.

2. **Added `CMD_CONNECT_SUCCEED`** (`4F 42 68 00 00 00 00 00`): sent in response
   to `45 4D 68 00 xx xx xx xx 01`.

3. **Added `CMD_BT_STATUS_POLL`** (`45 4D 3A 00 00 00 00 00`) polling every 1 s,
   started after `CMD_CONNECT_SUCCEED` is sent. Keeps the glasses alive.

4. **Added `4F 42 3A 00 ...` handler**: logs BT status (result, bonded,
   classicConnected); stops polling when `result=0 && classicConnected=true`.

5. **Added `longPressTimeoutResponse`** handler (last byte `0x1C`): logs error.

6. **Changed `0x0010` keepalive reply** from `CMD_CONFIRMATION` to
   `CMD_BT_STATUS_POLL` (more consistent with iOS SDK poll pattern).

## Expected log if fix works

```
TX: 01 — sending CMD_GLASS_LONG_PRESS (awaiting longPressConfirmResponse)
TX longPressConfirmResponse: ... — sending CMD_APP_SHOW_PAIR
TX: 45 4D 68 00 ... 01 — pairing confirmed; sending connectSucceedResponse + starting BT polling
BT status polling started (1 s interval)
TX BT status: result=0x0 bonded=... classicConnected=...
[repeated every ~1 s, connection stays alive]
```

## Next Steps if Fix Still Fails

1. Check whether `45 4D 68 00 ... 01` appears. If not → the glasses are still
   not reaching the `userPressPairingCheck` state. Investigate whether the
   initial button-press `CMD_APP_SHOW_PAIR` is sufficient or if the whole
   initial pairing needs to be rethought.
2. If `4F 42 3A 00 ...` arrives: check whether `classicConnected` is ever `true`.
   If always `false` on Windows and the polling alone keeps glasses alive, the fix
   works regardless.
3. If the glasses still disconnect after ~30 s: the firmware timer might reset only
   on `classicConnected=true`. In that case, explore whether any payload in
   `CMD_BT_STATUS_POLL` or `CMD_CONNECT_SUCCEED` can signal "BLE-only mode".
