# Measurement and energy model

What the numbers in the map mean, when they update, and how energy is kept so that a power cut,
reset or firmware update never silently loses or invents it.

## Sampling and windows

- All channels are measured **synchronously**; power is computed **per sample**
  (u·i), so non-linear loads (switch-mode supplies, dimmers, inverters) are measured truly.
- Samples aggregate into **measurement windows of whole mains cycles nearest 100 ms** — 5 cycles at
  50 Hz, 6 at 60 Hz (image 0x7C+; a fixed-length fallback without mains lock). Each finalized window
  is **published atomically** as a complete snapshot (all channels + totals); `WIN_SEQ` steps and
  DRDY pulses once per window, i.e. about **10 times a second**. Without mains lock the module keeps
  publishing on a slower fixed fallback tick, so WIN_SEQ and DRDY never stop
  ([DRDY](15_drdy_events.md)). *Images up to 0x7B: the window is 8 mains cycles (~160 ms at 50 Hz)
  and the publish tick runs every 200 ms.*
- Polling faster than the publish rate returns the same data. `WIN_NSAMP` (samples in the last
  window) and `WIN_PERIOD` (measured mains period in samples) describe the current lock.
- **Mains frequency** (`FREQ`, cHz) is derived from zero crossings over wall time (~2 s update) and
  is immune to sample drops; outside the measurable band it reads 0. `STATUS` bit 2 (mains_ok) is
  set while channel-0 voltage is above ~50 V — it is a mains-present flag, not a frequency check.
- **Lost samples are compensated:** if an internal operation (e.g. a firmware-update flash write)
  costs measurement frames, the gap is counted (`LOST_FRAMES`) and the energy integral is corrected for it —
  the counters have no gaps.
- **Low currents:** below roughly 20 mA the published current value is additionally smoothed over a
  few seconds for a stable reading (power and energy are unaffected); the smoothing resets on a load
  step, so transitions stay fast.

## Published quantities

