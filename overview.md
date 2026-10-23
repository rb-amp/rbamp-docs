# 00 · Overview

## What is rbAmp

**rbAmp** is a compact hardware module for precision measurement of AC mains parameters over an I2C interface. It ships with factory-stored calibration.

From the integrator's point of view, rbAmp behaves like a standard I2C slave: apply power, read registers — receive ready-to-use physical-unit values (volts, amperes, watts). No signal processing is required on the master side.

![Block diagram of an rbAmp module connected to a master over I2C, from the mains-side sensors to the registers the master reads.](images/overview-architecture.png)

## What it measures

| Quantity | Register (I0) | Type | Range | Accuracy |
|---|:---:|---|---|---|
| RMS voltage, U_rms | `0x86` | float32, V | 0…300 V | ±0.5% |
| Peak voltage, U_peak | `0x8A` | float32, V | 0…450 V | ±1% |
| RMS current, I_rms | `0x8E` | float32, A | 0…100 A (SKU-dependent) | ±0.5%…±1% — for the current channel with an SCT-013 see the table below |
| Peak current, I_peak | `0x9A` | float32, A | 0…140 A | ±1% |
| Active power, P | `0xA6` | float32, W (±) | ±23 kW | ±1% |
| Power factor, PF | `0xB2` | float32 (±) | −1…+1 | ±0.01 at PF ≈ 1 (resistive load) |
| Reactive power, Q | `0xD0` | float32, VAr (≥ 0) | 0…23 kVAr | — |
| Line frequency | `0x20` | u8, Hz | 45…65 Hz | ±0.5 Hz |
| Time-averaged P over period | `0xDC` | float32, W | same as P | ±1% |

The full register map is detailed in [03_realtime_polling.md](realtime-polling.md) and [04_period_metering.md](period-metering.md).

### Current-channel accuracy with SCT-013 sensors

**Current-channel error — module + your own SCT-013 sensor, resistive load, measured at 50 Hz.**
Module contribution: **±1 % of reading** for averaged readings at or above the load shown; add up to
~1 % for your sensor's unit-to-unit tolerance (≈ ±2 % total on current). Below that load the error
is larger and is not specified.
(Modules with a factory-paired CT sensor, and STD modules, are specified separately — this table does not apply to them.)

| Sensor rating | Averaged readings within ±1 % from about | Power factor reads (resistive load) | Clipping begins at about |
|---|---|---|---|
| 5 A   | 0.1–0.15 A | 0.985–0.993 | 115–120 % of rating |
| 10 A  | 0.2–0.3 A  | about 0.996 | 115–120 % of rating |
| 20 A  | 0.4–0.6 A  | about 1.00 | 115–120 % of rating |
| 30 A  | 0.6–0.9 A  | about 1.00 | 115–120 % of rating |
| 50 A  | 1–1.5 A    | about 1.00 | 115–120 % of rating |
| 60 A  | 1.2–1.8 A  | about 1.00 | 115–120 % of rating |
| 100 A | 2–3 A      | about 1.00 | 115–120 % of rating |

"From about" = 2–3 % of the sensor rating. Basis of the figures: 10, 30 and 100 A were verified on a module with the
production firmware; 5, 20, 50 and 60 A are derived from measurements of the sensors themselves.

- "Averaged" = the average of several 100 ms measurement windows, as used for energy and average power. A single
  reading is noisier near the low end of the range.
- Power factor is specified only at PF ≈ 1 on a resistive load. On the 5 A sensor its own phase shift lowers the
  reading to 0.985–0.993 even on a resistive load. No accuracy is stated for reactive or distorted loads.
- Clipping is a property of the SCT-013 sensor (its output level at rated current); stay below the rating for
  reliable readings.
- Changing the sensor model is writing the model code to the module — no recalibration step.

## Applications

- **Tariff metering** — energy billing in 1-min, 15-min, 1-hour, daily, monthly, or yearly intervals
- **Smart home** — whole-house consumption monitoring, per-load sub-metering, generation tracking
- **Load control** — feedback signal for power-regulating devices (water heaters, resistive loads, inverters)
- **Energy balancing** — separate accounting of consumption and generation in homes with solar
- **Grid quality monitoring** — both on the consumer side and on the utility side

## Module variants

