# API Reference
Formal reference for the rbAmp I2C register interface. This chapter is the authoritative specification: every register, every command, every error code, with size, access, units, default value, tier availability, and operational dependencies.

For prose-style explanations of what these registers mean and when to use them, see the preceding chapters. This chapter is the **machine-readable contract**.

## Table of contents

- [1. Bus protocol](#1-bus-protocol)
- [2. Tier availability](#2-tier-availability)
- [3. Conventions](#3-conventions)
- [4. Register map](#4-register-map)
  - [4.1 Control & status (0x00–0x07)](#41-control--status-0x000x07)
  - [4.2 Diagnostic LUT view (0x08–0x0F)](#42-diagnostic-lut-view-0x080x0f)
  - [4.3 Mains info (0x20–0x23)](#43-mains-info-0x200x23)
  - [4.4 I2C configuration (0x30)](#44-i2c-configuration-0x30)
  - [4.5 Current-sensor legacy block (0x54–0x7D)](#45-current-sensor-legacy-block-0x540x7d)
  - [4.6 Charge accumulator (0x7E–0x85, legacy)](#46-charge-accumulator-0x7e0x85-legacy)
  - [4.7 Real-time measurement results (0x86–0xCF)](#47-real-time-measurement-results-0x860xcf)
  - [4.8 Reactive power (0xD0–0xDB)](#48-reactive-power-0xd00xdb)
  - [4.9 Period snapshot (0xDC–0xEF)](#49-period-snapshot-0xdc0xef)
  - [4.10 Bidirectional period accumulator (STANDARD / PRO)](#410-bidirectional-period-accumulator-standard--pro)
  - [4.11 Noise floor (0xE4–0xEB)](#411-noise-floor-0xe40xeb)
  - [4.12 Gain calibration (0xF0–0xFF)](#412-gain-calibration-0xf00xff)
- [5. Commands](#5-commands)
- [6. Error codes](#6-error-codes)
- [7. Operation sequences](#7-operation-sequences)
- [8. Dependency summary](#8-dependency-summary)

---

## 1. Bus protocol

| Parameter | Value |
|---|---|
| Bus | I2C (Standard / Fast) |
| Slave role | Slave only (master in the system is external) |
| Default 7-bit address | `0x50` |
| Reprovisionable address range | `0x08..0x77` |
| Standard speed | 100 kHz |
| Fast speed | 400 kHz (validated) |
| Pointer auto-increment | **NOT supported** — every byte requires its own transaction with an explicit register address |
| Multi-byte endianness | **Little-endian** (LE) |
| Float format | IEEE-754 single precision (4 bytes, LE) |
| Multi-byte atomicity | The firmware updates whole register groups under critical-section locks; the master never sees torn snapshots when reading sequential bytes inside one group. |
| General Call (`0x00`) | Accepted for a limited command set — see [5. Commands](#5-commands) |

### Reading a 4-byte float (master pseudo-code)

```c
uint8_t buf[4];
for (int i = 0; i < 4; i++) {
    buf[i] = i2c_read_byte(addr, reg + i);    // explicit register address each time
}
float value;
memcpy(&value, buf, 4);
```

### Reading a multi-byte integer

```c
uint8_t buf[4];
for (int i = 0; i < 4; i++) buf[i] = i2c_read_byte(addr, reg + i);
uint32_t value = (uint32_t)buf[0] | (uint32_t)buf[1] << 8
               | (uint32_t)buf[2] << 16 | (uint32_t)buf[3] << 24;
```

---

## 2. Tier availability

rbAmp ships in three product tiers — each tier is a complete combination of hardware revision and firmware:

| Tier | Hardware | Firmware capability |
|---|---|---|
| **BASIC** | Entry-level analog front-end | Unidirectional consumption metering only |
| **STANDARD** | Extended analog stack | Bidirectional metering (consumption + export) |
| **PRO** | PRO-grade analog + tighter accuracy | Bidirectional metering + extended diagnostics |

### Register availability matrix

| Register range | BASIC | STANDARD | PRO |
|---|:---:|:---:|:---:|
| `0x00..0x07` Control & status | ✓ | ✓ | ✓ |
| `0x08..0x0F` Diagnostic LUT view | ✓ | ✓ | ✓ |
| `0x20..0x23` Mains info | ✓ | ✓ | ✓ |
| `0x30` I2C configuration | ✓ | ✓ | ✓ |
| `0x54..0x7D` Legacy current sensor | ✓ | ✓ | ✓ |
| `0x7E..0x85` Charge accumulator (legacy) | ✓ | ✓ | ✓ |
| `0x86..0xCF` Real-time results | ✓ | ✓ | ✓ |
| `0xD0..0xDB` Reactive power | ✓ | ✓ | ✓ |
| `0xDC..0xEF` Period snapshot (consumption side) | ✓ | ✓ | ✓ |
| Period snapshot (export side, `PERIOD_AVG_P_NEG`) | — | ✓ | ✓ |
| `0xE4..0xEB` Noise floor | ✓ | ✓ | ✓ |
| `0xF0..0xFF` Gain calibration | ✓ | ✓ | ✓ |
| Extended diagnostics block (PRO-only) | — | — | ✓ |

> The exact register addresses for the bidirectional `PERIOD_AVG_P_NEG[0..2]` and the PRO-only extended diagnostics block are documented in the SKU datasheet of the STANDARD / PRO product. They are stable within each tier but tier-specific in placement.

### Bidirectional registers — runtime detection

Master code that targets all tiers should:

1. Probe a known STANDARD/PRO-only register with a 1-byte read.
2. If it returns `NACK`, the module is BASIC tier — disable export-metering features.
3. If it returns a value, the module supports bidirectional metering — enable both consumption and export accumulators.

---

## 3. Conventions

### Access notation
- **R** — read-only (writes return `ERR_PARAM`)
- **W** — write-only (reads return implementation detail; do not rely on the value)
- **RW** — read-write

### Size and type
- **u8** — unsigned 8-bit
- **u16 LE** — unsigned 16-bit, low byte at lower address
- **u32 LE** — unsigned 32-bit, low byte at lower address
- **i8** — signed 8-bit (two's complement)
- **f32 LE** — IEEE-754 single-precision float, low byte at lower address

### Persistence
- **RAM** — value is lost on power cycle
- **Flash** — value is persisted in on-chip flash, restored at boot
- **Flash (via SAVE_GAINS)** — value lives in RAM until the master issues `CMD_SAVE_GAINS`, which writes the entire parameter block to flash

### Dependencies
Marked in the "Depends on" column for each register where applicable:
- **DATA_VALID** — master must wait `(0xCE & 0x01) == 1` after boot before reading
- **PERIOD_VALID** — master must read `(0x07 & 0x01) == 1` after `CMD_LATCH_PERIOD` before trusting the snapshot
- **CMD_LATCH_PERIOD** — register is updated only after a latch command
- **CMD_SAVE_GAINS** — write to register is non-persistent until SAVE_GAINS is issued

---

## 4. Register map

> **Reading the tables.** Each row has two lines in the **Register** column. The first line shows the address range and the canonical register name. The second line is a compact metadata strip with these fields (separated by `·`):
>
> - **Access** — `READ`, `WRITE`, or `READ/WRITE`.
> - **Type** — the storage type (which also implies size): `u8` = 1 byte, `u16 LE` = 2 bytes little-endian, `u32 LE` = 4 bytes LE, `i8` = 1 byte signed, `f32 LE` = 4-byte IEEE-754 float LE. See [3. Conventions](#3-conventions).
> - **Tier** — which product tiers / variants expose the register: `All` = every SKU and tier; `UI*` = voltage-equipped variants only (UI1, UI2, UI3); `UI2 / UI3` = variant-specific; `STANDARD / PRO` = bidirectional tiers only.
> - **Unit** — physical unit when applicable (`V`, `A`, `W`, `VAR`, `Hz`, `ms`, `ADC counts`, etc.).
> - **Sign** — `±` if the value is signed (`P`, `PF`, `Q`); `≥ 0` if always non-negative.
> - **Default** — the factory-default value for `READ/WRITE` registers, where one applies.


### 4.1 Control & status (0x00–0x07)

| Register | Description | Depends on |
|---|---|:---:|
| **`0x00`** `STATUS`<br><small>READ · u8 · Tier: All</small> | `bit0` = ready, `bit1` = error (mirrors `ERROR != 0`) | — |
| **`0x01`** `COMMAND`<br><small>WRITE · u8 · Tier: All</small> | Write a command code — see [5. Commands](#5-commands). Reads return implementation detail. | — |
| **`0x02`** `ERROR`<br><small>READ · u8 · Tier: All</small> | Last error code; `0x00` = OK, see [6. Error codes](#6-error-codes). Cleared only on reset. | — |
| **`0x03`** `VERSION`<br><small>READ · u8 · Tier: All</small> | Firmware version (`0x01` = v1.0) | DATA_VALID not required, but value of 0 is suspect |
| **`0x05`** `CT_MODEL`<br><small>READ/WRITE · u8 · Tier: All (modules with plug-in CT)</small> | CT sensor model code, see table below | Flash (via SAVE_GAINS) |
| **`0x06`** `V03_PHASE_SAMPLES`<br><small>READ/WRITE · u8 · Tier: All (UI*)</small> | Phase compensation: ADC samples to advance U relative to I in the cross-product. Range `0..30`, default `0`. Compensates voltage-transformer phase lag (~3.6° per sample at 5 kHz / 50 Hz). | Flash (via SAVE_GAINS) |
| **`0x07`** `PERIOD_VALID`<br><small>READ · u8 · Tier: All</small> | Set by `CMD_LATCH_PERIOD`. `bit0` = 1 if the snapshot at `0xDC..0xEF` and `0xC2..0xC9` contains fresh data; 0 = stale (premature latch or race). **Master must check this before using the snapshot.** | CMD_LATCH_PERIOD |

#### CT model codes (`REG_CT_MODEL`, `0x05`)

| Code | Sensor | Rated current | Sensitivity |
|:---:|---|:---:|---|
| `0x00` | not set (factory default on plug-in-CT SKUs) | — | — |
| `0x01` | SCT-013-005 | 5 A | 200 mV/A |
| `0x02` | SCT-013-010 | 10 A | 100 mV/A |
| `0x03` | SCT-013-030 | 30 A | 33 mV/A |
| `0x04` | SCT-013-050 | 50 A | 20 mV/A |
| `0x05` | SCT-013-100 | 100 A | 10 mV/A |
| `0x06` | CT-005A (generic) | 5 A | 10 mV/A |

Writing the code updates RAM immediately; persistence requires `CMD_SAVE_GAINS` (`0x26`).

---

### 4.2 Diagnostic LUT view (0x08–0x0F)

These registers expose ADC linearization-LUT status for diagnostic purposes. Typical user code does not read them; production-quality firmware always sets them correctly at boot.

| Register | Description | Depends on |
|---|---|:---:|
| **`0x08`** `LUT_VALID_MASK`<br><small>READ · u8 · Tier: All</small> | bit `n` = 1 if LUT slot `n` (channel) has a valid stored LUT; 0 = linear approximation in use | DATA_VALID |
| **`0x09`** `LUT_QUERY_SLOT`<br><small>READ/WRITE · u8 · Tier: All</small> | Write `0..3` to select a slot; the slot's metadata appears at `0x0A..0x0F` | DATA_VALID |
| **`0x0A`** `LUT_VIEW_TIER`<br><small>READ · u8 · Tier: All</small> | Tier of the selected slot's LUT: `0` = BASIC, `1` = STANDARD/PRO | `0x09` written |
| **`0x0B`** `LUT_VIEW_POINTS_LOG2`<br><small>READ · u8 · Tier: All</small> | Log₂ of the LUT point count: `8` (256 points) or `9` (512 points) | `0x09` written |
| **`0x0C..0x0D`** `LUT_VIEW_INL_MAX`<br><small>READ · u16 LE · Tier: All</small> | Measured `INL_max` (LSB of full scale) for the selected slot | `0x09` written |
| **`0x0E..0x0F`** `LUT_VIEW_DNL_MAX`<br><small>READ · u16 LE · Tier: All</small> | Measured `DNL_max` (LSB of full scale) for the selected slot | `0x09` written |

---

### 4.3 Mains info (0x20–0x23)

| Register | Description | Depends on |
|---|---|:---:|
| **`0x20`** `AC_FREQ`<br><small>READ · u8 · Tier: All (UI*) · Unit: Hz</small> | Mains frequency (50 or 60). `0` if no zero-cross detected (e.g. on I-only variants, or U sensor not yet seeing the line). | DATA_VALID |
| **`0x21..0x22`** `AC_PERIOD`<br><small>READ · u16 LE · Tier: All (UI*) · Unit: µs</small> | Half-period in microseconds (10 ms at 50 Hz, 8.33 ms at 60 Hz). Diagnostic. | DATA_VALID |
| **`0x23`** `CALIBRATION`<br><small>READ · u8 · Tier: All · Unit: flag</small> | `0` = calibration in progress, `1` = done. Boot-time diagnostic. | — |

---

### 4.4 I2C configuration (0x30)

| Register | Description | Depends on |
|---|---|:---:|
| **`0x30`** `I2C_ADDRESS`<br><small>READ/WRITE · u8 · Tier: All</small> | 7-bit slave address, range `0x08..0x77`. Writes outside the range are rejected with `ERR_PARAM`. | Flash (via SAVE_GAINS) + RESET |

To change the bus address:

1. Write the new address to `0x30`.
2. Issue `CMD_SAVE_GAINS` (`0x26`) to persist.
3. Wait ≥ 700 ms for the flash write.
4. Issue `CMD_RESET` (`0x01`) or power-cycle.
5. Communicate with the module on the new address.

---

### 4.5 Current-sensor legacy block (0x54–0x7D)

These registers implement the legacy current-sensor model (v0.2). Modern integrations use the V03 block starting at `0x86`. The legacy block is kept for backward compatibility — typical user code can ignore it.

| Register | Description |
|---|---|
| **`0x54`** `CS_CONFIG`<br><small>READ/WRITE · u8 · Tier: All · Default: 0x01</small> | `bit0` = channel-0 enable (backward compat) |
| **`0x55..0x56`** `CS_INTERVAL`<br><small>READ/WRITE · u16 LE · Tier: All · Default: 200</small> | No-op (kept for compat) |
| **`0x57`** `CS0_SENSOR_TYPE`<br><small>READ/WRITE · u8 · Tier: All · Default: 66</small> | Sensitivity mV/A (66 = ACS712-30A); used by legacy ADC pipeline |
| **`0x58`** `ACC_SEL`<br><small>READ/WRITE · u8 · Tier: All · Default: 0</small> | Selects which of 8 accumulators is reflected in the `0x60..0x77` window |
| **`0x59`** `COMMIT`<br><small>WRITE · u8 · Tier: All</small> | Write `N` (0..7) to commit accumulator `N` and clear the PA2 DRDY flag |
| **`0x5A`** `CS0_MODE`<br><small>READ/WRITE · u8 · Tier: All · Default: 0</small> | Sensor mode: `0` = current (ACS712-style), `1` = voltage (ZMPT107-style) |
| **`0x5B`** `CS0_NOISE_FLOOR`<br><small>READ/WRITE · u8 · Tier: All · Default: 19</small> | ADC noise floor (counts) for quadrature subtraction |
| **`0x5C..0x5F`** `CS_PERIOD_BUFS`<br><small>READ/WRITE · u32 LE · Tier: All · Default: 10</small> | No-op (kept for compat). 1 hour = 180 000 bufs. |
| **`0x60`** `CS0_STATUS`<br><small>READ · u8 · Tier: All</small> | `bit0` = data ready, `bit1` = RT mode, `bit7:5` = accumulator index |
| **`0x61..0x62`** `CS0_RMS`<br><small>READ · u16 LE · Tier: All</small> | RMS current (mA) |
| **`0x63..0x64`** `CS0_PEAK`<br><small>READ · u16 LE · Tier: All</small> | Peak current (mA) |
| **`0x65`** `CS0_DIR`<br><small>READ · i8 · Tier: All</small> | DC bias direction: `+1` = positive, `0` = zero, `-1` = negative |
| **`0x66`** `CS0_PERIOD_IDX`<br><small>READ · u8 · Tier: All</small> | Monotonic period index (0..255, wraps) |
| **`0x67..0x6A`** `CS0_DURATION`<br><small>READ · u32 LE · Tier: All</small> | Window duration (ms) |
| **`0x6B..0x6E`** `CS0_SAMPLES`<br><small>READ · u32 LE · Tier: All</small> | Sample count in window |
| **`0x6F..0x70`** `CS0_MIN`<br><small>READ · u16 LE · Tier: All</small> | Minimum ADC value in window |
| **`0x71..0x72`** `CS0_MAX`<br><small>READ · u16 LE · Tier: All</small> | Maximum ADC value in window |
| **`0x73..0x74`** `CS0_DC`<br><small>READ · i16 LE · Tier: All</small> | DC offset (signed, ADC units) |
| **`0x75..0x76`** `CS0_CREST`<br><small>READ · u16 LE · Tier: All</small> | Crest factor × 100 (141 = pure sine) |
| **`0x77`** `CS0_RESERVED`<br><small>READ · u8 · Tier: All</small> | Reserved (future THD) |
| **`0x78`** `VS_STATUS`<br><small>READ · u8 · Tier: All</small> | `bit7` = NO_HW (voltage sensor not present), `bit0` = data ready |
| **`0x79..0x7A`** `VS_RMS`<br><small>READ · u16 LE · Tier: All</small> | Voltage RMS (0.1 V units) — legacy U16 path; modern code uses the float at `0x86` |
| **`0x7B..0x7C`** `VS_PEAK`<br><small>READ · u16 LE · Tier: All</small> | Voltage peak (0.1 V units) |
| **`0x7D`** `VS_RATIO`<br><small>READ/WRITE · u8 · Tier: All · Default: 230</small> | Voltage transformer ratio (1..255) |

---

### 4.6 Charge accumulator (0x7E–0x85, legacy)

Volatile (RAM only) Coulomb counter for current-only modules. Cleared on power cycle and by `CMD_CHARGE_RESET` (`0x05`).

| Register | Description |
|---|---|
| **`0x7E..0x81`** `CHARGE_Q`<br><small>READ · u32 LE · Tier: All (I-only) · Unit: 0.1 mA·h</small> | Accumulated charge. Q = Σ(I_rms_mA) / 1800 (1 period = 200 ms = 1/18000 h) |
| **`0x82..0x85`** `CHARGE_N`<br><small>READ · u32 LE · Tier: All (I-only) · Unit: periods</small> | Period count (1 period = 200 ms). Useful for averaging: `avg_mA = Q × 18000 / N`. |

Volume sample: at 10 A continuous, Q increments by 10 000 (0.1 mA·h) every 200 ms — i.e. 5 (mA·h) per second. The `u32` maximum (`4 294 967 295` × 0.1 mA·h ≈ 429 496 Ah) covers > 10 years of continuous 5 A operation.

> The charge accumulator is not the same as the period-snapshot energy primitive. It is a current-time integral (Ampere-hours), not an energy integral (Watt-hours). Use it on I-only modules where the master has no voltage sensor; for energy use the period snapshot at `0xDC..0xEF`.

---

### 4.7 Real-time measurement results (0x86–0xCF)

All values are float32 little-endian unless noted. Refreshed approximately every 200 ms by the firmware after each RT window. The whole block is updated atomically inside the publishing ISR; reading any subset after `DATA_VALID = 1` returns coherent values.

#### Voltage

| Register | Description | Depends on |
|---|---|:---:|
| **`0x86..0x89`** `U_RMS`<br><small>READ · f32 LE · Tier: All (UI*) · Unit: V · Sign: ≥ 0</small> | RMS voltage over the last RT window | DATA_VALID |
| **`0x8A..0x8D`** `U_PEAK`<br><small>READ · f32 LE · Tier: All (UI*) · Unit: V · Sign: ≥ 0</small> | Peak excursion from mean during the window | DATA_VALID |

#### Currents

| Register | Description | Depends on |
|---|---|:---:|
| **`0x8E..0x91`** `I0_RMS`<br><small>READ · f32 LE · Tier: All · Unit: A · Sign: ≥ 0</small> | RMS current, channel 0 | DATA_VALID |
| **`0x92..0x95`** `I1_RMS`<br><small>READ · f32 LE · Tier: UI2 / UI3 / I2 / I3 · Unit: A · Sign: ≥ 0</small> | RMS current, channel 1 (0 on UI1 / I1) | DATA_VALID |
| **`0x96..0x99`** `I2_RMS`<br><small>READ · f32 LE · Tier: UI3 / I3 · Unit: A · Sign: ≥ 0</small> | RMS current, channel 2 | DATA_VALID |
| **`0x9A..0x9D`** `I0_PEAK`<br><small>READ · f32 LE · Tier: All · Unit: A · Sign: ≥ 0</small> | Peak current, channel 0 | DATA_VALID |
| **`0x9E..0xA1`** `I1_PEAK`<br><small>READ · f32 LE · Tier: UI2 / UI3 / I2 / I3 · Unit: A · Sign: ≥ 0</small> | Peak current, channel 1 | DATA_VALID |
| **`0xA2..0xA5`** `I2_PEAK`<br><small>READ · f32 LE · Tier: UI3 / I3 · Unit: A · Sign: ≥ 0</small> | Peak current, channel 2 | DATA_VALID |

On variants with fewer channels than the SKU's maximum, unsupported channel registers return `0.0f` rather than `NACK`.

#### Power (active)

| Register | Description | Depends on |
|---|---|:---:|
| **`0xA6..0xA9`** `P0_REAL`<br><small>READ · f32 LE · Tier: UI* · Unit: W · Sign: ±</small> | Active power, channel 0. `P > 0` = consumption, `P < 0` = export. `0` on I-only variants. | DATA_VALID |
| **`0xAA..0xAD`** `P1_REAL`<br><small>READ · f32 LE · Tier: UI2 / UI3 · Unit: W · Sign: ±</small> | Active power, channel 1. `0` on UI1 and I-only. | DATA_VALID |
| **`0xAE..0xB1`** `P2_REAL`<br><small>READ · f32 LE · Tier: UI3 · Unit: W · Sign: ±</small> | Active power, channel 2. `0` on UI1 / UI2 and I-only. | DATA_VALID |

#### Power factor

| Register | Description | Depends on |
|---|---|:---:|
| **`0xB2..0xB5`** `PF0`<br><small>READ · f32 LE · Tier: UI* · Unit: – · Sign: ±</small> | Power factor for channel 0, range `−1 … +1`. Sign matches `P0_REAL`. | DATA_VALID |
| **`0xB6..0xB9`** `PF1`<br><small>READ · f32 LE · Tier: UI2 / UI3 · Unit: – · Sign: ±</small> | Power factor, channel 1 | DATA_VALID |
| **`0xBA..0xBD`** `PF2`<br><small>READ · f32 LE · Tier: UI3 · Unit: – · Sign: ±</small> | Power factor, channel 2 | DATA_VALID |

#### Period diagnostic / per-channel period averages

| Register | Description | Depends on |
|---|---|:---:|
| **`0xBE..0xC1`** `PERIOD_COMMIT_COUNT`<br><small>READ · u32 LE · Tier: All · Unit: count · Sign: ≥ 0</small> | Diagnostic: number of RT commits accumulated in the current period (latched by `CMD_LATCH_PERIOD`) | CMD_LATCH_PERIOD |
| **`0xC2..0xC5`** `PERIOD_AVG_P_W[1]`<br><small>READ · f32 LE · Tier: UI2 / UI3 · Unit: W · Sign: ± (BASIC: ≥ 0)</small> | Time-averaged active power, channel 1, over the latched period | CMD_LATCH_PERIOD, PERIOD_VALID |
| **`0xC6..0xC9`** `PERIOD_AVG_P_W[2]`<br><small>READ · f32 LE · Tier: UI3 · Unit: W · Sign: ± (BASIC: ≥ 0)</small> | Time-averaged active power, channel 2 | CMD_LATCH_PERIOD, PERIOD_VALID |
| **`0xCA..0xCD`** `RT_PERIOD_MS`<br><small>READ · u32 LE · Tier: All · Unit: ms · Sign: ≥ 0</small> | Actual wall-clock duration of the last RT window (~200 ms). Diagnostic only. | DATA_VALID |
| **`0xCE`** `DATA_VALID`<br><small>READ · u8 · Tier: All · Unit: flag</small> | `bit0` = 1 once the first valid RT window has been computed since boot. Master must wait for this bit after power-on. | — |
| **`0xCF`** `reserved`<br><small>READ · u8 · Tier: All</small> | Reserved; reads 0 | — |

> On BASIC product tier, `PERIOD_AVG_P_W[ch]` is clamped to ≥ 0 — the per-window accumulator drops negative samples. On STANDARD / PRO it represents the consumption side of the bidirectional accumulator.

---

### 4.8 Reactive power (0xD0–0xDB)

| Register | Description | Depends on |
|---|---|:---:|
| **`0xD0..0xD3`** `Q0_REAC`<br><small>READ · f32 LE · Tier: UI* · Unit: VAR · Sign: ±</small> | Reactive power, channel 0. `+` = inductive, `−` = capacitive (IEEE 1459 convention). `0` on I-only. | DATA_VALID |
| **`0xD4..0xD7`** `Q1_REAC`<br><small>READ · f32 LE · Tier: UI2 / UI3 · Unit: VAR · Sign: ±</small> | Reactive power, channel 1 | DATA_VALID |
| **`0xD8..0xDB`** `Q2_REAC`<br><small>READ · f32 LE · Tier: UI3 · Unit: VAR · Sign: ±</small> | Reactive power, channel 2 | DATA_VALID |

---

### 4.9 Period snapshot (0xDC–0xEF)

The master-side energy primitive. Updated only by `CMD_LATCH_PERIOD` (`0x27`). Each latch atomically copies the live accumulator into this snapshot and resets the live accumulator to zero — the snapshot reflects the period that just ended.

| Register | Description | Depends on |
|---|---|:---:|
| **`0xDC..0xDF`** `PERIOD_AVG_P_W[0]`<br><small>READ · f32 LE · Tier: UI* · Unit: W · Sign: ± (BASIC: ≥ 0)</small> | Time-averaged active power, channel 0, over the latched period. **Production energy primitive.** Master computes `E_Wh = avg_p × dt_s / 3600`. | CMD_LATCH_PERIOD, PERIOD_VALID |
| **`0xE0..0xE3`** `PERIOD_MAX_P_W`<br><small>READ · f32 LE · Tier: UI* · Unit: W · Sign: ±</small> | Peak `P_real` observed in any RT window inside the latched period (channel 0). For peak-demand tariffs. | CMD_LATCH_PERIOD, PERIOD_VALID |
| **`0xE4..0xEB`** `(noise floor, see [4.11](#411-noise-floor-0xe40xeb))`<br><small></small> |  |  |
| **`0xEC..0xEF`** `PERIOD_LATCH_MS`<br><small>READ · u32 LE · Tier: All · Unit: ms · Sign: ≥ 0</small> | Chip's view of dt between successive `CMD_LATCH_PERIOD` commands. Diagnostic only — do not use for Wh computation; the master clock is authoritative. | CMD_LATCH_PERIOD, PERIOD_VALID |

> The address range `0xE4..0xEB` interleaves with the noise-floor registers — see [4.11](#411-noise-floor-0xe40xeb).

#### Master-side energy formula

```
E_Wh[ch] = PERIOD_AVG_P_W[ch] × master_dt_s / 3600
```

where `master_dt_s` is the wall-clock elapsed between two successive `CMD_LATCH_PERIOD` commands, measured by the master.

---

### 4.10 Bidirectional period accumulator (STANDARD / PRO)

> **STANDARD / PRO tiers only.** Not present on BASIC modules — these registers `NACK` on BASIC.

| Register | Description | Depends on |
|---|---|:---:|
| **`PERIOD_AVG_P_NEG_W[0]`**<br><small>Tier: STANDARD / PRO · Unit: W · Sign: ≥ 0</small> | Time-averaged **export** power, channel 0, over the latched period (magnitude of negative `P_real`). Master computes `E_export_Wh = avg_p_neg × dt_s / 3600`. | CMD_LATCH_PERIOD, PERIOD_VALID |
| **`PERIOD_AVG_P_NEG_W[1]`**<br><small>Tier: STANDARD / PRO (UI2 / UI3) · Unit: W · Sign: ≥ 0</small> | Export-side average, channel 1 | CMD_LATCH_PERIOD, PERIOD_VALID |
| **`PERIOD_AVG_P_NEG_W[2]`**<br><small>Tier: STANDARD / PRO (UI3) · Unit: W · Sign: ≥ 0</small> | Export-side average, channel 2 | CMD_LATCH_PERIOD, PERIOD_VALID |

The numeric register addresses for `PERIOD_AVG_P_NEG_W[0..2]` are SKU-specific and documented in the STANDARD / PRO product datasheet. They are stable within each tier.

Master code that wants tier-portable bidirectional support should:

1. Probe a candidate `PERIOD_AVG_P_NEG_W` address with a 1-byte read.
2. On `NACK`, fall back to RT-based bidirectional integration on the master side (see chapter 10, example 5).
3. On success, use the SKU-documented address set.

---

### 4.11 Noise floor (0xE4–0xEB)

Quadrature noise subtraction: `rms_corrected = √(rms_raw² − NF²)`. The defaults are calibrated at the factory; user overrides are rarely needed.

| Register | Description | Depends on |
|---|---|:---:|
| **`0xE4..0xE5`** `U_NF`<br><small>READ/WRITE · u16 LE · Tier: UI* · Unit: ADC counts · Default: 25</small> | Voltage RMS noise floor | Flash (via SAVE_GAINS) |
| **`0xE6..0xE7`** `I0_NF`<br><small>READ/WRITE · u16 LE · Tier: All · Unit: ADC counts · Default: 12</small> | Current channel 0 noise floor | Flash (via SAVE_GAINS) |
| **`0xE8..0xE9`** `I1_NF`<br><small>READ/WRITE · u16 LE · Tier: UI2 / UI3 / I2 / I3 · Unit: ADC counts · Default: 12</small> | Current channel 1 noise floor | Flash (via SAVE_GAINS) |
| **`0xEA..0xEB`** `I2_NF`<br><small>READ/WRITE · u16 LE · Tier: UI3 / I3 · Unit: ADC counts · Default: 12</small> | Current channel 2 noise floor | Flash (via SAVE_GAINS) |

---

### 4.12 Gain calibration (0xF0–0xFF)

Applied as a final multiplier in `Sensor_V03_Commit()` after the RMS/power computation. Writing all 4 bytes of a gain takes effect immediately in RAM; persistence requires `CMD_SAVE_GAINS`.

| Register | Description | Depends on |
|---|---|:---:|
| **`0xF0..0xF3`** `U_GAIN`<br><small>READ/WRITE · f32 LE · Tier: UI* · Default: 1.0</small> | Voltage channel gain multiplier | Flash (via SAVE_GAINS) |
| **`0xF4..0xF7`** `I0_GAIN`<br><small>READ/WRITE · f32 LE · Tier: All · Default: 1.0</small> | Current channel 0 gain multiplier | Flash (via SAVE_GAINS) |
| **`0xF8..0xFB`** `I1_GAIN`<br><small>READ/WRITE · f32 LE · Tier: UI2 / UI3 / I2 / I3 · Default: 1.0</small> | Current channel 1 gain multiplier | Flash (via SAVE_GAINS) |
| **`0xFC..0xFF`** `I2_GAIN`<br><small>READ/WRITE · f32 LE · Tier: UI3 / I3 · Default: 1.0</small> | Current channel 2 gain multiplier | Flash (via SAVE_GAINS) |

Safe range: `0.5..2.0`. Values outside this range almost certainly indicate a setup error and should not be written.

---

## 5. Commands

Write a command code to `REG_COMMAND` (`0x01`). The firmware processes the command within a few milliseconds (see per-command latency below) and stores any resulting error in `REG_ERROR`.

| Command | Effect | Wait | GC |
|---|---|:---:|:---:|
| **`0x00`** `CMD_NOP`<br><small>Tier: All</small> | No operation | — | yes |
| **`0x01`** `CMD_RESET`<br><small>Tier: All</small> | Software reset; the module reboots and reappears at the same I2C address ~300 ms later | 300 ms | **yes** |
| **`0x02`** `CMD_RECALIBRATE`<br><small>Tier: All (UI*)</small> | Re-measure mains half-period from zero-cross detector | 50 ms | — |
| **`0x03`** `CMD_SWITCH_UART`<br><small>Tier: All</small> | Toggle UART debug output (development builds only — no effect on production firmware) | — | — |
| **`0x05`** `CMD_CHARGE_RESET`<br><small>Tier: All</small> | Reset the charge accumulator (`Q` and `N` at `0x7E..0x85`) to zero | — | — |
| **`0x26`** `CMD_SAVE_GAINS`<br><small>Tier: All</small> | Persist the parameter block (I2C address, CT model, noise floors, gains, V03 phase samples) to flash | 700 ms | — |
| **`0x27`** `CMD_LATCH_PERIOD`<br><small>Tier: All</small> | Atomic snapshot + reset of the period accumulator. `0xDC..0xEF` and `0xC2..0xC9` are updated; `PERIOD_VALID` is set to 1 if the latched period contained ≥ 1 RT window. | 50 ms | **yes** |
| **`0xAA`** `CMD_FACTORY_RESET`<br><small>Tier: All</small> | Reset all flash parameters to factory defaults and reboot | 1 s | — |

Other codes return `ERR_PARAM` in `REG_ERROR`.

### General Call (broadcast)

Sending a command to I2C address `0x00` (general call) reaches all slaves simultaneously. rbAmp accepts a subset of commands on general call — currently `CMD_LATCH_PERIOD` and `CMD_RESET`. Address-specific commands (e.g. `CMD_SAVE_GAINS`) are ignored on general call.

Wire-level example:

```
START | 0x00 W | 0x01 | 0x27 | STOP    -> all modules latch their snapshot
```

---

## 6. Error codes

Returned from `REG_ERROR` (`0x02`). Read-only; cleared only by a reset (`CMD_RESET` or power-cycle).

| Code | Meaning | Action |
|---|---|---|
| **`0x00`** `ERR_OK` | Normal operation | — |
| **`0xFA`** `ERR_LUT_BAD` | ADC linearization LUT failed CRC at boot; running on linear approximation. Inspect `LUT_VALID_MASK` (`0x08`) for per-channel detail. Not fatal — measurements still work, with slightly degraded INL. | Have the supplier recalibrate / reprovision. |
| **`0xFB`** `ERR_FLASH_PARAMS_BAD` | Flash parameter block (page 511) failed CRC at boot; running on factory defaults. Defaults have been re-saved, so the next boot will be clean. | Rewrite custom parameters (I2C address, CT model) and issue `CMD_SAVE_GAINS`. |
| **`0xFC`** `ERR_NOT_READY` | The sensor pipeline has not yet committed its first RT window after boot (warm-up < 250 ms) | Wait 300 ms and recheck. |
| **`0xFD`** `ERR_SENSOR_OVERFLOW` | RMS computation hit the overflow guard — input clipping at the ADC rails for the entire window | Reduce current, or fit a CT with a larger range. |
| **`0xFE`** `ERR_PARAM` | Invalid command code, out-of-range register write, or general-call write of an address-specific command | Check the command code and the value range. |
| **`0xFF`** `ERR_UNHANDLED` | HardFault or unhandled condition — should never appear in normal operation | Power-cycle. If it recurs, contact the supplier. |

Reserved encoding: any value with `bit 7 = 1` indicates an error. Codes `0x01..0xF9` are reserved for future use.

---

## 7. Operation sequences

### 7.1 Boot smoke test

```
1. Apply power.
2. Wait 300 ms.
3. Read REG_VERSION (0x03)  — expect non-zero firmware version byte.
4. Read REG_DATA_VALID (0xCE), bit0  — wait until == 1.
5. Read REG_ERROR (0x02)   — expect 0x00 (or a known non-fatal code).
6. Read RT registers as needed.
```

### 7.2 Reading a real-time snapshot

```
1. (Optional) Wait for DRDY falling edge for tight synchronisation.
2. Read U_RMS    (4 bytes from 0x86)
3. Read I0_RMS   (4 bytes from 0x8E)
4. Read P0_REAL  (4 bytes from 0xA6)
5. Read PF0      (4 bytes from 0xB2)
6. (Optional) Read AC_FREQ (1 byte at 0x20)
```

### 7.3 Period metering — continuous polling

```
Master setup:
1. Write CMD_LATCH_PERIOD (0x27) to REG_COMMAND (0x01)  -- primer; discard result
2. Record t_prev = master_clock()

For each measurement period:
3. Wait the desired interval (e.g. 60 s)
4. Write CMD_LATCH_PERIOD
5. Record t_now = master_clock()
6. Wait 50 ms (let firmware finalise the snapshot)
7. Read REG_V03_PERIOD_VALID (0x07), bit0
       == 0  -> see retry pattern below
       == 1  -> continue
8. Read PERIOD_AVG_P_W[0] (4 bytes from 0xDC)
9. Compute E_Wh = avg_p × (t_now - t_prev) / 3600
10. t_prev = t_now
```

Retry pattern when `PERIOD_VALID == 0`:

```
Wait 250 ms        -- let at least one RT window complete
Write CMD_LATCH_PERIOD
Wait 50 ms
Read PERIOD_VALID
if still 0:
    log warning, reset t_prev = master_clock(), skip this period
```

### 7.4 Multi-module synchronous latch

```
1. General Call latch: START | 0x00 W | 0x01 | 0x27 | STOP
2. Record t_now
3. Wait 50 ms
4. For each module addr in your list:
     read PERIOD_VALID (0x07) on addr
     if valid:
         read PERIOD_AVG_P_W[0] (0xDC) on addr
         E_Wh[addr] = avg_p × (t_now - t_prev[addr]) / 3600
         t_prev[addr] = t_now
```

### 7.5 Change CT model (plug-in CT modules)

```
1. Write new model code to REG_CT_MODEL (0x05)
   e.g. rb_write_u8(addr, 0x05, 0x03)   -- SCT-013-030
2. Write CMD_SAVE_GAINS (0x26) to REG_COMMAND (0x01)
3. Wait 700 ms
4. (Optional) Read REG_CT_MODEL to verify
```

### 7.6 Change I2C address

```
1. Connect only the target module to the bus (current address: old_addr)
2. Write new_addr to REG_I2C_ADDRESS (0x30) on old_addr
3. Wait 10 ms
4. Write CMD_SAVE_GAINS (0x26) on old_addr
5. Wait 700 ms
6. Write CMD_RESET (0x01) on old_addr
7. Wait 300 ms
8. The module now responds on new_addr; verify with a read of REG_VERSION (0x03)
9. Physically mark the module with its new address
```

### 7.7 Gain calibration

```
1. Connect a known reference load (e.g. 1000 W resistive heater)
2. Read live P0_REAL several times (0xA6); average over ~5 s -> p_measured
3. Read current I0_GAIN as float (0xF4..0xF7) -> gain_now
4. Compute gain_new = gain_now × (P_reference / p_measured)
5. Sanity check: 0.5 ≤ gain_new ≤ 2.0   (otherwise abort)
6. Write gain_new as 4 bytes (LE) to 0xF4..0xF7
7. Write CMD_SAVE_GAINS (0x26)
8. Wait 700 ms
9. Verify by reading P0_REAL again — should match P_reference within calibration tolerance
```

### 7.8 Factory reset

```
1. Write CMD_FACTORY_RESET (0xAA) to REG_COMMAND (0x01)
2. Wait 1 s for the module to wipe flash and reboot
3. Wait an additional 300 ms for boot
4. Read REG_VERSION (0x03) to confirm the module is back online
5. Reconfigure as needed (address, CT model, gains)
```

---

## 8. Dependency summary

A register's value is meaningful only when its dependencies are satisfied.

| Register / range | Required precondition | Effect if unmet |
|---|---|---|
| `0x86..0xCD` Real-time results | `DATA_VALID` bit 0 == 1 | Reads return zeros / stale data |
| `0xC2..0xC9` Per-channel period averages | `CMD_LATCH_PERIOD` issued + `PERIOD_VALID` == 1 | Reads return stale or undefined data |
| `0xDC..0xDF` `PERIOD_AVG_P_W[0]` | `CMD_LATCH_PERIOD` issued + `PERIOD_VALID` == 1 | Reads return stale or undefined data |
| `0xE0..0xE3` `PERIOD_MAX_P_W` | `CMD_LATCH_PERIOD` issued + `PERIOD_VALID` == 1 | Reads return stale or undefined data |
| `0xEC..0xEF` `PERIOD_LATCH_MS` | `CMD_LATCH_PERIOD` issued | Reads return 0 until first latch |
| `0x05` `CT_MODEL` (write) | `CMD_SAVE_GAINS` to persist | Setting is RAM-only; lost on power cycle |
| `0x30` `I2C_ADDRESS` (write) | `CMD_SAVE_GAINS` + `CMD_RESET` to apply | Module keeps old address until persisted and reset |
| `0xE4..0xEB` Noise floor (write) | `CMD_SAVE_GAINS` to persist | RAM-only, lost on power cycle |
| `0xF0..0xFF` Gains (write) | `CMD_SAVE_GAINS` to persist | RAM-only, lost on power cycle |
| `CMD_LATCH_PERIOD` | One RT window (~200 ms) must have completed since the previous latch | `PERIOD_VALID` returns 0; snapshot is stale |
| `PERIOD_AVG_P_NEG_W[*]` | STANDARD / PRO product tier | Register NACKs on BASIC tier |

### Initialization order

For a master driver that brings up a fresh installation:

```
1. I2C bus init  (set frequency, configure SDA/SCL)
2. Wait 250 ms for module power-on
3. Probe address (default 0x50); if not present, scan range 0x08..0x77
4. Read REG_VERSION (0x03)
5. Poll REG_DATA_VALID (0xCE) bit 0 until 1 (timeout 2 s)
6. Read REG_ERROR (0x02); log if non-zero
7. (One-time setup, if applicable):
   - Set CT model:  write 0x05 + CMD_SAVE_GAINS
   - Set I2C addr:  write 0x30 + CMD_SAVE_GAINS + CMD_RESET
8. (Optional) Detect variant:
   - Probe 0xC2 (UI2/UI3 marker)
   - Read U_RMS (0x86): non-zero => UI*, ≈0 => I-only
9. (Optional) Detect tier:
   - Probe a STANDARD/PRO-only address; NACK => BASIC
10. Issue primer CMD_LATCH_PERIOD; discard the result
11. Start the main read loop
```

---


## See also

- [Overview](overview.md) — what rbAmp does and tier semantics
- [Hardware Connection](hardware-connection.md) — pinout, wiring, CT installation
- [Initialization](initialization.md) — bring-up examples in C and Arduino
- [Real-time Polling](realtime-polling.md) — RT register usage with code
- [Period Metering](period-metering.md) — latch semantics, energy formula
- [Arduino Examples](https://www.rbamp.com/docs/modules-basic-standard-arduino-examples) — reference sketches that exercise this API
- [Arduino Library — Overview](https://www.rbamp.com/docs/modules-basic-standard-library-arduino-overview) — high-level wrapper if you prefer a `RbAmp` class to raw registers
- [Troubleshooting](troubleshooting.md) — error code remediation


---

[← Period Metering](period-metering.md) | [Contents](README.md) | [Troubleshooting →](troubleshooting.md)
