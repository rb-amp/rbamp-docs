# Real-Time Polling

Real-time polling — reading instantaneous values of U, I, P, PF, flow direction, and related metrics at regular intervals (typically every 200 ms). It is used for:

- Live dashboards (current power, voltage)
- Threshold alarms
- Event detection (load turn-on / turn-off)
- Mains-quality monitoring (frequency, PF)
- **Determining current direction — consumption or export** (from the sign of P_real)

For **tariff energy accumulation** (Wh over a period) use the period-latch mechanism described in [Period metering](period-metering.md).

## The 200 ms window

### What it is

The module continuously samples its ADCs at 5 kHz and produces a result set every ~200 ms:

- RMS of voltage and currents
- Peak values of U and I
- Active power P, computed as `mean(u(t) · i(t))` — **signed**, with the sign indicating flow direction:
  - `P > 0` — the load consumes from the grid (normal mode)
  - `P < 0` — the load exports to the grid (solar inverter, regenerative load)
- Reactive power Q (computed on a phase-shifted sample stream; the sign distinguishes inductive (+) from capacitive (−))
- Power factor PF = P / S, with S = U_rms × I_rms. The sign of PF matches that of P — an additional indicator of flow direction.

After computing a window, the firmware **atomically** publishes the entire result set (50+ bytes) into the I²C register block `0x86..0xCD`.

### Window length is fixed

The window is **hard-coded to 200 ms**, independent of mains frequency. The firmware collects exactly 10 ADC buffers of 20 ms each (5 kHz sample rate × 100 samples per buffer) and computes the result.

At 50 Hz mains this covers exactly 10 cycles; at 60 Hz, 12 cycles. In both cases the integer number of mains cycles keeps RMS and P accurate.

> The register `RT_period_ms` (`0xCA..0xCD`, u32 LE) reports the **actual wall-clock duration** of the last window — usually close to 200 ms. It is diagnostic only; it does not affect measurement accuracy or the energy calculation (see [Period metering](period-metering.md)).

### When the next window starts

Immediately after the previous one is published. There is no gap between windows — ADC sampling is continuous.

## DATA_READY (DRDY) — the ready pin

DRDY is an open-drain output pulled up to 3.3 V on the master side. The firmware drives it LOW for ~10 µs **after** the RT registers have been refreshed — a hardware signal that says "data is ready to read".

### Pulse parameters

- LOW duration: ~10 µs (comfortable for any master with EXTI)
- Frequency: ~5 Hz (one pulse per 200 ms window)
- Output type: open-drain (supports wired-OR in multi-module topologies)
- Semantics: **the falling edge guarantees all RT registers are coherent and up to date**

### When DRDY is not needed

- The master polls slower than 5 Hz (once per second or minute) — data is refreshed five times a second, so any read picks up fresh data.
- A trivial polling loop — `read_rt()` can be called any time; you get the latest committed result.

### When DRDY is useful

- The master wants to minimise I²C traffic or free up CPU for other tasks (battery-powered systems)
- Precise synchronisation: read the exact moment of measurement completion (useful for high-resolution logs)
- Event detection: react to every window (e.g. instantaneous-power tracking at 200 ms granularity)

### Usage (Arduino, one module)

```cpp
const uint8_t PIN_DRDY = 15;
volatile bool data_ready = false;

void on_drdy_falling() {
  data_ready = true;
}

void setup() {
  Serial.begin(115200);
  Wire.begin();
  pinMode(PIN_DRDY, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(PIN_DRDY), on_drdy_falling, FALLING);
}

void loop() {
  if (data_ready) {
    data_ready = false;
    float u = rb_read_float_le(0x50, 0x86);
    float i = rb_read_float_le(0x50, 0x8E);
    float p = rb_read_float_le(0x50, 0xA6);
    const char* dir = (p >= 0.0f) ? "consume" : "export";
    Serial.printf("U=%.1f I=%.3f P=%.1f (%s)\n", u, i, p, dir);
  }
}
```

## The sign of P and flow direction

In the RT registers, the sign of active power is the **primary indicator of current direction**:

| Condition | Meaning |
|---|---|
| `P_real > 0` | Load consumes (current flowing in) |
| `P_real < 0` | Load exports to the grid (generation) |
| `P_real ≈ 0` while `I_rms > 0` | Purely reactive load (unloaded transformer, capacitor) |

RMS values of current and voltage are **always ≥ 0** (RMS is non-negative by definition). Therefore `I_rms` **cannot** be used to determine flow direction — use `P_real`.

