# 06 · Energy and periods

How to get energy from an STD module: cumulative counters, hourly history, the event log, and the eight snapshot banks. Full
rules: [Measurement & energy model](reference/16_measurement_energy.md).

## Cumulative counters

The module keeps life-time energy counters that survive power-off: active import, export and net in Wh,
mWh and µWh; reactive import and export in varh and µvarh. Read them in one transaction over
`0x0800–0x0843` (and `0x08D0–0x08F7` for the fine-resolution set).

Rules:

- **Read energy from the module; never integrate P·dt on the host.** The module's integral is
  gap-compensated and survives restarts.
- **Interval energy = the difference of two readings** at the interval boundaries. A missed boundary
  merges two intervals and loses nothing.
- After a restart the counters continue from the last saved value; up to about the last second of energy
  may be missing. Take the difference as `max(0, new − last)` and re-anchor — do not discard the period.
- `LT_GEN` (0x0840) steps on every counter reset or adopt. If it changed, do not take the difference:
  re-anchor and check the event log.
- **Starting current** (`CFG_START_UA`, default 400 µA): below it a channel adds no energy to any counter,
  which stops no-load creep. Readings are not affected.

---

## Hourly history

The module keeps the last 7 hourly records (energy and an electrical summary for each hour) in non-volatile memory (`HIST_CAP`, 0x0C08, gives the number of records). The records survive a restart; a longer archive is kept by the master.

- Records are written only after the master has set the time, and the master sets it again after every module restart: there is no battery-backed clock (see [Getting started](getting-started.md#5-set-the-time-optional-but-recommended)).
- A record closes when the module's clock passes the hour boundary. A part-hour record (the time was set mid-hour, or the module powered up mid-hour) carries a flag.
- Of the 15-minute period only the last finished summary is available (electrical values, no energy). It is kept in RAM and is lost on restart.

Query interface: [register map](reference/13_register_map.md#history--event-log-query-0x0c000x0c6f).

---

## Event log

The **event log** holds counter resets and adopts, calibration saves, firmware updates and rejected
records, with before/after values; drain it periodically.

---

## Snapshot banks

When `CAPABILITIES` bit 8 = 1, the module has **eight independent snapshot banks**: every
master freezes the period it needs, and several consumers do not disturb one another. Use them when
several modules must cut the same period at the same instant; for one module the counter differences
above are simpler.

1. **Arm.** Write `PERIOD_ARM` (0x0063): bit *b* = 1 makes bank *b* accumulate. A newly armed bank
   starts empty.
2. **Snapshot.** Write `LATCH_BANK` (0x0062) = *b*, then command `0x03` (`LATCH_PERIOD`). The cut is taken at
   the next window boundary; the bank restarts from zero. On a Modbus bus the same writes may be sent on
   broadcast (unit 0) for one cut across modules.
3. **Wait.** After `CMD_SEQ` steps, wait until `WIN_SEQ` has advanced by **2** (about 200 ms).
4. **Read.** The bank is at `0x2000 + b·0x800` (Modbus input `0x7000 + b·0x100`): P / Q / S averaged over
   the period, signed energy, window count, peaks, and per-channel energy. `PD_VALID` bit *b* says the
   bank holds a snapshot; it stays set and is not a freshness flag.

Banks are RAM: a reset clears them. A bank snapshotted during the first 3 windows after boot holds zeros with
`WIN_CNT` = 0. Layout and units: [register map](reference/13_register_map.md#snapshot-banks-0x2000).

---

## In the client libraries

> **In the client libraries** — the same task without register access:
>
> | Library | Call | What it does |
> |---|---|---|
> | Arduino | [`stdArmBanks`, `stdLatch`, `stdReadLatch`, `stdLatchWh`](https://github.com/rb-amp/rbamp-arduino/blob/main/docs/09_api_reference.md#period-snapshot-banks) | arm banks, take a snapshot, read it and convert it to Wh; available when the module reports banks (bit 8) |
> | ESP-IDF | [`rbamp_std_arm_banks`, `rbamp_std_latch`, `rbamp_std_read_latch`, `rbamp_std_latch_wh`](https://github.com/rb-amp/rbamp-esp-idf#std-snapshot-banks) | the same in the ESP-IDF component; bank 0 belongs to the library, use banks 1 and up (example `std_meter`) |
> | ESPHome | [`period_banks:`, `rbamp.latch_bank`](https://github.com/rb-amp/rbamp-esphome/blob/main/docs/09_api_reference.md#period_banks) | declare the banks to latch in YAML, on an `interval:` and/or by the action (from v1.9.1) |
> | Python | `std_arm_banks`, `std_latch`, `std_read_latch`, `std_latch_wh` | the same calls in the Python client (from v1.6.1); a bank's valid flag is sticky, read a bank at least 300 ms after latching it |

See [Examples](examples.md).
