# 05 · Troubleshooting

This chapter is a quick reference for common problems and their solutions. Each entry follows the structure:

- **Symptom** — what the user observes
- **Causes** — why it can happen
- **What to check / do** — concrete actions

![Flowchart for narrowing down a problem from the symptom.](images/troubleshoot-flowchart.png)

---

## I2C bus

### 1. NACK / no response from the module

**Symptom**: `Wire.endTransmission()` returns non-zero; `i2cdetect` does not show the address on Linux; `rb_read_u8()` returns `0xFF`.

**Causes and checks**:

- ❌ **Module power is missing or unstable** — measure VCC with a multimeter; it must be within 4.5…5.5 V.
- ❌ **No common ground** — the single most frequent mistake. Master GND and module GND **must** be tied together.
- ❌ **SDA and SCL swapped** — verify against the wiring diagram in [01_hardware.md](hardware-connection.md).
- ❌ **No pull-ups on the bus** — if the built-in pull-ups have been cut on a module and no external ones are installed, the bus is dead. With everything unpowered, measure SDA↔VCC: it must read 1.5–4.7 kΩ.
- ❌ **Too many pull-ups in parallel** — built-in pull-ups still active on every module, plus master pull-ups, dropping the equivalent below ~1 kΩ. The master cannot pull the bus to LOW. Cut the built-in pull-ups on all but one module.
- ❌ **Long cable without twisted pair or buffer** — past ~0.15 m, parallel wires couple capacitively and edges round off. Use UTP cat-5 (up to 1 m) or a PCA9515 buffer (up to 3 m). See [01_hardware.md → Bus length](hardware-connection.md).
- ❌ **Wrong slave address** — the module may have been readdressed. Run an I2C scan:

```cpp
void i2c_scan() {
  for (uint8_t addr = 0x08; addr < 0x78; addr++) {
    Wire.beginTransmission(addr);
    if (Wire.endTransmission() == 0) {
      Serial.printf("Found device at 0x%02X\n", addr);
    }
  }
}
```

- ❌ **I2C speed too high for the cable length**. Drop to 50 kHz: `Wire.setClock(50000);`.

---

### 2. Intermittent read/write timeouts

**Symptom**: transactions succeed sometimes, fail other times — especially in electrically noisy environments or near inductive loads.

**Causes**:

- ❌ Long cable without twisted pair
- ❌ Cable routed parallel to mains wiring → 50 Hz / 100 Hz coupling
- ❌ Noisy ground on the master side (e.g. a separate regulator without bulk capacitance)
- ❌ Stray capacitance between SDA and GND > 100 pF (e.g. a splice with long stubs)

**Solutions**:

- Use twisted pair (SDA + GND in one pair, SCL + GND in the other)
- Avoid routing I2C parallel to power wiring in the same conduit
- Reduce speed to 50 kHz / add 100 nF between VCC and GND of the module near the connector
- For long runs outside an enclosure, add ESD protection (TVS diodes on SDA and SCL)