### Flow-classification example

```cpp
void classify_flow(float p) {
  if (p > 10.0f) {
    Serial.println("CONSUMING from grid");
  } else if (p < -10.0f) {
    Serial.println("EXPORTING to grid");
  } else {
    Serial.println("idle / reactive only");
  }
  // ±10 W threshold to avoid chatter near zero
}
```

> On BASIC-tier firmware the tariff accumulator **ignores** export (see [Period metering](period-metering.md)). The RT registers, however, always carry the real-time sign of P — so even on a BASIC SKU the master can detect export events from RT reads, log them, or build a master-side bidirectional accumulator on top of BASIC firmware.

## RT register reference

All values are float32 little-endian (4 bytes), read with 4 separate transactions (no auto-increment). Integer fields are marked with their type.

### Voltage

| Address | Field | Type | Unit | Description |
|---:|---|---|---|---|
| `0x86` | U_rms | float32 | V | RMS voltage over the last ~200 ms window |
| `0x8A` | U_peak | float32 | V | Maximum deviation from mean during the window |

### Currents (per channel: 0 = primary, 1/2 on UI2/UI3, more on UI5/UI7)

| Address | Field | Type | Unit | Description |
|---:|---|---|---|---|
| `0x8E` | I0_rms | float32 | A | RMS current, channel 0 (primary) |
| `0x92` | I1_rms | float32 | A | RMS current, channel 1 (UI2/UI3) |
| `0x96` | I2_rms | float32 | A | RMS current, channel 2 (UI3) |
| `0x9A` | I0_peak | float32 | A | Peak current sample within the window, channel 0 |
| `0x9E` | I1_peak | float32 | A | Peak current sample within the window, channel 1 |
| `0xA2` | I2_peak | float32 | A | Peak current sample within the window, channel 2 |

> On variants with fewer channels (e.g. UI1), the unused channel registers return `0.0f` instead of NACK. Higher-channel-count variants (UI5/UI7) place their additional I_rms / P_real / PF / Q values in registers above `0xCF`; refer to the SKU datasheet for the extended map.

### Power

| Address | Field | Type | Unit | Sign | Description |
|---:|---|---|---|:---:|---|
| `0xA6` | P0_real | float32 | W | ± | Active power, channel 0 — **P > 0 = consumption, P < 0 = export** |
| `0xAA` | P1_real | float32 | W | ± | Active power, channel 1 (UI2/UI3) |
| `0xAE` | P2_real | float32 | W | ± | Active power, channel 2 (UI3) |
| `0xB2` | PF0 | float32 | – | ± | Power factor, range −1…+1, sign matches P0 |
| `0xB6` | PF1 | float32 | – | ± |  |
| `0xBA` | PF2 | float32 | – | ± |  |
| `0xD0` | Q0_reac | float32 | VAR | ± | Reactive power, channel 0; +inductive, −capacitive (IEEE 1459) |
| `0xD4` | Q1_reac | float32 | VAR | ± |  |
| `0xD8` | Q2_reac | float32 | VAR | ± |  |

> Apparent power S can be derived master-side: `S = U_rms × I_rms`. RMS values are **always ≥ 0**; flow direction lives in the sign of P.

### System

| Address | Field | Type | Description |
|---:|---|---|---|
| `0x00` | STATUS | u8 | bit0 = ready, bit1 = error |
| `0x02` | ERROR | u8 | Last error code (0x00 = OK) |
| `0x03` | VERSION | u8 | Firmware version |
| `0x20` | AC_FREQ | u8 | Mains frequency (Hz), 0 if no zero-cross detected |
| `0xCE` | DATA_VALID | u8 | bit0 = valid result available since boot |
| `0xCA` | RT_period_ms | u32 LE | Actual wall-clock duration of the last RT window (ms, diagnostic) |

## Polling — single module

### Simplest 1 Hz polling loop

```cpp
void loop() {
  float u = rb_read_float_le(0x50, 0x86);
  float i = rb_read_float_le(0x50, 0x8E);
  float p = rb_read_float_le(0x50, 0xA6);
  float pf = rb_read_float_le(0x50, 0xB2);
  float q = rb_read_float_le(0x50, 0xD0);
  uint8_t freq = rb_read_u8(0x50, 0x20);
  const char* dir = (p >= 0.0f) ? "-> consume" : "<- export ";

  Serial.printf("U=%.1fV  I=%.3fA  P=%.1fW %s  PF=%.3f  Q=%.1fVAR  f=%dHz\n",
                u, i, p, dir, pf, q, freq);
  delay(1000);
}
```

