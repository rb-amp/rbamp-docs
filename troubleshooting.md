# Troubleshooting

This chapter is a quick reference for common problems and their solutions. Each entry follows the structure:

- **Symptom** — what the user observes
- **Causes** — why it can happen
- **What to check / do** — concrete actions

## I²C bus

### 1. NACK / no response from the module

**Symptom**: `Wire.endTransmission()` returns non-zero; `i2cdetect` does not show the address on Linux; `rb_read_u8()` returns `0xFF`.

**Causes and checks**:

- ❌ **Module power is missing or unstable** — measure VCC with a multimeter; it must be 5 V ±5%.
- ❌ **No common ground** — the single most frequent mistake. Master GND and module GND **must** be tied together.
- ❌ **SDA and SCL swapped** — verify against the wiring diagram in [Hardware connection](hardware-connection.md).
- ❌ **No pull-ups on the bus** — if the built-in pull-ups have been cut on a module and no external ones are installed, the bus is dead. With everything unpowered, measure SDA↔VCC: it must read 1.5–4.7 kΩ.
- ❌ **Too many pull-ups in parallel** — built-in pull-ups still active on every module, plus master pull-ups, dropping the equivalent below ~1 kΩ. The master cannot pull the bus to LOW. Cut the built-in pull-ups on all but one module.
- ❌ **Long cable without twisted pair or buffer** — past ~0.3 m, parallel wires couple capacitively and edges round off. Use UTP cat-5 (up to 1 m) or a PCA9515 buffer (up to 3 m). See [Hardware connection — Bus length](hardware-connection.md#bus-length).
- ❌ **Wrong slave address** — the module may have been readdressed. Run an I²C scan:

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

- ❌ **I²C speed too high for the cable length**. Drop to 50 kHz: `Wire.setClock(50000);`.

### 2. Burst read returns the same byte four times (auto-increment trap)

**Symptom**: a 4-byte burst read of `0x86` (intended to read U_rms as float32) returns four identical bytes — or all zeros, or all `0xFF`. Voltage readings look plausible-shaped but completely wrong (e.g. `127.0` always, regardless of mains).

**Cause**: rbAmp does **not** auto-increment its internal register pointer. Code that issues a single `Wire.requestFrom(addr, 4)` after `Wire.write(0x86)` reads the byte at `0x86` four times, not bytes `0x86`, `0x87`, `0x88`, `0x89`. This is the most reflexive coding mistake for engineers used to PZEM / ATM90E32 / 24Cxx EEPROMs where multi-byte burst reads are standard.

**Solution**: read each byte with its own transaction and explicit register address. Use the `rb_read_float_le()` helper in [Initialization](initialization.md#bring-up-on-arduino-wireh) — it loops `rb_read_u8(addr, reg + i)` for `i = 0..3`. Same for `u32` reads (see [Period metering — Clock-drift diagnostics](period-metering.md#clock-drift-diagnostics)).

### 3. Intermittent read/write timeouts

**Symptom**: transactions succeed sometimes, fail other times — especially in electrically noisy environments or near inductive loads.

**Causes**:

- ❌ Long cable without twisted pair
- ❌ Cable routed parallel to mains wiring → 50 Hz / 100 Hz coupling
- ❌ Noisy ground on the master side (e.g. a separate regulator without bulk capacitance)
- ❌ Stray capacitance between SDA and GND > 100 pF (e.g. a splice with long stubs)

**Solutions**:

- Use twisted pair (SDA + GND in one pair, SCL + GND in the other)
- Avoid routing I²C parallel to power wiring in the same conduit
- Reduce speed to 50 kHz / add 100 nF between VCC and GND of the module near the connector
- For long runs outside an enclosure, add ESD protection (TVS diodes on SDA and SCL)

### 4. `DATA_VALID` (`0xCE`) never becomes 1

**Symptom**: master reads `0xCE` and always gets `0x00`. Other registers may also stay at zero.

**Causes**:

- ❌ The module has not finished booting (normal for the first ~250 ms after power-on)
- ❌ Firmware initialisation error — read `REG_ERROR` (`0x02`):
  - `0xFB` (`ERR_FLASH_PARAMS_BAD`) — flash parameter block is corrupted; the module runs on defaults. Recalibrate / reprovision.
  - `0xFA` (`ERR_LUT_BAD`) — ADC LUT calibration is broken. Not fatal — measurements still work, but accuracy is slightly degraded.
- ❌ MCU clock failure (rare) — power-cycle the module

**Solutions**:

- Wait 300 ms after power-on before polling `0xCE`.
- If `DATA_VALID = 1` is not seen after > 1 s — power-cycle.
- If the problem persists after a power-cycle — check power (ripple, brown-out).

## Calibration and accuracy

### 5. `I_rms` close to 0 while a load is running

**Symptom**: the lamp is lit, the heater is warm, but `I_rms` shows 0.001 A or jitters between 0 and a few mA.

**Causes** (in decreasing order of frequency):

- ❌ **CT clamp not on the wire** — the conductor goes past the clamp instead of through it.
- ❌ **CT clamp on both L and N at the same time** — fluxes from line and neutral cancel, the sensor reads near zero. See [Hardware connection — Current sensor](hardware-connection.md#current-sensor--ct-and-sct-013-for-skus-with-external-ct). The clamp must be on **one** conductor (line/L).
- ❌ **Clamp not fully closed** — a gap > 0.1 mm between the core halves dramatically increases magnetic reluctance, so the sensor output drops well below normal. Unclip and re-clip until it clicks shut. Dirt or oxide in the gap is a common culprit — wipe the mating surfaces.
- ❌ **Loose 3.5 mm jack** — remove and reinsert the plug.
- ❌ **`REG_CT_MODEL` not set** — the module does not know the sensor's sensitivity. See [Initialization — Setting the external sensor model](initialization.md#setting-the-external-sensor-model). Register `0x05` must contain the correct model code.
- ❌ **Wrong model code written** — for example, an SCT-013-030 sensor with the SCT-013-100 code → current shown will be **3.3× lower** than reality. Verify the sensor's case marking against the model code.
- ❌ **Current below the noise floor** — for very small loads (< ~10 mA on a 30 A clamp) the sensor output is close to ADC noise. The firmware applies quadrature noise subtraction: `I_corrected = √(I_raw² − NF²)`. If `I_raw ≤ NF`, the result is clamped to 0. This is intended behaviour.

### 6. `I_rms` off by a large factor

**Symptom**: real load is ~10 A (per reference meter), the module shows 3 A or 30 A.

**Causes**:

- ❌ **Wrong CT model code** — most common case. See #5.
- ❌ **Wrong CT model installed** — somebody fitted a 100 A clamp where the configuration calls for a 30 A clamp (or vice versa).
- ❌ **Clamp not fully closed** (see #5) — typically reads 5–20 % low.
- ❌ **Factory calibration disturbed** (`I0_GAIN` at `0xF4..0xF7` has an abnormal value). Check:

```cpp
float gain = rb_read_float_le(0x50, 0xF4);
Serial.printf("I0_GAIN = %.4f\n", gain);
```

Normal range: 0.9…1.1. If far outside, the module needs service / factory recalibration.

### 7. Sign of `P_real` is wrong (consumption shows as export)

**Symptom**: the load is clearly consuming (warm heater, lit lamp), but `P_real < 0`.

**Causes and solutions**:

- ❌ **L and N swapped on the voltage terminals** — the most common cause. See [Hardware connection — Voltage sensor](hardware-connection.md#voltage-sensor-ui-variants). Swap L and N on the PCB.
- ❌ **CT clamp installed backwards** — the arrow on the clamp must point **in the direction of current** toward the load (panel → load). If it points the other way, unclip and reverse the clamp.
- ❌ **Physical reversal is impractical** — compensate in master code:

```cpp
float p_corrected = -p_raw;
```

  but the physical fix is preferable — otherwise every piece of code must remember the inversion.

### 8. Energy total disagrees with a reference meter by a few percent

**Symptom**: `E_Wh` over one hour differs from the utility meter by 2–10 %.

**Likely causes** (in decreasing order):

- ❌ **Clamp not fully closed** (gap) — typically −5 to −20 % low. The single most common cause. Unclip, clean the mating surfaces, clip again until fully shut.
- ❌ **Clamp installed on a "secondary" wire** — inside the breaker panel, on a short pigtail past a breaker. Stray fields from neighbouring breakers introduce errors. Move the clamp onto the main feed or onto an isolated section of conductor.
- ❌ **Close to strong reactive loads** — near transformers, asynchronous motors, UPSes. Magnetic fields from neighbouring inductors create a stray signal. Move the clamp ≥ 10 cm away from any ferromagnetic component.
- ❌ **Low PF** (≤ 0.5) — large reactive loads require precise phase compensation. Factory calibration assumes cos(φ) = 1.0 (resistive loads) with a ~5-sample compensating shift. For specific cases tune `REG_V03_PHASE_SAMPLES` (`0x06`).
- ❌ **Distorted mains voltage** (micro-sags, high-order harmonics) — RMS and mean computation are more robust than peak detection, but distortions can still introduce 1–3 % error.
- ❌ **Damaged CT cable or loose jack** — open-circuit or cold-solder inside the plug. Check with a multimeter (the CT clamp is a step-down transformer; the secondary winding should read 50–200 Ω depending on model).

### 9. `avg_P_W` is much lower than expected

**Symptom**: a 1000 W heater is on, RT `P_real ≈ 1000 W`, but `PERIOD_AVG_P_W` reads ~200 W over a one-minute period.

**Causes**:

- ❌ The load is actually intermittent (cycling thermostat, pulsing element) — `avg_P_W` correctly reports the time average. Cross-check with `PERIOD_MAX_P_W` (`0xE0`) to see the peak inside any 200 ms window.
- ❌ If the load is steady and RT `P_real ≈ avg_P_W` does not hold:
  - On **BASIC tier**: check the RT sign of `P_real`. If `P` is occasionally negative (a solar inverter is feeding back, for example), negative samples are excluded from `PERIOD_AVG_P_W`, dragging the average down. See [Period metering — BASIC tier](period-metering.md#what-is-not-available-on-basic).
  - On **STANDARD / PRO**: compute `total_avg = avg_P_consume + avg_P_export` (export taken as magnitude) to see the channel's true activity.

### 10. `PERIOD_VALID = 0` after a latch

**Symptom**: the master issues `CMD_LATCH_PERIOD`, reads `0x07`, gets 0. Snapshot is not fresh.

**Causes**:

- ❌ **Latches too close together** — no RT window (200 ms) completed between two latches. The accumulator is empty, the snapshot is discarded.
- ❌ **Race with an RT commit** — rare: the master wrote the command at the exact instant the firmware was committing the live accumulator. Possible on very fast masters.

**Solution** — the retry pattern (see [Period metering — Basic cycle](period-metering.md#basic-cycle-one-module-60-second-period)):

```cpp
if ((rb_read_u8(0x50, 0x07) & 0x01) == 0) {
  delay(250);   // let at least one RT window complete
  rb_write_u8(0x50, 0x01, 0x27);
  delay(50);
  // retry read of PERIOD_VALID
}
```

## Multi-module issues

### 11. Address collision

**Symptom**: several modules added to the bus — some "vanish" or return garbage. The same address answers with different values.

**Cause**: two or more modules share the same address → I²C arbitration / collision. Frequent right after unboxing, when every module is on the factory default `0x50`.

**Solution**:

1. Disconnect every module.
2. Reconnect one at a time; readdress each to a unique address.
3. After readdressing all modules, reconnect them in parallel.

See [Initialization — Recommended workflow](initialization.md#recommended-multi-module-deployment-workflow).

### 12. Wired-OR DRDY fires several times in quick succession

**Symptom**: with [wired-OR DRDY](realtime-polling.md#strategy-3--wired-or-drdy-compact), the master EXTI fires every 1–5 ms a few times in a row.

**Cause**: the modules are not perfectly synchronised — each has its own crystal, and RT window periods differ slightly (199.8 ms vs 200.1 ms). Over the ~200 ms span, the modules pull the line at slightly different instants.

**This is normal.** Solutions:

- Ignore "double" triggers: after the first trigger, read all modules; suppress further triggers within ~10 ms.
- Use broadcast latch (see [Period metering — Approach 2](period-metering.md#approach-2--broadcast-latch-via-general-call-precise)) for precise synchronisation — but only for period snapshots, not for RT.
- Move to DRDY-per-module — each module on its own GPIO.

### 13. I²C bus busy when DRDY fires

**Symptom**: DRDY fires while the master is already in a transaction with another module. The EXTI handler is delayed.

**This is normal.** The RT registers remain valid until the next commit (200 ms later). The master:

- Finishes the current transaction.
- Services the delayed EXTI and reads the next module.
- Latency: 1–10 ms typically — not critical.

### 14. Multi-module clock drift in long tests

**Symptom**: cumulative energy diverges between modules by fractions of a percent over a week.

**Cause**: each rbAmp has its own crystal with ±50 ppm drift (1–5 s/day).

**Solution** — built into the architecture: the master uses its **own** clock to multiply by dt. All modules share the same time reference, so the system-wide Wh total does not diverge.

To measure chip drift for QA, compare `PERIOD_LATCH_MS` (`0xEC`) with master dt. Note rbAmp has no pointer auto-increment, so a `u32` read requires 4 separate transactions (see helper in [Period metering — Clock-drift diagnostics](period-metering.md#clock-drift-diagnostics)):

```cpp
uint32_t chip_dt = rb_read_u32_le(0x50, 0xEC);
float ratio = (float)chip_dt / (float)master_dt;
printf("Chip drift: %.3f (1.000 = perfect)\n", ratio);
```

Acceptable range: 0.97–1.03 (3 % drift = 4 min/day — large, but harmless for energy). Beyond this range, the module needs service.

## Power and flash

### 15. Power loss during `CMD_SAVE_GAINS` → partial flash write

**Symptom**: after a power interruption during SAVE_GAINS, the module boots with `REG_ERROR = 0xFB` (`ERR_FLASH_PARAMS_BAD`).

**What happened**: the firmware detected a CRC failure in the flash parameter block, loaded defaults, and rewrote them.

**Solution**:

- Factory defaults are restored — the module works, but with baseline accuracy.
- If you had configured a custom address or CT model, write them again and issue `SAVE_GAINS`.
- To avoid this in the future: do not cut power immediately after `SAVE_GAINS` — wait at least 1 second.

### 16. Module is hot

**Symptom**: the enclosure feels noticeably warm.

**Normal**: at mains current up to nominal (SKU-dependent), light warming of the LV side (LDO regulator) up to ~40 °C is expected.

**Not normal**: hot enclosure (≥ 60 °C), burning smell, plastic discoloration.

**Causes**:

- ❌ Overvoltage on the LV side (VCC > 5.5 V) — check the supply.
- ❌ Internal damage on the mains side (short in the divider or CT circuit).
- ❌ External CT overloaded (current beyond the sensor range).

**Action**: disconnect power immediately, isolate the module, contact the supplier.

## Sensor wiring — critical errors

### 17. Sanity check — what "correctly wired" means

After installing or reinstalling a module, run a smoke test with a **known purely-resistive load** (incandescent bulb, electric kettle, heating element) — for example, a 100 W bulb on 230 V.

**Expected**:

- `U_rms ≈ 220..240` (depends on local mains)
- `I_rms ≈ 100 W / 230 V ≈ 0.43 A`
- `P_real ≈ +100 W` (positive, consumption)
- `PF ≈ 0.998..1.000` (resistive load)
- `Q ≈ 0` (no reactive content)

**If `P_real` is negative**: see #7 — L/N swapped or CT installed backwards.
**If `|I_rms − expected|` > 10 %**: see #6 — wrong CT model or clamp not closed.
**If `|P_real − U × I|` > 5 %** at PF = 1: phase compensation may be off — contact the supplier.

### 18. Pre-commissioning checklist

Before powering up a module in a production system:

- [ ] L wired to the L terminal of the PCB (not swapped with N)
- [ ] CT clamp on the **L conductor** (not on N, not on both at once)
- [ ] Arrow on the CT clamp **pointing toward the load**
- [ ] CT clamp fully closed (no gap, no slack — clip until it clicks)
- [ ] CT cable intact, jack fully seated in the socket
- [ ] Power VCC = 5 V ±5 %, clean ground
- [ ] I²C pull-ups verified (see [Hardware connection — I²C pull-ups](hardware-connection.md#ic-pull-ups--installed-on-the-board))
- [ ] For modules with a plug-in CT: `REG_CT_MODEL` written and saved to match the installed sensor
- [ ] Smoke test with a resistive load completed — all five values (U, I, P, PF, Q) within the expected ranges

If any item is unchecked, expect strange readings. The most common causes of metering errors are sensor installation issues, not firmware or electronics faults.

## Error codes (`REG_ERROR` = `0x02`)

| Code | Name | Meaning | Action |
|:---:|---|---|---|
| `0x00` | `ERR_OK` | Normal | — |
| `0xFA` | `ERR_LUT_BAD` | ADC LUT calibration CRC failed; the module runs on linear approximation | Not fatal. ADC accuracy may be off by 1–5 %. Have the supplier recalibrate. |
| `0xFB` | `ERR_FLASH_PARAMS_BAD` | Flash parameter block corrupted; defaults restored | Reconfigure address / CT model and `SAVE_GAINS`. |
| `0xFC` | `ERR_NOT_READY` | First RT window has not yet completed after boot | Wait 300 ms. |
| `0xFD` | `ERR_SENSOR_OVERFLOW` | Signal clipping on the ADC rails (full-scale exceeded) | Reduce current or use a larger CT. |
| `0xFE` | `ERR_PARAM` | Invalid command code or out-of-range value | Check command and value range. |
| `0xFF` | `ERR_UNHANDLED` | HardFault or unhandled condition | Power-cycle. If it recurs, contact the supplier. |

The register is **read-only**; it is cleared only on reset or power-cycle.

## When to contact the supplier

Symptoms that warrant service:

- The module does not exit `DATA_VALID = 0` after three power-cycles
- `ERR_UNHANDLED` (`0xFF`) recurs
- `I0_GAIN` (`0xF4`) lies outside 0.9…1.1, or is unreadable
- The enclosure is hot or smells of burnt material
- Disagreement with a reference meter > 5 % despite the checklist in #18 being satisfied
- A new module boots with `ERR_LUT_BAD` (`0xFA`) on first power-on

## See also

- [Overview](overview.md) — variants, tiers, what rbAmp does and does not do
- [Hardware connection](hardware-connection.md) — wiring reference
- [Initialization](initialization.md) — bring-up, CT model setup, address change
- [Real-time polling](realtime-polling.md) — RT register reference
- [Period metering](period-metering.md) — atomic latch and tariff accounting


---

[← API Reference](api-reference.md) | [Contents](README.md)