> **In the client libraries** — the same task without register access:
>
> | Library | Call | What it does |
> |---|---|---|
> | Arduino | [`RbAmp::retryExhaustionCount()`](https://github.com/rb-amp/rbamp-arduino/blob/main/docs/09_api_reference.md#uint32_t-retryexhaustioncount-const-noexcept) / [`RbAmp::sanityRejectCount()`](https://github.com/rb-amp/rbamp-arduino/blob/main/docs/09_api_reference.md#uint32_t-sanityrejectcount-const-noexcept) | count exhausted bus retries and rejected readings |
> | ESP-IDF | [`rbamp_retry_exhaustion_count()`](https://github.com/rb-amp/rbamp-esp-idf/blob/main/docs/09_api_reference.md#rbamp_retry_exhaustion_count--rbamp_sanity_reject_count--rbamp_reset_counters) / [`rbamp_sanity_reject_count()`](https://github.com/rb-amp/rbamp-esp-idf/blob/main/docs/09_api_reference.md#rbamp_retry_exhaustion_count--rbamp_sanity_reject_count--rbamp_reset_counters) | count exhausted bus retries and rejected readings |
> | Python | [`dev.sanity_reject_count`](https://github.com/rb-amp/rbamp-python/blob/main/docs/09_api_reference.md#devsanity_reject_count-int) / [`MachineI2CBackend` counters](https://github.com/rb-amp/rbamp-python/blob/main/docs/09_api_reference.md#machinei2cbackend-diagnostic-counters-micropython-only) | count rejected readings; retry counters on the MicroPython backend |
> | ESPHome | — | no diagnostic counters exposed |

---

### 3. `DATA_VALID` (`0xCE`) never becomes 1

**Symptom**: master reads `0xCE` and always gets `0x00`. Other registers may also stay at zero.

**Causes**:

- ❌ The module has not finished booting (normal for the first ~250 ms after power-on)
- ❌ Firmware initialisation error — read `REG_ERROR` (`0x02`):
  - `0xFB` (`ERR_FLASH_PARAMS_BAD`) — flash parameter block is corrupted; the module runs on defaults. Recalibrate / reprovision.
  - `0xFA` (`ERR_LUT_BAD`) — the factory calibration table is broken. Not fatal — measurements continue; accuracy may be reduced. Have the supplier recalibrate.
- ❌ MCU clock failure (rare) — power-cycle the module

**Solutions**:

- Wait 300 ms after power-on before polling `0xCE`.
- If `DATA_VALID = 1` is not seen after > 1 s — power-cycle.
- If the problem persists after a power-cycle — check power (ripple, brown-out).

> **In the client libraries** — the same task without register access:
>
> | Library | Call | What it does |
> |---|---|---|
> | Arduino | [`RbAmp::waitReady(timeout_ms)`](https://github.com/rb-amp/rbamp-arduino/blob/main/docs/09_api_reference.md#bool-waitreadyuint32_t-timeout_ms--1000-noexcept) / [`RbAmp::readLastError()`](https://github.com/rb-amp/rbamp-arduino/blob/main/docs/09_api_reference.md#uint8_t-readlasterror-noexcept) | wait for the ready flag; read the device error code |
> | ESP-IDF | [`rbamp_wait_ready()`](https://github.com/rb-amp/rbamp-esp-idf/blob/main/docs/09_api_reference.md#rbamp_wait_ready) / [`rbamp_read_last_error()`](https://github.com/rb-amp/rbamp-esp-idf/blob/main/docs/09_api_reference.md#rbamp_read_last_error) | wait for the ready flag; read the device error code |
> | Python | [`dev.wait_ready(timeout_ms=1000)`](https://github.com/rb-amp/rbamp-python/blob/main/docs/09_api_reference.md#devwait_readytimeout_ms1000---none) / [`dev.read_last_error()`](https://github.com/rb-amp/rbamp-python/blob/main/docs/09_api_reference.md#devread_last_error---int) | wait for the ready flag (raises on timeout); read the device error code |
> | ESPHome | [`update()` ready check](https://github.com/rb-amp/rbamp-esphome/blob/main/docs/09_api_reference.md#3-data-flow-and-timing) / [`get_last_error_str()`](https://github.com/rb-amp/rbamp-esphome/blob/main/docs/09_api_reference.md#5-identity-surface-v13-new--public-c-methods) | reads are published once the module reports ready; the error code as text |

---

## Calibration and accuracy

### 4. `I_rms` close to 0 while a load is running

**Symptom**: the lamp is lit, the heater is warm, but `I_rms` shows 0.001 A or jitters between 0 and a few mA.

**Causes** (in decreasing order of frequency):

- ❌ **CT clamp not on the wire** — the conductor goes past the clamp instead of through it.
- ❌ **CT clamp on both L and N at the same time** — fluxes from line and neutral cancel, the sensor reads near zero. See [01_hardware.md → Current sensor](hardware-connection.md). The clamp must be on **one** conductor (line/L).
- ❌ **Clamp not fully closed** — a gap > 0.1 mm between the core halves dramatically increases magnetic reluctance, so the sensor output drops well below normal. Unclip and re-clip until it clicks shut. Dirt or oxide in the gap is a common culprit — wipe the mating surfaces.
- ❌ **Loose 3.5 mm jack** — remove and reinsert the plug.
- ❌ **`REG_CT_MODEL` not set** — the module does not know the sensor's sensitivity. See [02_initialization.md → Setting the sensor class and CT model](initialization.md#setting-the-sensor-class-and-ct-model). Register `0x05` must contain the correct model code.
- ❌ **Wrong model code written** — for example, an SCT-013-030 sensor with the SCT-013-010 code → current shown will be **~3× lower** than reality (and conversely, a too-large code reads high). Verify the sensor's case marking against the model code.
- ❌ **Current below the noise floor** — for very small loads (< ~10 mA on a 30 A clamp) the sensor output is close to the noise floor. The firmware applies quadrature noise subtraction: `I_corrected = √(I_raw² − NF²)`. If `I_raw ≤ NF`, the result is clamped to 0. This is intended behaviour.

---

### 5. `I_rms` off by a large factor

**Symptom**: real load is ~10 A (per reference meter), the module shows 3 A or 30 A.

**Causes**:

- ❌ **Wrong CT model code** — most common case. See #4.
- ❌ **Wrong CT model installed** — somebody fitted a 100 A clamp where the configuration calls for a 30 A clamp (or vice versa).
- ❌ **Clamp not fully closed** (see #4) — typically reads 5–20 % low.
- ❌ **Suspected factory calibration drift** — if all of the above are ruled out and the disagreement persists on a known reference load, contact your module supplier; recalibration is a factory operation.

---

### 6. Sign of `P_real` is wrong (consumption shows as export)

**Symptom**: the load is clearly consuming (warm heater, lit lamp), but `P_real < 0`.

**Causes and solutions**:

- ❌ **L and N swapped on the voltage terminals** — the most common cause. See [01_hardware.md → Voltage sensor](hardware-connection.md). De-energise the circuit first, then swap L and N on the PCB.
- ❌ **CT clamp installed backwards** — the arrow on the clamp must point **in the direction of current** toward the load (panel → load). If it points the other way, unclip and reverse the clamp.
- ❌ **Physical reversal is impractical** — on BASIC the period accumulator (`PERIOD_AVG_P_W`) clamps
  negative samples to zero, so with inverted polarity **the energy register stays 0 until the
  physical polarity is corrected** — negating in master code (`p_corrected = -p_raw`) only fixes
  your own RT integration, never the on-module period data. Fix the wiring (de-energised!).

---

### 7. Energy total disagrees with a reference meter by a few percent

**Symptom**: `E_Wh` over one hour differs from the utility meter by 2–10 %.

**Likely causes** (in decreasing order):

- ❌ **Clamp not fully closed** (gap) — typically −5 to −20 % low. The single most common cause. Unclip, clean the mating surfaces, clip again until fully shut.
- ❌ **Clamp installed on a "secondary" wire** — inside the breaker panel, on a short pigtail past a breaker. Stray fields from neighbouring breakers introduce errors. Move the clamp onto the main feed or onto an isolated section of conductor.
- ❌ **Close to strong reactive loads** — near transformers, asynchronous motors, UPSes. Magnetic fields from neighbouring inductors create a stray signal. Move the clamp ≥ 10 cm away from any ferromagnetic component.
- ❌ **Low PF** (≤ 0.5) — large reactive loads require precise phase compensation. Accuracy is specified for resistive loads (PF ≈ 1); it is not specified for strongly reactive loads. If accuracy on a specific PF profile is critical, contact your module supplier for guidance.
- ❌ **Distorted mains voltage** (micro-sags, high-order harmonics) — RMS and mean computation are more robust than peak detection, but accuracy is not specified for heavily distorted mains voltage; contact your supplier if this matters for your installation.
- ❌ **Damaged CT cable or loose jack** — open-circuit or cold-solder inside the plug. Check with a multimeter, with the jack unplugged from the module (the CT clamp is a step-down transformer; the secondary winding should read 50–200 Ω depending on model).

---

### 8. `avg_P_W` is much lower than expected

**Symptom**: a 1000 W heater is on, RT `P_real ≈ 1000 W`, but `PERIOD_AVG_P_W` reads ~200 W over a one-minute period.

**Causes**:

- ❌ The load is actually intermittent (cycling thermostat, pulsing element) — `avg_P_W` correctly reports the time average. Cross-check with `PERIOD_MAX_P_W` (`0xE0`) to see the peak inside any measurement window.
- ❌ If the load is steady and RT `P_real ≈ avg_P_W` does not hold:
  - On **BASIC tier**: check the RT sign of `P_real`. If `P` is occasionally negative (a solar inverter is feeding back, for example), negative samples are excluded from `PERIOD_AVG_P_W`, dragging the average down. See [04_period_metering.md → BASIC tier](period-metering.md).
  - On **STANDARD (STD)**: compute `total_avg = avg_P_consume + avg_P_export` (export taken as magnitude) to see the channel's true activity.

---

### 9. `PERIOD_VALID = 0` after a snapshot

**Symptom**: the master issues `CMD_LATCH_PERIOD`, reads `0x07`, gets 0. Snapshot is not fresh.

**Causes**:

- ❌ **Snapshots too close together** — no RT window (~100 ms on current firmware; 200 ms before v1.2) completed between two snapshots. The accumulator is empty, the snapshot is discarded.
- ❌ **Race with an RT commit** — rare: the master wrote the command at the exact instant the firmware was committing the live accumulator. Possible on very fast masters.

**Solution** — the retry pattern (see [04_period_metering.md → Basic cycle](period-metering.md)):

```cpp
if ((rb_read_u8(0x50, 0x07) & 0x01) == 0) {
  delay(250);   // let at least one RT window complete
  rb_write_u8(0x50, 0x01, 0x27);
  delay(50);
  // retry read of PERIOD_VALID
}
```

> **In the client libraries** — the same task without register access:
>
> | Library | Call | What it does |
> |---|---|---|
> | Arduino | [`RbAmp::isPeriodValid()`](https://github.com/rb-amp/rbamp-arduino/blob/main/docs/09_api_reference.md#bool-isperiodvalid-noexcept) / [`RbAmp::readPeriodSnapshot()`](https://github.com/rb-amp/rbamp-arduino/blob/main/docs/09_api_reference.md#bool-readperiodsnapshotrbampperiodsnapshot-out-uint16_t-settle_ms--50-bool-skip_latch--false-noexcept) | check the snapshot-ready bit; take and validate a period snapshot |
> | ESP-IDF | [`rbamp_is_period_valid()`](https://github.com/rb-amp/rbamp-esp-idf/blob/main/docs/09_api_reference.md#rbamp_is_period_valid) / [`rbamp_read_period_snapshot()`](https://github.com/rb-amp/rbamp-esp-idf/blob/main/docs/09_api_reference.md#rbamp_read_period_snapshot) | check the snapshot-ready bit; take and validate a period snapshot |
> | Python | [`dev.is_period_valid()`](https://github.com/rb-amp/rbamp-python/blob/main/docs/09_api_reference.md#devis_period_valid---bool) / [`dev.read_period_snapshot()`](https://github.com/rb-amp/rbamp-python/blob/main/docs/09_api_reference.md#devread_period_snapshotsettle_ms50-skip_latchfalse---rbampperiodsnapshot) | check the snapshot-ready bit; take and validate a period snapshot |
> | ESPHome | [Period snapshot not valid](https://github.com/rb-amp/rbamp-esphome/blob/main/docs/10_troubleshooting.md#period-snapshot-not-valid--energy-does-not-grow) | what the component logs when the valid flag is 0; the first snapshot after boot is discarded |

---

## Multi-module issues

### 10. Address collision

**Symptom**: several modules added to the bus — some "vanish" or return garbage. The same address answers with different values.

**Cause**: two or more modules share the same address → I2C arbitration / collision. Frequent right after unboxing, when every module is on the factory default `0x50`.

**Subtler form**: the address answers **stably**, but the UID read at `0x5C` (ch. 11 §4.8) matches no module you know. Two modules are answering together — the bus wire-ANDs their outputs into a stable but fictitious UID. If a hub sees an "unknown UID" at an address it provisioned, suspect a collision before suspecting a foreign module.

**Solution**:

1. Disconnect every module.
2. Reconnect **one at a time** — never readdress with two factory-default modules on the bus; readdress each to a unique address.
3. After readdressing all modules, reconnect them in parallel and verify each address by reading its UID.

See [02_initialization.md → Recommended workflow](initialization.md).

> **In the client libraries** — the same task without register access:
>
> | Library | Call | What it does |
> |---|---|---|
> | Arduino | [`RbAmpFleet::checkConflict()`](https://github.com/rb-amp/rbamp-arduino/blob/main/docs/09_api_reference.md#addressing-and-conflict-checking) / [`RbAmpFleet::scan()`](https://github.com/rb-amp/rbamp-arduino/blob/main/docs/09_api_reference.md#discovery--scan) | check an address for a second device; scan the bus for modules |
> | ESP-IDF | [`rbamp_fleet_check_conflict()`](https://github.com/rb-amp/rbamp-esp-idf/blob/main/docs/09_api_reference.md#addressing-and-conflict-checking) / [`rbamp_fleet_scan()`](https://github.com/rb-amp/rbamp-esp-idf/blob/main/docs/09_api_reference.md#discovery--rbamp_fleet_scan) | check an address for a second device; scan the bus for modules |
> | Python | [`fleet.check_conflict(addr)`](https://github.com/rb-amp/rbamp-python/blob/main/docs/09_api_reference.md#addressing-and-conflict-checking) / [`fleet.scan()`](https://github.com/rb-amp/rbamp-python/blob/main/docs/09_api_reference.md#discovery--scan) | check an address for a second device; scan the bus for modules |
> | ESPHome | [`rbamp:` list](https://github.com/rb-amp/rbamp-esphome/blob/main/docs/09_api_reference.md#1-component-block--rbamp) | one block per module, each with its own `id` and `address` |

---

### 11. Wired-OR DRDY fires several times in quick succession

**Symptom**: with [wired-OR DRDY](realtime-polling.md), the master EXTI fires every 1–5 ms a few times in a row.

**Cause**: the modules are not perfectly synchronised — each has its own crystal, and RT window periods differ slightly (99.9 ms vs 100.1 ms). Over the window span, the modules pull the line at slightly different instants.

**This is normal.** Solutions:

- Ignore "double" triggers: after the first trigger, read all modules; suppress further triggers within ~10 ms.
- Use broadcast snapshot (see [04_period_metering.md → Approach 2](period-metering.md)) for precise synchronisation — but only for period snapshots, not for RT.
- Move to DRDY-per-module — each module on its own GPIO.

---

### 12. I2C bus busy when DRDY fires

**Symptom**: DRDY fires while the master is already in a transaction with another module. The EXTI handler is delayed.

**This is normal.** The RT registers remain valid until the next commit (one window, ~100 ms, later). The master:

- Finishes the current transaction.
- Services the delayed EXTI and reads the next module.
- Latency: 1–10 ms typically — not critical.

---

### 13. Multi-module clock drift in long tests

**Symptom**: cumulative energy diverges between modules by fractions of a percent over a week.

**Cause**: each rbAmp has its own crystal with ±50 ppm drift (1–5 s/day).

**Solution** — built into the architecture: the master uses its **own** clock to multiply by dt. All modules share the same time reference, so the system-wide Wh total does not diverge.

To measure chip drift for QA, compare `PERIOD_LATCH_MS` (`0xEC`) with master dt:

```cpp
uint32_t chip_dt = rb_read_u32(0x50, 0xEC);
float ratio = (float)chip_dt / (float)master_dt;
printf("Chip drift: %.3f (1.000 = perfect)\n", ratio);
```

**Acceptance threshold for `PERIOD_LATCH_MS`: up to 30 % undercount is normal under load** (ratio as low as ~0.70). The module's internal diagnostic timebase reads short when the module is busy — by design. The 30 % envelope is a property of the module's timebase, not a fault. Apply this threshold uniformly across all SKUs and tiers.

The legacy 3 % threshold from earlier documentation is **wrong** for v1.3 and will mass-reject healthy modules; do not use it.

This drift only affects the chip-side diagnostic register; **energy accuracy is unaffected** because Wh is computed master-side as `avg_P × master_dt` (see [04_period_metering.md](period-metering.md)). If `master_dt` itself wanders (master MCU clock drifts), apply the master's own NTP/SNTP correction — `PERIOD_LATCH_MS` is not the right reference.

> **In the client libraries** — the same task without register access:
>
> | Library | Call | What it does |
> |---|---|---|
> | Arduino | [`RbAmp::readPeriodLatchMs()`](https://github.com/rb-amp/rbamp-arduino/blob/main/docs/09_api_reference.md#uint32_t-readperiodlatchms-noexcept) | read the module-side period length (diagnostic only) |
> | ESP-IDF | [`rbamp_read_period_latch_ms()`](https://github.com/rb-amp/rbamp-esp-idf/blob/main/docs/09_api_reference.md#rbamp_read_period_latch_ms) | read the module-side period length (diagnostic only) |
> | Python | [`dev.read_period_latch_ms()`](https://github.com/rb-amp/rbamp-python/blob/main/docs/09_api_reference.md#devread_period_latch_ms---int) | read the module-side period length (diagnostic only) |
> | ESPHome | — | not exposed |

---

## Power and flash

### 14. Power loss during a flash-write command → partial flash write

**Symptom**: after a power interruption during a write-to-flash command (`CMD_SAVE_USER_CONFIG`, `CMD_COMMIT_ADDR`), the module boots with `REG_ERROR = 0xFB` (`ERR_FLASH_PARAMS_BAD`) and `EVENT_FLAGS.bit3` set.

**What happened**: the firmware detected a CRC failure in the user-config flash block, loaded factory defaults, and rewrote them.

**Solution**:

- Factory defaults are restored — the module works, but its custom address / sensor configuration is gone.
- Re-apply the configuration: set `SENSOR_CLASS` + bind `CT_MODEL` per channel + issue `CMD_SAVE_USER_CONFIG` (and re-do address change if needed).
- To avoid this in the future: do not cut power for at least **1 second** after any flash-write command. The flash erase + write window itself is ~700 ms; allow margin.

---

### 15. Module is hot

**Symptom**: the enclosure feels noticeably warm.

**Normal**: at mains current up to nominal (SKU-dependent), light warming of the LV side (LDO regulator) up to ~40 °C is expected.

**Not normal**: hot enclosure (≥ 60 °C), burning smell, plastic discoloration.

**Causes**:

- ❌ Overvoltage on the LV side (VCC > 5.5 V) — check the supply.
- ❌ Internal damage on the mains side (short in the divider or CT circuit).
- ❌ External CT overloaded (current beyond the sensor range).

**Action**: disconnect power immediately, isolate the module, contact the supplier.

---

## Sensor wiring — critical errors

### 16. Sanity check — what "correctly wired" means

After installing or reinstalling a module, run a smoke test with a **known purely-resistive load** (incandescent bulb, electric kettle, heating element) — for example, a 100 W bulb on 230 V.

**Expected**:

- `U_rms ≈ 220..240` (depends on local mains)
- `I_rms ≈ 100 W / 230 V ≈ 0.43 A`
- `P_real ≈ +100 W` (positive, consumption)
- `PF ≈ 0.998..1.000` (resistive load)
- `Q ≈ 0` (no reactive content)

**If `P_real` is negative**: see #6 — L/N swapped or CT installed backwards.
**If `|I_rms − expected|` > 10 %**: see #5 — wrong CT model or clamp not closed.
**If `|P_real − U × I|` > 5 %** at PF = 1: phase compensation may be off — contact the supplier.

---

### 17. Pre-commissioning checklist

Before powering up a module in a production system:

- [ ] L wired to the L terminal of the PCB (not swapped with N)
- [ ] CT clamp on the **L conductor** (not on N, not on both at once)
- [ ] Arrow on the CT clamp **pointing toward the load**
- [ ] CT clamp fully closed (no gap, no slack — clip until it clicks)
- [ ] CT cable intact, jack fully seated in the socket
- [ ] Power VCC within 4.5…5.5 V, clean ground
- [ ] I2C pull-ups verified (see [01_hardware.md → I2C pull-ups](hardware-connection.md))
- [ ] For BASIC SCT-series modules: `REG_CT_MODEL` written and saved to match the installed sensor
- [ ] Smoke test with a resistive load completed — all five values (U, I, P, PF, Q) within the expected ranges

If any item is unchecked, expect strange readings. The most common causes of metering errors are sensor installation issues, not firmware or electronics faults.

---

## Error codes (`REG_ERROR` = `0x02`)

| Code | Name | Meaning | Action |
|:---:|---|---|---|
| `0x00` | `ERR_OK` | Normal — last write succeeded | — |
| `0xF9` | `ERR_CLONE` | Reserved — not raised on current firmware | If ever observed, contact the supplier. |
| `0xFA` | `ERR_LUT_BAD` | factory calibration table CRC failed; the module runs on linear approximation | Not fatal. Measurements continue; accuracy may be reduced. Have the supplier recalibrate. |
| `0xFB` | `ERR_FLASH_PARAMS_BAD` | Flash parameter block corrupted; defaults restored | Reconfigure address / `SENSOR_CLASS` / `CT_MODEL` and issue `CMD_SAVE_USER_CONFIG` (`0x32`). **Normal on a virgin module** (see chapter 11 §6.5) — do not abort first-time provisioning on `0xFB`. |
| `0xFC` | `ERR_NOT_READY` | First RT window has not yet completed after boot | Wait 300 ms. |
| `0xFD` | `ERR_SENSOR_OVERFLOW` | Input signal clipping (full scale exceeded) | Reduce current or use a larger CT. |
| `0xFE` | `ERR_PARAM` | Invalid command code or out-of-range value | Check command and value range. |
| `0xFF` | `ERR_UNHANDLED` | HardFault or unhandled condition | Power-cycle. If it recurs, contact the supplier. |

---

### Semantics — `REG_ERROR` is NOT sticky (v1.3)

`REG_ERROR` holds the outcome of the **last write transaction**. Every subsequent write — successful or not — overwrites it. There is no need to clear `REG_ERROR` between attempts; in fact, **reading it later than immediately after the failing write may show a fresher result from a subsequent operation**.

If you need a **durable** error signal that survives later writes, use `EVENT_FLAGS.bit3` (`ERROR_OCCURRED`) — it latches W1C and clears only when the master writes `1` to the bit. See chapter 11 §5.3 for the full error model.

Explicit clear (rarely needed): `CMD_CLEAR_ERROR` (opcode `0x31`) zeros `REG_ERROR`.

> **In the client libraries** — the same task without register access:
>
> | Library | Call | What it does |
> |---|---|---|
> | Arduino | [`RbAmp::readLastError()`](https://github.com/rb-amp/rbamp-arduino/blob/main/docs/09_api_reference.md#uint8_t-readlasterror-noexcept) / [`RbAmp::clearError()`](https://github.com/rb-amp/rbamp-arduino/blob/main/docs/09_api_reference.md#bool-clearerror-noexcept) | read the last-write error code; clear it |
> | ESP-IDF | [`rbamp_read_last_error()`](https://github.com/rb-amp/rbamp-esp-idf/blob/main/docs/09_api_reference.md#rbamp_read_last_error) / [`rbamp_clear_error()`](https://github.com/rb-amp/rbamp-esp-idf/blob/main/docs/09_api_reference.md#rbamp_clear_error) | read the last-write error code; clear it |
> | Python | [`dev.read_last_error()`](https://github.com/rb-amp/rbamp-python/blob/main/docs/09_api_reference.md#devread_last_error---int) / [`dev.clear_error()`](https://github.com/rb-amp/rbamp-python/blob/main/docs/09_api_reference.md#devclear_error---none) | read the last-write error code; clear it |
> | ESPHome | [`get_last_error_str()`](https://github.com/rb-amp/rbamp-esphome/blob/main/docs/09_api_reference.md#5-identity-surface-v13-new--public-c-methods) / [`write_clear_error()`](https://github.com/rb-amp/rbamp-esphome/blob/main/docs/09_api_reference.md#7-native-api-services-v13-new--recovery-automations) | the error code as text; clear the error |

---

## When to contact the supplier

Symptoms that warrant service:

- The module does not exit `DATA_VALID = 0` after three power-cycles
- `ERR_UNHANDLED` (`0xFF`) recurs
- Suspected calibration drift — disagreement > 5 % with a reference meter on a known resistive load persists after the diagnostic checklist in #17 is satisfied
- The enclosure is hot or smells of burnt material
- Disagreement with a reference meter > 5 % despite the checklist in #17 being satisfied
- A new module boots with `ERR_LUT_BAD` (`0xFA`) on first power-on

---

## Next

- [10_arduino_examples.md](arduino-examples.md) — examples with error handling
- For smart-home integrations (ESPHome / Home Assistant, Python), see the [client-libraries table in the README](README.md#client-libraries-high-level-api-per-platform)