### 200 ms polling without DRDY

```cpp
unsigned long last_read = 0;

void loop() {
  unsigned long now = millis();
  if (now - last_read >= 200) {
    last_read = now;
    float p = rb_read_float_le(0x50, 0xA6);
    Serial.println(p);
  }
}
```

This yields ~5 samples per second — the same rate at which the firmware refreshes.

## Multi-channel current (UI2 / UI3)

On UI2/UI3 variants the master reads currents and powers for all channels:

```cpp
// UI3: three current channels on one module
void read_ui3(uint8_t addr) {
  float u = rb_read_float_le(addr, 0x86);
  float i0 = rb_read_float_le(addr, 0x8E);
  float i1 = rb_read_float_le(addr, 0x92);
  float i2 = rb_read_float_le(addr, 0x96);
  float p0 = rb_read_float_le(addr, 0xA6);
  float p1 = rb_read_float_le(addr, 0xAA);
  float p2 = rb_read_float_le(addr, 0xAE);

  Serial.printf("U=%.1f  Ch0: %.3fA/%.1fW  Ch1: %.3fA/%.1fW  Ch2: %.3fA/%.1fW\n",
                u, i0, p0, i1, p1, i2, p2);
}
```

The current channels are **independent** — each has its own CT clamp with its own coefficient. A UI3 module can monitor, for example, the main feed, the water heater, and the lighting circuit simultaneously on the same phase. The sign of P can differ between channels: mains-input P > 0 (the house consumes from grid) while the solar-output channel shows P < 0 (the inverter exports while feeding the same phase that the house draws from).

### All channels share one voltage U

UI* variants contain a **single** voltage sensor — it measures the phase that powers the module. All channels' P values are computed against this **shared** U:

```
P0 = mean(u(t) · i0(t))
P1 = mean(u(t) · i1(t))
P2 = mean(u(t) · i2(t))
```

This assumes all CT clamps are on conductors of the **same phase**. For true multi-phase measurement, see the roadmap (rbAmp-U3I3).

## Multi-module polling — three strategies

### Strategy 1 — round-robin polling (simple)

The master reads the modules in turn, e.g. every 200 ms or every second:

```cpp
const uint8_t modules[] = {0x50, 0x51, 0x52};
const int N = 3;
int idx = 0;

void loop() {
  uint8_t addr = modules[idx];
  float p = rb_read_float_le(addr, 0xA6);
  Serial.printf("[0x%02X] P=%.1f W\n", addr, p);
  idx = (idx + 1) % N;
  delay(200);
}
```

**Pros**: minimal code, no extra GPIO required.
**Cons**: master CPU continuously busy; no precise "new window" timestamp.

### Strategy 2 — DRDY per module (precise, GPIO-heavy)

Each module's DRDY is wired to its own master GPIO, each with its own EXTI handler:

```cpp
const uint8_t modules[] = {0x50, 0x51, 0x52};
const uint8_t pins[]    = {15, 16, 17};
volatile bool ready[]   = {false, false, false};

void IRAM_ATTR drdy_0() { ready[0] = true; }
void IRAM_ATTR drdy_1() { ready[1] = true; }
void IRAM_ATTR drdy_2() { ready[2] = true; }

void setup() {
  Wire.begin();
  void (*isr_arr[])() = {drdy_0, drdy_1, drdy_2};
  for (int i = 0; i < 3; i++) {
    pinMode(pins[i], INPUT_PULLUP);
    attachInterrupt(digitalPinToInterrupt(pins[i]), isr_arr[i], FALLING);
  }
}

void loop() {
  for (int i = 0; i < 3; i++) {
    if (ready[i]) {
      ready[i] = false;
      float p = rb_read_float_le(modules[i], 0xA6);
      Serial.printf("[%d] P=%.1f W\n", i, p);
    }
  }
}
```

**Pros**: precise synchronisation, minimal read latency.
**Cons**: one GPIO + EXTI per module; does not scale beyond ~4 modules.

### Strategy 3 — wired-OR DRDY (compact)

All DRDY pins (open-drain) are joined on a single wire. The master detects the falling edge of **any** module and reads **all** modules:

```
rbAmp #1 DRDY ──┐
rbAmp #2 DRDY ──┼──┬─[10K]── +3.3V
rbAmp #3 DRDY ──┘  │
                   ▼
              Master GPIO IN (EXTI falling)
```