| Variant | U | I channels | P | Status | When to choose |
|---|:---:|:---:|:---:|---|---|
| **UI1** | yes | 1 | yes | shipping | single accounting point — whole-house meter, boiler, single inverter |
| **UI2** | yes | 2 | yes | shipping | two sub-meters on one phase (lights + outlets, generation + consumption) |
| **UI3** | yes | 3 | yes | not yet available | three flows on one phase |
| **UI5 / UI7** | yes | 5 / 7 | yes | not yet available | high-channel-count phase monitoring (large panels, lab benches) |
| **I1** | no | 1 | no | shipping | current-only monitor (master computes P with its own U source) |
| **I2** | no | 2 | no | shipping | two-channel current monitor |
| **I3** | no | 3 | no | shipping | three-channel current monitor |

> On I-only variants, active power is not computed (`P_real = 0`), and `PERIOD_AVG_P_W` returns 0. The master must either provide its own voltage source for power calculation, or use the current data alone.

## Product tiers

rbAmp is offered in two product tiers. Each tier is a complete combination of **hardware revision and firmware** — not a firmware-only flag. Moving between tiers requires a physical SKU change, not a firmware update.

![The BASIC and STANDARD (STD) rbAmp module lines side by side.](images/module-lines.png)

### BASIC — entry-level product line

- **Hardware**: cost-optimized analog front-end suitable for typical residential loads.
- **Firmware**: unidirectional consumption metering. The per-period accumulator clamps each per-window active-power sample to zero before integrating: only positive (consumed) energy is counted, mirroring a classical mechanical disc meter that only spins forward.
- **`PERIOD_AVG_P_W` is always ≥ 0**; reverse-flow events are dropped from the period accumulator.
- **Use cases**: households without on-site generation; sub-metering of pure consumers (heaters, motors, appliances).

### STANDARD — bidirectional product line

- **Hardware**: extended analog stack supporting accurate measurement of both consumption and reverse flow.
- **Firmware**: bidirectional metering with signed on-device energy. Bidirectional products belong to the **rbAmp STD line**, which uses a different, 16-bit register map — see the STD reference documentation.
- **Use cases**: homes with rooftop solar, wind generators, battery storage, regenerative loads, V2G EV chargers.

### Where the tier shows up in this documentation

Throughout the chapters, features that depend on the tier are explicitly marked, for example:

> **STANDARD (STD) only** — on a BASIC module this register is not available.

The BASIC 8-bit register map (`0x00..0xCF`) is what this documentation describes. STANDARD (STD) modules use a **different, 16-bit register map** — see the STD reference documentation.

## Key features

- **I²C slave**: recommended 50 kHz; workable at 100 kHz with application-level retry. 400 kHz is **not** validated for v1.3.
- **Address**: 7-bit `0x50` by default, reprovisionable to the `0x08..0x77` range via two-phase commit (see [02_initialization.md](initialization.md)).
- **DATA_READY (DRDY)**: optional open-drain pin, ~10 µs LOW pulse every measurement window (~100 ms on current firmware; 200 ms before v1.2), marks the moment when RT registers are coherent.
- **Atomic period latch**: command `0x27` freezes the per-period averaged power; *one write closes and opens* — no dropped or duplicated samples at the boundary.
- **General-Call broadcast latch**: address `0x00` reserved for simultaneous LATCH on all modules sharing a bus. **Opt-in per module** via `FLEET_CONFIG.bit0` (default OFF). When OFF, GC frame NACKs at the bus level — masters detect this and fall back to per-module sequential latch (skew on 50 kHz bus is ~1 ms × N, ≪ 0.03% of 60 s period). See [04_period_metering.md](period-metering.md) for enable sequence and frame format.
- **Master-side wall-clock**: energy is computed master-side as `avg_P × dt_master`; the chip-side period timestamp register (`0xEC`) is **diagnostic-only** (can undercount up to ~30% under load — by design). Eliminates per-module crystal drift in multi-module systems.
- **Calibration in flash**: factory calibration (gains, noise floor, phase compensation) is stored on-chip; **no user calibration is required** — the master only writes `SENSOR_CLASS` and `CT_MODEL` per channel.
- **Auto-configuration**: the module starts measuring immediately after power-on (~250 ms to the first valid result, ~700 ms additional for first persistence write).

## What rbAmp does not do

- Does not integrate energy (Wh) internally — energy accumulation is the master's responsibility (see [04_period_metering.md](period-metering.md))
- Does not drive loads, contain automation logic, or run programmable rules
- Has no built-in display

## Next

- [01_hardware.md](hardware-connection.md) — connection, power, wiring
- [02_initialization.md](initialization.md) — first power-on
