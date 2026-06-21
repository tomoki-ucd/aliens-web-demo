# Investigation: ~30-Second Disconnect on Windows

## Symptom

On Windows with Chrome Web Bluetooth, the glasses (Ai Lens 071E) disconnect
after almost exactly **30 seconds** from the moment the pairing sequence
completes ("Glasses ready!"). The connection is otherwise healthy: data frames
are sent and ACKs are received up until the moment of disconnect.

---

## What Was Ruled Out

### 1. Long-press UI timeout (initial hypothesis — wrong)
**Theory:** `CMD_GLASS_LONG_PRESS` starts a 30-second timer waiting for the user
to physically press the glasses. If nobody presses, the glasses disconnect.

**Fix attempted:** Added a "Continue (long-press done)" button so pairing commands
are sent only *after* the user has actually long-pressed.

**Result:** Disconnect still occurred at 30 seconds from "Glasses ready!" — not
from when `CMD_GLASS_LONG_PRESS` was sent. The timer starts from pairing
completion, not from the long-press trigger.

**Note:** The long-press fix was still kept because it is required for correct
pairing: the remaining commands (`CMD_APP_SHOW_PAIR`, `CMD_DIRECT_PAIR`,
`CMD_CONFIRMATION`) must be sent *after* the physical long-press, not
with a blind 500 ms delay.

### 2. Missing CCCD / TX notifications not subscribed
**Theory:** `startNotifications()` on the TX characteristic fails on Windows
with `"GATT Error: invalid attribute length"` before pairing. If the CCCD is
not written, the glasses cannot deliver ACK notifications and may disconnect
when they see no evidence the host is listening.

**Fix attempted:** Retry `startNotifications()` after the pairing commands,
at which point the BLE link is bonded/secured and the CCCD write may succeed.

**Result:** The retry succeeded ("TX notifications enabled (post-pairing)").
ACKs (`4F 42 …`) started arriving for every translation frame. But the
disconnect still occurred at ~30 seconds.

---

## Key Log Evidence

From a session where TX notifications were working and ACKs were received:

```
[20:49:00.9] TX notifications enabled (post-pairing)
[20:49:00.9] Glasses ready!
[20:49:00.9] TX: 01          ← initial TX events (pairing artefacts)
[20:49:00.9] TX: 01
[20:49:02.2] Auto-send started (every 2 s)
[20:49:02.4] TX ACK: 4f 42 1a 00 06 00 0c 00 00 67 00 00 00 00
[20:49:02.4] TX ACK: 4f 42 1a 00 06 00 0c 00 00 67 00 00 00 00  ← duplicate (bug, now fixed)
...
[20:49:24.6] TX: 45 4d 10 00 02 00 04 00 19 00   ← mystery message (×2)
[20:49:24.6] TX: 45 4d 10 00 02 00 04 00 19 00
...
[20:49:31.7] *** gattserverdisconnected fired ***  ← 30.8 s after ready
```

### Observations
- Disconnect is **initiated by the glasses**, not Chrome.
- It happens **exactly ~30 seconds** after "Glasses ready!" regardless of how
  much data is being sent or whether TX notifications are active.
- An **unknown TX message** (`45 4D 10 00 02 00 04 00 19 00`) appears at
  ~23.7 s after ready — **7 seconds before** the disconnect.
- The message appeared twice due to a duplicate event listener bug (now fixed).

### Duplicate TX event listener bug (now fixed)
`startNotifications()` was attempted twice (once pre-pairing, once post-pairing).
Although the pre-pairing attempt threw an exception, Chrome/Windows was still
partially registering an internal listener, causing every notification to fire
twice. Fixed by tracking `txNotificationsWorking` and only adding the
`characteristicvaluechanged` event listener once, inside the `startNotifications()`
call that actually succeeds.

---

## Current Hypothesis

### `TX 45 4D 10 00` is a session-renewal request

Parsed against the standard EM command frame format:

| Field    | Bytes     | Value                     |
|----------|-----------|---------------------------|
| Magic    | `45 4D`   | "EM"                      |
| cmdType  | `10 00`   | `0x0010` (16)             |
| len      | `02 00`   | 2                         |
| len2     | `04 00`   | 4 (= len × 2 ✓)          |
| payload  | `19 00`   | `[0x19, 0x00]` (25, 0)   |

The glasses send this at ~24 s into a session. If the host does not respond
within ~7 seconds, the glasses disconnect. This is consistent with a
**keep-alive / session-renewal protocol** where the glasses periodically verify
the host is still responsive.

On iOS, the iOS SDK routes this message to `delegate?.onTxValueReceived(glasses:self, data:value)`.
The iOS *app* (not the SDK) presumably handles it and sends a response. We do
not have the iOS app source code, so the correct response command is unknown.

---

## Fix Attempted (status: untested)

In `onTX`, detect cmdType `0x0010` and reply immediately with `CMD_CONFIRMATION`
(`45 4D 69 00 01 00 02 00 01`):

```js
if (data[0] === 0x45 && data[1] === 0x4D && data[2] === 0x10 && data[3] === 0x00) {
  log(`TX 0x0010 session-renewal: ${hex} — replying with CMD_CONFIRMATION`);
  if (writeChar) {
    writeChar.writeValueWithoutResponse(CMD_CONFIRMATION)
      .catch(e => log(`0x0010 reply error: ${e.message}`, 'err'));
  }
  return;
}
```

`CMD_CONFIRMATION` was chosen as a first guess because it is the last pairing
command and semantically represents "I am ready". If this is wrong, the
disconnect will still occur at 30 s. Other candidates to try:

| Response to try | Rationale |
|---|---|
| Echo exact bytes back (`45 4D 10 00 02 00 04 00 19 00`) | Generic ping-pong acknowledgment |
| `CMD_DIRECT_PAIR` + `CMD_CONFIRMATION` | Full session re-confirmation |
| `45 4D 11 00 00 00 00 00` | cmdType 0x0011 as a possible paired "response" opcode |
| `45 4D 10 00 02 00 04 00 19 00` mirrored | Symmetric keepalive |

---

## Next Steps if Current Fix Fails

1. **Share the new log** after the `CMD_CONFIRMATION` response attempt — check
   whether the `0x0010` message is acknowledged or if the disconnect timing
   changes.
2. **Try echoing the exact bytes** back to the glasses as a generic keepalive
   response.
3. **Inspect the iOS app** source code (not just the SDK) to find how
   `onTxValueReceived` handles cmdType `0x0010`.
4. **Ask the glasses firmware team** what `0x0010` expects as a response and
   what `payload[0] = 0x19` (25) represents (possibly a countdown or session ID).
