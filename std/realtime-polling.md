# 05 · Real-time polling

How to read live values from an STD module without missing or double-counting a window. Register-level
detail: [Measurement & energy model](reference/16_measurement_energy.md) and
[Data-ready output & events](reference/15_drdy_events.md).

## The measurement window

Samples aggregate into **windows of whole mains cycles nearest 100 ms** — 5 cycles at 50 Hz, 6 at 60 Hz.
Each finished window is published **atomically** as a complete snapshot (all channels and totals), and
`WIN_SEQ` (0x0016) steps by exactly 1. That is about **ten snapshots a second**. Polling faster returns
the same data.

Without mains the module keeps publishing on a slower fallback tick (500 ms) that refreshes status and
clock; on that tick `WIN_SEQ` steps but there is no DRDY pulse.

---

## Option 1 — poll `WIN_SEQ`

1. Read `WIN_SEQ`. If it is unchanged since the last read, there is no new snapshot.
2. If it changed, read the blocks you need — **one transaction per block, up to 128 bytes**; fields in one
   transaction come from one publication and cannot tear.

---

## Option 2 — react to the DRDY edge

DRDY is an open-drain output without a pull-up on the module; when you use DRDY, fit a pull-up on the master side — 10 kΩ to the master's 3.3 V logic supply is recommended. It idles high and gives a ~10 µs low pulse for each **new window**, after the registers are
updated — a read triggered by the falling edge always sees the fresh snapshot.

- On each serviced edge, check that `WIN_SEQ` stepped by exactly 1; a larger step means missed edges.
- **Alarm mode:** an event bit set in both `EVENT_FLAGS` and `EVENT_MASK` holds DRDY **low** until the
  flag is cleared or unmasked. Still low after ~100 µs means alarm — read `EVENT_FLAGS`.

---

## What a snapshot gives you

Per voltage channel: RMS voltage. Per current channel: RMS current, active power (signed; positive =
import), reactive power (positive = inductive), apparent power, power factor (signed). Per device:
frequency and totals (`k = 0` grand, `k = 1..3` per phase). Units and scales are in the
[register map](reference/13_register_map.md).

Below about 20 mA the published current is also smoothed over a few seconds for a stable reading; power
and energy are not affected.

---

## After a restart

Detect a restart with `UPTIME` (0x08A0, decreasing) or `BOOT_COUNT` (0x00CE, stepping). Then: re-arm the
event mask and thresholds, set the time again, and re-anchor your energy deltas. Right after boot
`WIN_SEQ` reads 0 for about one window, `FREQ` reads 0 for ~2 s, and the first ~3 windows add no energy;
allow ~1 s before judging health.

---

## In the client libraries

`readAll()` returns one coherent snapshot on both product lines; the libraries pair it with DRDY or with
`WIN_SEQ` for you. See [Examples](examples.md). On a module with snapshot banks and energy counters use Arduino [1.6.2 or later](https://github.com/rb-amp/rbamp-arduino/blob/main/docs/09_api_reference.md#energy-source-and-minimum-library-version) or ESP-IDF [1.7.3 or later](https://github.com/rb-amp/rbamp-esp-idf#std-line-minimum-version).
