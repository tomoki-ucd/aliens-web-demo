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

## Current Fix (status: untested as of last push)

After enabling TX notifications post-pairing, the `onTX` handler now correctly
implements the glasses' state machine:

1. **TX `[0x01]`** (post-ready) → send only `CMD_GLASS_LONG_PRESS`
2. **`longPressConfirmResponse`** (`4F 42 00 00 01 00 02 00 00`) → send
   `CMD_APP_SHOW_PAIR + CMD_DIRECT_PAIR + CMD_CONFIRMATION`
3. **`45 4D 68 00`** → log "glasses signaled ready!" (this should now appear
   and confirm the full pairing cycle completed)

The hypothesis is that responding properly to `longPressConfirmResponse` will
cause the glasses to send `45 4D 68 00` and stop the 30-second timer.

In direct-pair mode (`needPair = false` on iOS), the glasses are expected to
send `longPressConfirmResponse` **automatically** without requiring a second
physical long-press from the user (confirmed by user: iOS never needs a second
long-press).

---

## Next Steps if Current Fix Fails

1. Check whether `TX: 45 4D 68 00 — glasses signaled ready!` appears in the log.
   If yes but still disconnects → the 30-second timer is triggered by something
   else entirely.
2. Check timing of disconnect: if it shifts from 30.8 s after "Glasses ready!"
   to 30.8 s after something else → that new anchor point is the real trigger.
3. **Ask the firmware/iOS app team** what the `45 4D CE 00` and `45 4D C7 00`
   messages mean, and whether the host is expected to respond to them.
4. Investigate whether the `MGG09B…` serial number in the `CMD_DIRECT_PAIR` ACK
   payload needs to be stored and echoed back in subsequent commands.