Per voltage channel: RMS voltage. Per current channel: RMS current, active power (signed), reactive
power (signed), apparent power, power factor (signed). Per device: frequency, totals. Units and
scales — see [Conventions](README.md#conventions-used-throughout).

- **Signs:** P > 0 import; PF carries the sign of P; Q > 0 inductive (IEC 62053-23).
- **TOTALS k = 0** is the grand total over all channels; **k = 1..3** are per-phase totals (phase =
  channel on three-phase members; single-voltage builds sum everything into k = 1). TOTALS carry
  I/P/Q/S/PF only — their energy field positions read 0. Grand import is the positive part of the
  *aggregate* power per window — the net at a service entry — which differs from the sum of
  per-channel imports when phases disagree in direction; both views are available.
- THD and displacement quantities exist in the map but are not offered by current modules: they
  read 0 on I²C and the sentinel on the Modbus native profile.
- **Voltage phase angle** (image 0x7A+): each voltage channel publishes the fundamental's phase
  angle relative to voltage 0 (`ANGLE`, 0.01°, range (−180°, +180°]). Voltage 0 reads 0 by
  definition; `0x8000` means no angle (that input, or voltage 0, is below 50 V). On a forward
  sequence v1 ≈ −120°, v2 ≈ +120°.
- **Phase sequence** (`PHASE_SEQ` 0x0012, image 0x7A+), derived from the voltage angles:
  0 = not available (fewer than 3 voltage inputs, or one below 50 V) · 1 = forward L1-L2-L3 ·
  2 = reverse · 3 = abnormal (no 120° pattern — e.g. two inputs wired to the same phase). A useful
  installation check before trusting per-phase readings.

## Energy

### Per-window accumulators (realtime block)

`E_POS` / `E_NEG` in each current channel integrate signed energy in **mW·samples** since boot
(TOTALS do not carry them). The split is per window: each window's energy lands in the import or
export bucket by the sign of that window's power, so a mixed flow splits correctly. These are
diagnostic/derivation values — billing uses the cumulative counters.

### Cumulative counters

The **ENERGY block (0x0800)** holds life-time, device-persisted cumulative energy: active
import/export/net in whole **Wh** and in **mWh** (i64), reactive import/export in varh. Modules
with capability bit 15 (image 0x7A+) also publish the active counters at **µWh** resolution —
grand import/export/net at 0x08D0/0x08D8/0x08E0 and per channel at 0x0C80 + g·0x10 — in the same
snapshot as the mWh fields. Import/export mWh = floor(µWh / 1000); the **net** counter, in mWh and
in µWh alike, is import − export in that unit (with a negative net its mWh value can differ from
floor(net µWh / 1000) by 1). Rules:

- **Read energy from the module; never integrate P·dt on the host.** The module's sample-domain
  integral is gap-compensated and survives restarts; a host-side integral is neither.
- **Interval energy = delta of two cumulative readings** at the interval boundaries. The counters
  are monotonic within a generation; a missed boundary merges two intervals and loses nothing.
- **Persistence:** after a restart the counters **continue from the last saved value** — they never
  go below it; energy during the restart itself is not measured. The grand counters are saved every
  second *including the sub-Wh remainder*, and mirrored every publish window (~100 ms on image 0x7C+;
  200 ms up to 0x7B) to a backup
  area that survives a warm restart (reset, watchdog, firmware update). So: a warm restart resumes
  from about the last window; an ungraceful power loss loses at most about the **last second of
  energy** (P × 1 s — e.g. ~3 mWh at 10 kW). The per-channel counters are saved every second as
  whole mWh, so their step-back adds **less than 1 mWh** on top. After a restart a counter may
  therefore resume slightly *below* the last value read — take the delta as `max(0, new − last)`
  and re-anchor; do not discard the period.
- **µWh counters** (image 0x7A+): the same counters, the same rules — the restart step-back above
  applies unchanged (grand: about a second of energy; per-channel: plus the unsaved sub-mWh
  remainder). Take `max(0, Δ)` exactly as for the mWh fields.
- **µvarh counters** (image 0x7B+, same capability bit 15): reactive import/export at µvarh
  resolution, 0x08E8/0x08F0 — **grand only** (there is no per-channel reactive energy); varh =
  floor(µvarh / 1 000 000); same snapshot and restart rules as the µWh fields. The coherent
  fine-resolution read is **one transaction over 0x08D0–0x08F7 (40 B)**: active µWh
  import/export/net + reactive µvarh import/export.
- **Starting current** (`CFG_START_UA` 0x0084, image 0x7B+): while a channel's current is below
  this threshold, that channel adds **no active or reactive energy** to any counter — grand or
  per channel. Readings (I/P/Q/S/PF) are not affected. This stops the no-load creep where
  measurement noise of either sign would slowly feed both import and export; loads above the
  threshold are counted fully, starting within about a second of switch-on. Default 400 µA;
  0 disables it; 10 µA steps up to 2540 µA (out-of-range writes are rejected). A write applies at
  once; SAVE_CONFIG persists it.
- **Generations:** `LT_GEN` (0x0840) increments on every counter reset or adopt. **Never take a
  delta across a generation change** — re-anchor instead. Resets/adopts are keyed commands and are
  event-logged with before/after values ([Commands](14_commands.md)).
- **Per-channel counters** (GROUP_ENERGY, group g = channel g) are persisted every second too but
  are **informative**: they restart at 0 with a generation change or on new/foreign storage
  (`LTG_STATUS` = 3, transient — clears after the first per-channel write, ~1 s). The grand block is
  the cumulative counters.
- **Store health:** `LTA_STATUS` ≠ 0/1 (FAULT/CORRUPT) means energy is not persisting — the device
  keeps metering and recovers by itself when the store returns; `STATUS` bit 5 mirrors it. Records
  in the store are authenticated; foreign or edited records are rejected and counted
  (`NVM_REJ_CNT`). (Rollback-anchor detection — `SEC_ROLLBACK`, events 10/11 — is reserved, not
  implemented on current firmware.)

### Snapshot banks (capability bit 8)

The family contract also defines 8 independent snapshot banks (master-timed period snapshots with
per-channel energy). **Current firmware does not provide them** (capability bit 8 = 0; the snapshot
command answers "unknown command") — interval energy comes from cumulative-counter deltas as above.
Gate any snapshot usage on the capability bit.

## Time

Modules have **no battery clock** (capability bit 12 = 0). The master owns time:

- After **every** module reset the clock is invalid: `STATUS` bit 4 (time_valid) = 0,
  `RTC_EPOCH` reads 0. The master writes the epoch to `TIME_SET` (0x08C0; Modbus 0x8100–01) and
  issues command 0x06 — at discovery and whenever it sees time_valid = 0.
- Cumulative energy does **not** depend on the clock. Only timestamped history pauses until time is
  set.
- Before the first time-set nothing is bucketed into history. The first set **opens** a period
  flagged PARTIAL + POWERUP; a later step/correction marks the affected period with the CLOCK flag.
  Time never moves backwards inside the history.

## History records

The module keeps a short on-device history as a back-fill buffer for masters that were offline (the
deep archive belongs to the master/hub). Query interface: [register map](13_register_map.md#history--event-log-query-0x0c000x0c6f).

- **15-minute records** — the most recent, from RAM (lost on reset; `HIST_CAP` reads 0 here).
- **Hourly ring** — persisted; depth follows the storage size — read it from `HIST_CAP` (`0x0C08`).
- Records are written only while the clock is valid.

**Record layout (80 B, little-endian):**

| Off | Field | Type | Meaning |
|---|---|---|---|
| 0x00 | TS | u32 | period start, unix |
| 0x04 | FLAGS | u16 | see below |
| 0x06 | N | u16 | 15-min: 200-ms snapshots in the period · hourly: 15-min children (4 = complete) |
| 0x08 | V_DV | u16 | average voltage (channel 0), 0.1 V |
| 0x0A | I_MA | u32 | average current (sum of channels), mA |
| 0x0E / 0x12 / 0x16 | P_W / Q_VAR / S_VA | i32 each | period averages, W/var/VA |
| 0x1A | PF | i16 | Q10 |
| 0x1C–0x2B | MIN_P / MAX_P (i32, W) · MIN_I / MAX_I (u32, mA) | | period envelope |
| 0x2C / 0x32 | E_IMP_WH / E_EXP_WH | u48 | **absolute life-time Wh at period end** |
| 0x38 / 0x44 | PH_IMP_CWH / PH_EXP_CWH | u32×3 | per-phase period energy, 0.01 Wh |

Flags: bit0 PARTIAL (period not fully observed) · bit1 POWERUP (first after a boot) · bit2 CLOCK
(time set/stepped inside the period) · bit3 LTA_JUMP (life-time reset/adopt inside the period).
**Diff consecutive records only when the timestamp steps by exactly one period and neither record
has LTA_JUMP** — the absolute Wh fields make gap-filling after a master outage exact.

## Event log

`HIST_PERIOD` = 2 selects the **event log ring** through the same query window. Every
metering-relevant action leaves a 32-B record with before/after values:

| Off | Field | Type |
|---|---|---|
| 0x00 | TS | u32 (0 = clock was not set) |
| 0x04 | UPTIME_S | u32 |
| 0x08 | TYPE | u8 |
| 0x09 | FLAGS | u8 |
| 0x0A | BOOT | u16 (boot counter) |
| 0x0C / 0x10 | A_BEFORE / A_AFTER | u32 (meaning per type) |
| 0x14 / 0x1A | E_BEFORE / E_AFTER | u48, grand import Wh |

Event types: 1 counter reset · 2 counter adopt · 3 late counter load (store recovered from FAULT) ·
4 legacy-store migration · 5 calibration save (A = calibration checksum) · 6 firmware commit
(A = version before → after) · 7 history clear · 8 late config load · 9 foreign store records
rejected (A = count / first address) · 10/11 reserved (rollback detection — not implemented).

Capacity on current boards: 10 event records (`HIST_CAP` with `HIST_PERIOD` = 2).

## Boot readiness

Right after a boot a host should expect: the register interface answers, but measurements and
`WIN_SEQ` read 0 until the first publish (about one window after start — ~100 ms on image 0x7C+,
~200 ms up to 0x7B); the first ~3 windows add no energy
(settling); `FREQ` reads 0 for the first ~2 s; the certificate check runs ~3 s after boot
(`SEC_CERT_STATE` = 3 until then). Allow ~1 s before judging health.
