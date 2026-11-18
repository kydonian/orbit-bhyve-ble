# orbit-bhyve-ble, for the B-hyve WT25G2 sprinkler controller

**The integration's documentation is at [ljmerza/orbit-bhyve-ble](https://github.com/ljmerza/orbit-bhyve-ble)**,
where it's developed. This fork publishes two changes I run at home. I don't maintain it for
anyone else, and the changes haven't been proposed upstream.

## Where it comes from

- [ljmerza/orbit-bhyve-ble](https://github.com/ljmerza/orbit-bhyve-ble) is the integration: local
  Bluetooth control of Orbit B-hyve timers in Home Assistant.
- [pete-jorgensen/orbit-bhyve-ble](https://github.com/pete-jorgensen/orbit-bhyve-ble) adds the
  WT25G2 sprinkler controller, on its `wt25g2` branch. That work isn't in ljmerza's repo yet.
- This fork is pete-jorgensen's `wt25g2` (at `d1e861e`) plus the two changes below, on branch
  `wt25g2`.

## Why it exists

My WT25G2 worked with pete-jorgensen's branch, but it was slow and unreliable:
- each start or stop took 5–23 s;
- the first start after a quiet spell usually failed;
- Home Assistant showed a valve's new state up to 10 s after the command had worked.

Two changes fix that:

1. **The controller stays connected.** The integration used to connect, do its encrypted handshake
   and disconnect for every command. A check meant to catch a broken connection also fired on
   harmless status messages, and cut commands off mid-send. Now the WT25G2, which is
   mains-powered, keeps its connection. The check only reconnects when a reply really is garbled.
   The battery timers still connect per command.
2. **A valve shows its new state as soon as the command returns**, for open, close and
   `stop_all`. Before, the state updated whenever Home Assistant next refreshed.

Measured on my controller, through an ESPHome Bluetooth proxy, with the water off:

| | Before | After |
|---|---|---|
| A start takes | 5–23 s; the first after a quiet spell failed | 1.6–1.7 s |
| A stop takes | 6–23 s | 4.1–4.3 s |
| The valve state when the call returns | Up to 10 s stale | Current |
| Reconnects | One per command | None |

## State of play

- **Running:** one WT25G2-0001 (firmware 0096), Home Assistant 2026.11.1, since 2026-11-18.
- **Not proven:**
  - a connection held for days;
  - the cost of keeping one of the Bluetooth proxy's connection slots permanently;
  - any other controller.
  
  The hose timers get the stricter connection check from change 1, untested here.
- **Known gap:** one other failure path can still drop the connection mid-read
  ([ljmerza/orbit-bhyve-ble#72](https://github.com/ljmerza/orbit-bhyve-ble/issues/72)).
- **Upstream:** not proposed. Change 2 alone, rebuilt on ljmerza's `main`, is on branch
  `fix/valve-confirmed-state` for anyone who wants it.
- **Issues:** off on this fork.

## Using it

Copy `custom_components/orbit_bhyve/` from branch `wt25g2` into Home Assistant's
`/config/custom_components/` and restart Home Assistant. There are no releases, and it isn't set up
for HACS.
