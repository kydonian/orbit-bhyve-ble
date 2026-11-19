# Authors note

This is all 100% vibe-coded and unreviewed. The AI found your integrations and got it working on my hardware, I told it to record anything that might be helpful to the original author's. Presented for what it is, just in case it's useful.

# orbit-bhyve-ble, for the B-hyve WT25G2 sprinkler controller

**The integration's documentation is at [ljmerza/orbit-bhyve-ble](https://github.com/ljmerza/orbit-bhyve-ble)**,
where it's developed. This fork publishes two changes I run at home. I don't maintain it for
anyone else, and the changes haven't been proposed upstream.

**The code changes are totally vibe-coded.** An AI assistant wrote them, and I haven't reviewed them
line by line. What I can vouch for is the testing on my own controller ([Testing at
home](#testing-at-home)) and the evidence behind each claim ([Evidence](#evidence)).

## Where it comes from

- [ljmerza/orbit-bhyve-ble](https://github.com/ljmerza/orbit-bhyve-ble) is the integration: local
  Bluetooth control of Orbit B-hyve timers in Home Assistant.
- [pete-jorgensen/orbit-bhyve-ble](https://github.com/pete-jorgensen/orbit-bhyve-ble) adds the
  WT25G2 sprinkler controller, on its `wt25g2` branch. That work isn't in ljmerza's repo yet
  ([E1](#e1-wt25g2-support-isnt-upstream)).
- This fork is pete-jorgensen's `wt25g2` (at `d1e861e`) plus the two changes below, on branch
  `wt25g2`.

## Why it exists

My WT25G2 worked with pete-jorgensen's branch, but it was slow and unreliable:
- **Slow:** starts took 5.5–28.9 s and stops 7.9–24.3 s ([E2](#e2-command-times)).
- **Failing:** 5 of 23 starts failed outright, each at the moment the integration logged a "CTR
  desync" ([E3](#e3-the-desync-check-fired-on-good-frames-and-broke-commands)).
- **Stale:** Home Assistant kept showing a valve's old state for up to 10 s after a command had
  worked ([E4](#e4-the-valve-state-lagged-by-up-to-10-s)).

Two changes fix that:

1. **The controller stays connected.**
   - The integration used to connect, do its encrypted handshake and disconnect for every command
     ([E5](#e5-a-connection-per-command)).
   - Its check for a garbled connection ("CTR desync") also fired on status messages that were
     fine, and its disconnect broke the command being sent
     ([E3](#e3-the-desync-check-fired-on-good-frames-and-broke-commands)).
   - Now the WT25G2, which is mains-powered, keeps its connection, and the check reconnects only
     when a reply really is garbled. The battery timers still connect per command
     ([E6](#e6-only-the-wt25g2-keeps-its-connection)).
2. **A valve shows its new state as soon as the command returns**, for open, close and
   `stop_all` ([E4](#e4-the-valve-state-lagged-by-up-to-10-s)).

Before and after, measured the same way (the script's own log of each Home Assistant service call):

| | Before | After |
|---|---|---|
| A start takes | 5.5–28.9 s; 5 of 23 failed | 2.1–2.2 s; 0 of 8 failed |
| A stop takes | 7.9–24.3 s | 3.5–4.2 s |
| The valve state when the call returns | Old in 5 of 7 calls | Current in 4 of 4 |
| "CTR desync" warnings in Home Assistant's log | 108 in 3 days | None since |

## Testing at home

The setup:
- One WT25G2-0001 (firmware 0096).
- An ESPHome Bluetooth proxy (ESP32-S3, ESPHome 2026.10.2) and Home Assistant 2026.11.1.
- The water off throughout. Times are Pacific.

What I ran:
1. **Before**, on pete-jorgensen's code: my blowout script's runs, 2026-11-17 21:33–21:59 and
   2026-11-18 10:54–12:19 ([E2](#e2-command-times)).
2. **12:24, change 1 deployed.**
3. **12:30–12:37, timing test:** 7 cycles of start, wait 10 s, close, fresh read, across zones
   1–3. The last two came after 90 s idle, longer than the old 60 s idle disconnect.
   - The integration's debug log shows no reconnects.
   - The kind of status frame that used to trip the desync check was logged as "ignored", and
     decrypted to a timestamp ([E3](#e3-the-desync-check-fired-on-good-frames-and-broke-commands)).
   - Home Assistant showed the old valve state in 5 of 7 cycles, which led to change 2.
4. **12:50, chirp test on zone 1:** start, then close after a hold of 0–5 s. The controller
   reported the zone on for 3.0–7.4 s, about 2 s more than the hold.
5. **12:58–12:59, five-second blasts** on zones 9, 8, 7 and 6, back to back. The controller had
   each on for 4.5–6.7 s, with no errors.
6. **13:04, change 2 deployed.**
7. **13:11, timing test:** 4 cycles, one after 90 s idle. The valve state was current after every
   start and close.
8. **13:10–14:32, three blowout runs** with my script: 10 s and 60 s blasts, and 10.5-minute gaps
   between blasts. All 8 starts and 5 stops worked first time ([E2](#e2-command-times)).
9. **Since 13:11:** the controller has stayed connected, with no read timing out, through 14:31
   (Home Assistant's history of its Connected and Consecutive timeouts sensors).

Seen and not investigated: after each Home Assistant restart, the first read timed out (12:25:32
and 13:05:13). The controller showed as disconnected until the next read: 16 s the first time,
6 minutes the second.

## State of play

- **Running:** on my controller since 2026-11-18.
- **Not proven:**
  - a connection held for days;
  - the cost of keeping one of the Bluetooth proxy's connection slots permanently;
  - any other controller.

  The hose timers get change 1's stricter desync check, untested here.
- **Known gap:** one other failure path still disconnects in the middle of a read, the
  decrypt-failure branch at
  [`connection.py:434`](https://github.com/kydonian/orbit-bhyve-ble/blob/36db236cb90d98628af90ebaa01a4b17c4565bd3/custom_components/orbit_bhyve/connection.py#L434).
  That's
  [ljmerza/orbit-bhyve-ble#72](https://github.com/ljmerza/orbit-bhyve-ble/issues/72).
- **Upstream:** not proposed. Change 2 alone, rebuilt on ljmerza's `main`, is on branch
  `fix/valve-confirmed-state`.
- **Issues:** off on this fork.

## Using it

Copy `custom_components/orbit_bhyve/` from branch `wt25g2` into Home Assistant's
`/config/custom_components/` and restart Home Assistant. There are no releases, and it isn't set up
for HACS.

## Evidence

`d1e861e` below is pete-jorgensen's code before these changes, and `36db236` is this fork's code
with them.

### E1. WT25G2 support isn't upstream

pete-jorgensen opened it as
[ljmerza/orbit-bhyve-ble#73](https://github.com/ljmerza/orbit-bhyve-ble/pull/73) and closed it on
2026-11-07:

> Closing this for now so you have one PR from me to review at a time; sorry for the noise. WT25
> support will come back after #70 and the disconnect fix, rebased so it carries only the WT25
> changes.

### E2. Command times

These are the Home Assistant service-call durations my blowout script logged, in seconds; ✗
marks a call that failed with HTTP 500.

Before (pete-jorgensen's code):
- **Starts, 2026-11-17:** 8.7, 13.7, 17.7.
- **Starts, 2026-11-18:** 22.7, 11.7✗, 3.8✗, 11.0, 2.9✗, 10.6, 12.8✗, 22.7, 28.9, 12.8✗, 10.9,
  8.4, 5.5, 13.6, 5.5, 14.5, 10.8, 5.5, 17.5, 16.8.
- **Stops (`stop_all`), 2026-11-17:** 22.8, 7.9, 10.4, 24.3.

After (this fork), 2026-11-18:
- **Starts:** 2.1, 2.1, 2.2, 2.1, 2.1, 2.2, 2.1, 2.1.
- **Stops:** 4.1, 3.5, 4.2, 4.1, 4.1.

### E3. The desync check fired on good frames and broke commands

Each of the 5 failed starts came within a second of a desync warning, at 11:08:26, 11:08:32,
11:22:03, 11:28:11 and 11:52:47. The one at 11:28:11 failed in the integration's own write, 30 ms
after the warning:

```
# Home Assistant core log, 2026-11-18: a start that failed (trimmed)
11:28:11.133 WARNING [custom_components.orbit_bhyve.connection] 44:67:55:0A:5F:D0: CTR desync detected (bad header 3a34362d) — scheduling disconnect to self-heal
11:28:11.163 ERROR [aiohttp.server] Error handling request from 192.168.1.106
  File "/config/custom_components/orbit_bhyve/devices/protobuf.py", line 515, in start_watering
  File "/config/custom_components/orbit_bhyve/connection.py", line 658, in send
  File "/config/custom_components/orbit_bhyve/connection.py", line 613, in _write_frame
bleak.exc.BleakError: Peripheral 44:67:55:1A:5F:D0 changed connection status while waiting for BluetoothGATTWriteResponse
```

That disconnect comes from
[`connection.py:380–390` at `d1e861e`](https://github.com/pete-jorgensen/orbit-bhyve-ble/blob/d1e861e1726e94a7c2b33ba3d81dca0bd6246a5e/custom_components/orbit_bhyve/connection.py#L380-L390).
When the first frame of an exchange lacks the expected header, it schedules a disconnect,
without waiting for the command being written.

The frames it flagged were fine. Home Assistant's log has 108 such warnings from 2026-11-15
12:58 to 2026-11-18 11:52, and 87 of their "bad headers" are plain text:

| "Bad header" | As text | Times |
|---|---|---|
| `32303236` | `2026` | 21 |
| `3a35342d`, `3a31352d`, … | `:54-`, `:15-`, … | 66 |
| `0800221d`, `10880e50`, … | not text | 21 |

These are pieces of the timestamps in the controller's status messages, which
[pete-jorgensen's notes](https://github.com/pete-jorgensen/orbit-bhyve-ble/blob/d1e861e1726e94a7c2b33ba3d81dca0bd6246a5e/custom_components/orbit_bhyve/devices/wt25g2.py#L19-L23)
say the later frames carry. A frame that decrypted wrongly would be random bytes, not text, so
the decryption was in step. After change 1, the debug log at 12:30:27 shows the same kind of
frame decrypting to `2026-11-18T12:30` and being ignored. There have been no desync warnings since the change was
deployed at 12:24.

### E4. The valve state lagged by up to 10 s

After a confirmed start or close, the valve called `async_request_refresh()`
([`valve.py:97` and `:101` at `d1e861e`](https://github.com/pete-jorgensen/orbit-bhyve-ble/blob/d1e861e1726e94a7c2b33ba3d81dca0bd6246a5e/custom_components/orbit_bhyve/valve.py#L97-L101)).
Home Assistant debounces that with a 10 s cooldown
([`REQUEST_REFRESH_DEFAULT_COOLDOWN = 10`](https://github.com/home-assistant/core/blob/dev/homeassistant/helpers/update_coordinator.py)):
within the cooldown the call returns at once and the entities update when it ends.

Measured:
- **Before change 2 (12:30 test):** right after the call, Home Assistant showed the valve closed
  after 5 of 7 starts, and watering after 5 of 7 closes, though the integration's debug log had
  the controller confirm each within about 2 s.
- **After change 2 (13:11 test):** the state was current after 4 of 4 starts and 4 of 4 closes.

### E5. A connection per command

At `d1e861e`, every operation in
[`devices/protobuf.py`](https://github.com/pete-jorgensen/orbit-bhyve-ble/blob/d1e861e1726e94a7c2b33ba3d81dca0bd6246a5e/custom_components/orbit_bhyve/devices/protobuf.py#L541-L545)
ends with `finally: await self.connection.disconnect()`. That covers start at line 545, stop at
582, the status read at 500, and others. An idle timer closes anything left open after 60 s.

### E6. Only the WT25G2 keeps its connection

- pete-jorgensen's notes call the WT25G2 Orbit's mains-powered controller
  ([`wt25g2.py:4`](https://github.com/pete-jorgensen/orbit-bhyve-ble/blob/d1e861e1726e94a7c2b33ba3d81dca0bd6246a5e/custom_components/orbit_bhyve/devices/wt25g2.py#L4)).
- In this fork, keeping the connection is off by default
  ([`protobuf.py:283`](https://github.com/kydonian/orbit-bhyve-ble/blob/36db236cb90d98628af90ebaa01a4b17c4565bd3/custom_components/orbit_bhyve/devices/protobuf.py#L283)),
  and only the WT25G2 turns it on
  ([`wt25g2.py:46`](https://github.com/kydonian/orbit-bhyve-ble/blob/36db236cb90d98628af90ebaa01a4b17c4565bd3/custom_components/orbit_bhyve/devices/wt25g2.py#L46)).