```cpp
const uint8_t modules[] = {0x50, 0x51, 0x52};
const uint8_t PIN_DRDY = 15;
volatile bool any_ready = false;

void IRAM_ATTR drdy() { any_ready = true; }

void setup() {
  Wire.begin();
  pinMode(PIN_DRDY, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(PIN_DRDY), drdy, FALLING);
}

void loop() {
  if (any_ready) {
    any_ready = false;
    for (int i = 0; i < 3; i++) {
      float p = rb_read_float_le(modules[i], 0xA6);
      Serial.printf("[0x%02X] P=%.1f W\n", modules[i], p);
    }
  }
}
```

**Pros**: one GPIO + EXTI for any number of modules; wire-economical.
**Cons**: the master does not know which module asserted the line — it must read all of them. Slightly-offset module periods (each module has its own crystal) produce "double" triggers (one module ready, another a millisecond later). Harmless — just read all modules.

### Trade-off table

| Approach | Master GPIO | Latency | Complexity | Scalability |
|---|:---:|:---:|:---:|:---:|
| Polling without DRDY | 0 | ~200 ms | minimal | up to ~16 modules |
| DRDY per module | N | < 1 ms | medium | up to ~4 modules |
| Wired-OR DRDY | 1 | ~1–5 ms | low | up to ~10 modules |

### Recommendation

- **1–3 modules**, low latency required → DRDY per module
- **3–10 modules**, GPIO budget matters → wired-OR DRDY
- **Any number**, latency does not matter → simple polling

## I-only variants (no voltage sensor)

On I1/I2/I3 (and higher-channel I*-only variants), without a voltage sensor:

| Register | Returns |
|---|---|
| `U_rms` (`0x86`) | `0.0f` |
| `U_peak` (`0x8A`) | `0.0f` |
| `I*_rms`, `I*_peak` | Real values |
| `P*_real` (`0xA6`, `0xAA`, `0xAE`) | `0.0f` (no U → no P) |
| `PF*`, `Q*` | `0.0f` |
| `AC_FREQ` (`0x20`) | `0` (no U → no zero-cross detection) |

Without P there is no information about current direction — `I_rms` is always ≥ 0 and does not show which way current flows.

I-only variants are intended for:

- Masters that provide their own voltage source and compute P themselves (e.g. an ESP32 with a built-in ZMPT module plus an rbAmp-I3 for three loads)
- Current-only applications: load presence detection, overcurrent protection, balancing of resistive heating elements

## RT register quick reference

```
0x00 STATUS (u8)         — bit0=ready, bit1=error
0x02 ERROR (u8)          — last error code
0x03 VERSION (u8)        — firmware version
0x05 CT_MODEL (u8, RW)   — for modules with plug-in CT
0x20 AC_FREQ (u8)        — mains frequency, Hz
0x30 I2C_ADDRESS (u8 RW) — slave address (change requires SAVE_GAINS + RESET)

0x86 U_rms (f32)         — V
0x8A U_peak (f32)        — V
0x8E I0_rms (f32)        — A
0x92 I1_rms (f32)        — A
0x96 I2_rms (f32)        — A
0x9A I0_peak (f32)       — A
0x9E I1_peak (f32)       — A
0xA2 I2_peak (f32)       — A
0xA6 P0_real (f32, ±)    — W  (sign: + consume, − export)
0xAA P1_real (f32, ±)    — W
0xAE P2_real (f32, ±)    — W
0xB2 PF0 (f32, ±)        — −1..+1
0xB6 PF1 (f32, ±)        — −1..+1
0xBA PF2 (f32, ±)        — −1..+1

0xCA RT_period_ms (u32)  — diagnostic, actual window duration
0xCE DATA_VALID (u8)     — bit0 = valid after boot

0xD0 Q0_reac (f32, ±)    — VAR (sign: + inductive, − capacitive)
0xD4 Q1_reac (f32, ±)    — VAR
0xD8 Q2_reac (f32, ±)    — VAR
```

## Next

- [Period metering](period-metering.md) — atomic period latch for tariff accumulation
- [Troubleshooting](troubleshooting.md) — what to do if values look wrong

## See also

- [Initialization](initialization.md) — bring-up procedure and helper functions used in these examples
- [Hardware connection](hardware-connection.md) — DRDY pull-up requirement


---

[← Initialization](initialization.md) | [Contents](../README.md) | [Period Metering →](period-metering.md)
