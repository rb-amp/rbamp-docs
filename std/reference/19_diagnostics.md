# Diagnostics

Everything a host (or a person with a bus adapter) can read to answer "is this module healthy, and
if not — why".

## STATUS (0x0006)

| Bit | Name | Meaning |
|---|---|---|
| 0 | CAL_OK | every channel calibrated. Image 0x7A+: set from a marker stored with the calibration when it is saved — a unit calibrated under older firmware keeps the 0x79 rule until its calibration is saved once more. Image 0x79: non-default gains and the stored calibration matches this firmware. Before 0x79 judge by `CAL_SRC` instead |
| 1 | SENSOR_OK | measurement path alive (measurement frames arriving) |
| 2 | MAINS_OK | mains present: channel-0 voltage above ~50 V (not a frequency check; FREQ reads 0 out of band) |
| 3 | LOCKED | reserved (production lock) |
| 4 | TIME_VALID | clock has been set since boot |
| 5 | NVM_FAULT | energy store FAULT/CORRUPT — energy not persisting |
| 6 | — | service mode indicator |

## Boot forensics

- **RESET_CAUSE (0x00CC)** — why *this* boot happened, raw reset flags: bit0 PIN · bit1 POWER ·
  bit2 SOFTWARE (reset command, update activation, address change) · bit3/4 watchdog ·
  bit5 LOW_POWER · bit6 OPTION. PIN accompanies most internal resets — judge by the other bits.
- **BOOT_COUNT (0x00CE)** — persisted boot counter (low 16 bits). A master that missed a restart
  sees it step; pair with EVENT_FLAGS bit 5 (RESET_OCCURRED).
- **UPTIME (0x08A0)** — seconds since boot; decreasing between polls = restart.

## Measurement-path counters

| Reg | Healthy | Meaning when not |
|---|---|---|
| FRAME_CNT (0x0020) | strictly increasing | frozen = acquisition stopped (SENSOR_OK will drop) |
| LOST_FRAMES (0x012E) | flat (small constant steps during firmware updates are normal and energy-compensated) | growing in normal operation = acquisition problem |
| RING_DROP (0x0106) | 0 | sample loss under load — report it |
| WIN_NSAMP / WIN_PERIOD (0x0110/0x0112) | stable; WIN_NSAMP ≈ 5 mains periods per window at 50 Hz / 6 at 60 Hz on image 0x7C+ (≈8 up to 0x7B) | unstable = mains lock problems |
| CRC_ERR (0x0024) / FRAME_OK (0x0104) | 0 / 1 | measurement-link errors |

## Storage and energy health

| Reg | Meaning |
|---|---|
| LTA_STATUS (0x0870) | energy store: 0 OK · 1 new (first life) · 2 FAULT · 3 CORRUPT. 2/3 → STATUS bit 5; metering continues, persistence resumes when the store recovers |
| LTA_SRC (0x0871) | where the counter came from at boot: 0 none · 1 NVM · 2 retention · 3 legacy (4 reserved) |
| LTG_STATUS (0x0873) | per-channel counters: 3 = restarted at 0 (new/foreign storage or layout change); transient — clears after the first write (~1 s) |
| CAL_SRC (0x0872) | 3 = stored calibration not for this firmware → defaults in use, needs service |
| EE_RD_ERR / EE_WR_ERR / EE_RECOVER (0x0874–0x0879) | store I/O error/recovery counters |
| LTA_AGE_S (0x087A) | seconds since the last successful energy write. A write happens only when the counters changed, so **large values at idle / very low load are normal** (saturates at 65535) — judge store health by LTA_STATUS / LTA_FAIL_CNT, not by age |
| NVM_REJ_CNT / NVM_REJ_ADDR (0x0898/0x089A) | records rejected as not written by this device |
| SEC_ROLLBACK (0x0BED) | reserved — rollback-anchor detection not implemented on current firmware (reads 0) |

## Transport diagnostics

- **LAST_EXCEPTION (0x0158 / holding 0x830A)** — the last Modbus exception with function code and
  address: the first thing to read when a SCADA integration "sometimes gets errors".
- **I2C_DIAG (0x013A)** — bus health counters (address matches / bus errors / overruns).
- **CMD_SEQ / CMD_STATE** — command flow sanity ([Commands](14_commands.md)).

## Status LEDs

| LED | Pattern | Meaning |
|---|---|---|
| green | short flash every ~2 s | metering runs |
| green | double flash | first ~8 s of every boot (until the image confirms) or an update session is open |
| green | off | module dead · LEDs disabled · no measurement frames for ~0.5 s |
| red | short blink | a request addressed to this module |
| red | steady | fault: no measurement frames, or energy store FAULT/CORRUPT |
| both | alternating ~5 Hz for 10 s | IDENTIFY command (0x0E) — locating the module |

`CFG_LED` = 0 turns the LEDs off (IDENTIFY still blinks).

## Health checklist for a supervising master

Poll cheap, alert on:

1. `STATUS`: cal_ok (0x79+), sensor_ok, mains_ok set; nvm_fault clear; time_valid set (else re-feed
   time).
2. `FRAME_CNT` advancing; `RING_DROP` = 0; `LOST_FRAMES` flat.
3. `BOOT_COUNT` / `UPTIME` — unexpected restarts; after one: re-arm EVENT_MASK/thresholds, re-set
   time, re-anchor energy deltas (`max(0, new − last)`).
4. `LT_GEN` unchanged (else re-anchor energy, check the event log for who reset it).
5. `LTA_STATUS` ≤ 1, `LTA_FAIL_CNT` flat, `CAL_SRC` ∈ {1, 2}.
6. Event log (HIST_PERIOD = 2) drained periodically — resets, adopts, calibration saves, firmware
   commits, rejected-record events, with before/after values.
