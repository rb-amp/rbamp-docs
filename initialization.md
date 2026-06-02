# Initialization and Start-up

rbAmp starts measuring **immediately** after power-on. The master only needs to:

1. Bring up its I²C bus (standard `Wire` / HAL initialization).
2. Wait ~250 ms, or poll `DATA_VALID = 1` at register `0xCE`.
3. Optionally read the firmware version from `0x03` as a sanity check.
4. Read data — ready.

No start command, no register-level configuration on the chip is required. Factory calibration is loaded automatically on boot, so the module emits correct physical-unit values from the first reading.

> **Important for modules with an external (plug-in) CT**: after first installation, you must **write the sensor model** to the model register once. The module needs the model to apply the correct mV-per-ampere coefficient. The value is stored in flash and persists across reboots — there is no need to repeat the write on every boot. Details are in the section [Setting the external sensor model](#setting-the-external-sensor-model) below.

## Helper functions (platform-neutral C)

All examples below use three small helpers that are easy to implement on any platform:

```c
// Read one byte from a slave register.
uint8_t rb_read_u8(uint8_t addr, uint8_t reg);

// Read a little-endian float32 (4 bytes from reg, reg+1, reg+2, reg+3).
// rbAmp does NOT auto-increment its pointer, so each byte requires
// its own transaction with an explicit register address.
float rb_read_float_le(uint8_t addr, uint8_t reg);

// Write one byte to a slave register.
void rb_write_u8(uint8_t addr, uint8_t reg, uint8_t val);
```

## Bring-up on Arduino (`Wire.h`)

### Helpers

```cpp
#include <Wire.h>

uint8_t rb_read_u8(uint8_t addr, uint8_t reg) {
  Wire.beginTransmission(addr);
  Wire.write(reg);
  Wire.endTransmission(false);   // repeated start, do not release bus
  Wire.requestFrom(addr, (uint8_t)1);
  return Wire.read();
}

float rb_read_float_le(uint8_t addr, uint8_t reg) {
  uint8_t buf[4];
  for (int i = 0; i < 4; i++) {
    buf[i] = rb_read_u8(addr, reg + i);
  }
  float f;
  memcpy(&f, buf, 4);            // little-endian host (AVR, ESP32, ARM)
  return f;
}

void rb_write_u8(uint8_t addr, uint8_t reg, uint8_t val) {
  Wire.beginTransmission(addr);
  Wire.write(reg);
  Wire.write(val);
  Wire.endTransmission();
}
```

### Smoke test (single module)

```cpp
#include <Wire.h>
#define RB_ADDR 0x50

void setup() {
  Serial.begin(115200);
  Wire.begin();
  Wire.setClock(100000);
  delay(300);                           // wait for module boot

  // Wait DATA_VALID bit
  while ((rb_read_u8(RB_ADDR, 0xCE) & 0x01) == 0) {
    Serial.println("waiting for DATA_VALID...");
    delay(50);
  }

  uint8_t ver = rb_read_u8(RB_ADDR, 0x03);
  Serial.printf("rbAmp ready, version 0x%02X\n", ver);
}

void loop() {
  float u_rms = rb_read_float_le(RB_ADDR, 0x86);
  float i_rms = rb_read_float_le(RB_ADDR, 0x8E);
  float p     = rb_read_float_le(RB_ADDR, 0xA6);
  float pf    = rb_read_float_le(RB_ADDR, 0xB2);

  Serial.printf("U=%.1f V  I=%.3f A  P=%.1f W  PF=%.3f\n",
                u_rms, i_rms, p, pf);
  delay(1000);
}
```

Expected output (a 60 W incandescent lamp on 230 V):

```
rbAmp ready, version 0x01
U=229.8 V  I=0.262 A  P=60.2 W  PF=0.998
U=230.1 V  I=0.262 A  P=60.3 W  PF=0.998
...
```

## Bring-up on STM32 HAL (C)

```c
#include "stm32f1xx_hal.h"
extern I2C_HandleTypeDef hi2c1;

#define RB_ADDR_8BIT (0x50 << 1)        // HAL expects 8-bit address format

uint8_t rb_read_u8(uint8_t reg) {
  uint8_t v;
  HAL_I2C_Mem_Read(&hi2c1, RB_ADDR_8BIT, reg, I2C_MEMADD_SIZE_8BIT, &v, 1, 100);
  return v;
}

float rb_read_float_le(uint8_t reg) {
  uint8_t buf[4];
  for (int i = 0; i < 4; i++) {
    HAL_I2C_Mem_Read(&hi2c1, RB_ADDR_8BIT, reg + i, I2C_MEMADD_SIZE_8BIT,
                     &buf[i], 1, 100);
  }
  float f;
  memcpy(&f, buf, 4);
  return f;
}

void main_app(void) {
  while ((rb_read_u8(0xCE) & 0x01) == 0) HAL_Delay(50);

  uint8_t ver = rb_read_u8(0x03);
  printf("rbAmp version: 0x%02X\n", ver);

  while (1) {
    float u = rb_read_float_le(0x86);
    float i = rb_read_float_le(0x8E);
    float p = rb_read_float_le(0xA6);
    printf("U=%.1f V  I=%.3f A  P=%.1f W\n", u, i, p);
    HAL_Delay(1000);
  }
}
```

## Setting the external sensor model

> Required **only** for SKUs with a plug-in CT (3.5 mm jack). Modules with built-in sensors are already configured at the factory — skip this section.

The firmware needs to know **which** sensor is connected in order to apply the correct mV-per-ampere coefficient. Writing the model code once is sufficient — the value is saved to flash and is restored on every reboot.

### Model register

| Name | Address | Type | Value |
|---|---|---|---|
| `REG_CT_MODEL` | `0x05` | u8 RW | CT model code |

CT model codes (commonly used SCT-013 series and a generic CT entry; the table is extended in newer firmware):

| Code | Sensor | Rated current | Sensitivity |
|:---:|---|:---:|---|
| `0x00` | not set (default on factory-fresh plug-in-CT SKUs) | — | — |
| `0x01` | SCT-013-005 | 5 A | 200 mV/A |
| `0x02` | SCT-013-010 | 10 A | 100 mV/A |
| `0x03` | SCT-013-030 | 30 A | 33 mV/A |
| `0x04` | SCT-013-050 | 50 A | 20 mV/A |
| `0x05` | SCT-013-100 | 100 A | 10 mV/A |
| `0x06` | CT-005A (generic) | 5 A | 10 mV/A |

### Procedure

1. Plug the CT clamp into the module's jack (arrow toward the load — see [Hardware connection](hardware-connection.md)).
2. Write the model code to `REG_CT_MODEL = 0x05`:
   ```cpp
   rb_write_u8(0x50, 0x05, 0x03);    // example: SCT-013-030
   ```
3. Write `CMD_SAVE_GAINS = 0x26` to `REG_COMMAND = 0x01`:
   ```cpp
   rb_write_u8(0x50, 0x01, 0x26);
   ```
4. Wait **≥ 700 ms** for flash erase/write:
   ```cpp
   delay(700);
   ```
5. Done. The value persists across reboots; subsequent power-ons do not require this step.

### Verification

```cpp
uint8_t model = rb_read_u8(0x50, 0x05);
Serial.printf("CT model = 0x%02X\n", model);
```

If the readback equals the value you wrote, the configuration is saved.

> When commissioning a new module, perform a sanity check with a known resistive load (for example a 100 W incandescent lamp): `I_rms ≈ 100 W / U_rms ≈ 0.43 A` and `P_real ≈ 100 W`. If readings are off by a large factor, the model code is probably wrong.

## Single-module workflow

1. Wire the module according to [Hardware connection](hardware-connection.md).
2. For SKUs with a plug-in CT, write the sensor model once (previous section).
3. Flash the master with the smoke-test sketch.
4. Open the serial monitor — you should see U/I/P every second.
5. If readings look wrong, see [Troubleshooting](troubleshooting.md).

## Multi-module setup

Scenario: three rbAmp modules on a single bus to monitor different sections of a house (main feed, water heater, air conditioner).

### Step 1 — Addresses

Out of the box all three modules carry address `0x50`. They must be readdressed one at a time. **Connect only one module to the bus until readdressing is complete.**

### Step 2 — Smoke test with one module

Use the section above. Confirm one module works correctly.

### Step 3 — Readdress (and, if needed, set the CT model)

Use the address-change procedure below. If the modules have different plug-in CTs, set the CT model on each one before readdressing.

### Step 4 — Multi-module polling loop

```cpp
#include <Wire.h>

const uint8_t modules[] = {0x50, 0x51, 0x52};   // main, boiler, AC
const char* labels[]    = {"MAIN", "BOIL", "AC  "};
const int N = sizeof(modules) / sizeof(modules[0]);   // element count, not byte count

void setup() {
  Serial.begin(115200);
  Wire.begin();
  Wire.setClock(100000);
  delay(300);

  for (int i = 0; i < N; i++) {
    Wire.beginTransmission(modules[i]);
    uint8_t rc = Wire.endTransmission();
    Serial.printf("Module %s (0x%02X): %s\n",
                  labels[i], modules[i], rc == 0 ? "OK" : "MISSING");
  }
}

void loop() {
  for (int i = 0; i < N; i++) {
    float u = rb_read_float_le(modules[i], 0x86);
    float p = rb_read_float_le(modules[i], 0xA6);
    Serial.printf("[%s] U=%.1f  P=%.1f W\n", labels[i], u, p);
  }
  Serial.println();
  delay(1000);
}
```

Expected output:

```
Module MAIN (0x50): OK
Module BOIL (0x51): OK
Module AC   (0x52): OK
[MAIN] U=229.8  P=850.3 W
[BOIL] U=229.7  P=1850.0 W
[AC  ] U=229.9  P=0.0 W
...
```

## Address change

The factory address `0x50` is stored in the flash parameter block. To change it:

### Registers and commands

| Name | Address | Type |
|---|---|---|
| `REG_I2C_ADDRESS` | `0x30` | u8 RW |
| `CMD_SAVE_GAINS` | `0x26` written to `REG_COMMAND` (`0x01`) | command |
| `CMD_RESET` | `0x01` written to `REG_COMMAND` (`0x01`) | command |

### Procedure

1. **Connect only one module** (still on `0x50`) to the bus.
2. Write the new address to `0x30`:
   ```cpp
   rb_write_u8(0x50, 0x30, 0x51);   // new address 0x51
   ```
3. Write `CMD_SAVE_GAINS = 0x26` to `REG_COMMAND = 0x01`:
   ```cpp
   rb_write_u8(0x50, 0x01, 0x26);
   ```
4. Wait **at least 700 ms** for the flash erase/write to complete:
   ```cpp
   delay(700);
   ```
5. Reset the module:
   - software reset: `rb_write_u8(0x50, 0x01, 0x01);`
   - or power-cycle.
6. Wait ~300 ms for boot.
7. Verify the new address with a presence probe (do NOT use a register-read for verification — `rb_read_u8()` returns `0xFF` on NACK, which a naïve `!= 0` check passes incorrectly):
   ```cpp
   Wire.beginTransmission(0x51);
   if (Wire.endTransmission() == 0) {
     Serial.println("Address change OK");
   } else {
     Serial.println("Address change FAILED — module not responding at new address");
   }
   ```
8. Mark the PCB with the new address using a wax pencil or label so you can identify the module physically.

### Important notes

- `CMD_SAVE_GAINS` saves the **entire flash parameter block**: I²C address, calibration gains, noise floor, CT model, VS configuration. Issue it only after the parameter change you actually intend — it rewrites the whole block.
- The new address must lie in the range `0x08..0x77` (valid 7-bit I²C). Out-of-range writes are rejected; the module keeps its current address.
- Reserved addresses: `0x00` (general call — see below), `0x01..0x07` (CBus reserved), `0x78..0x7F` (10-bit reserved). Do not use these.

### Full address-change example

```cpp
#include <Wire.h>

bool change_address(uint8_t old_addr, uint8_t new_addr) {
  if (new_addr < 0x08 || new_addr > 0x77) {
    Serial.println("ERROR: invalid address");
    return false;
  }

  Serial.printf("Changing 0x%02X -> 0x%02X\n", old_addr, new_addr);

  rb_write_u8(old_addr, 0x30, new_addr);
  delay(10);
  rb_write_u8(old_addr, 0x01, 0x26);    // CMD_SAVE_GAINS
  delay(700);                            // flash erase/write
  rb_write_u8(old_addr, 0x01, 0x01);    // CMD_RESET
  delay(300);                            // boot

  Wire.beginTransmission(new_addr);
  if (Wire.endTransmission() == 0) {
    Serial.printf("OK, module now at 0x%02X\n", new_addr);
    return true;
  }
  Serial.printf("FAIL: module not found at 0x%02X\n", new_addr);
  return false;
}

void setup() {
  Serial.begin(115200);
  Wire.begin();
  delay(300);
  change_address(0x50, 0x51);
}
```

## General Call (broadcast at address 0x00)

The I²C protocol reserves address **`0x00` (general call)** — packets sent to this address are received by **all** slaves on the bus simultaneously. rbAmp listens to general-call frames and recognises a limited command set:

- `CMD_LATCH_PERIOD = 0x27` — synchronous period latch on **all** modules on the bus (see [Period metering](period-metering.md))
- `CMD_RESET = 0x01` — synchronous software reset of all modules

Other commands whose meaning is address-specific (for example `CMD_SAVE_GAINS`) are ignored on general call.

> Available from firmware release 1.0 onward.

Broadcast LATCH example:

```cpp
// Single write to address 0x00 — all modules latch their snapshot at once
Wire.beginTransmission(0x00);
Wire.write(0x01);     // REG_COMMAND
Wire.write(0x27);     // CMD_LATCH_PERIOD
Wire.endTransmission();
```

After the broadcast LATCH each module updates its snapshot independently. The master then reads each module by its individual address.

Application: precise period synchronisation for multi-module energy balancing (for example `mains_input − solar_output`), where the latch instant must be identical across modules.

## Recommended multi-module deployment workflow

```
Step 1: Unbox all modules — every module is on address 0x50.

Step 2: For each module, one at a time:
   a) Connect only this module to the bus.
   b) If the SKU has a plug-in CT: write the CT model (0x05) and SAVE_GAINS.
   c) Run change_address(0x50, <new_address>).
   d) Verify the module responds at the new address.
   e) Physically mark the module (label with 0x51, 0x52, ...).
   f) Disconnect and move to the next module.

Step 3: After readdressing all modules, connect them in parallel.
        If more than one module is on the bus, cut the built-in pull-up
        jumpers on all but one (see Hardware connection → I²C pull-ups).

Step 4: At master boot, run a scan loop and verify every readdressed
        module responds at its expected address.
```

## Module variant auto-detection

The master can detect the variant (UI1 / UI2 / UI3 / I*) by probing register `0xC2` (period-averaged power for channel I1):

```cpp
bool has_channel_1(uint8_t addr) {
  Wire.beginTransmission(addr);
  Wire.write(0xC2);
  Wire.endTransmission(false);
  uint8_t rc = Wire.requestFrom(addr, (uint8_t)1);
  return (rc == 1);   // NACK means register is not available (UI1)
}
```

Alternative: reading `0x8E` (I0_rms) returns a real current value on both UI* and I*; reading `0x86` (U_rms) returns 0 on I-only variants — useful for detecting the presence of a voltage sensor.

## Next

- [Real-time polling](realtime-polling.md) — real-time polling in depth (200 ms window, DRDY)
- [Period metering](period-metering.md) — tariff energy accumulation


---

[← Hardware Connection](hardware-connection.md) | [Contents](README.md) | [Real-time Polling →](realtime-polling.md)
