# 14 · STM32 HAL Examples

This chapter is the STM32 equivalent of [10_arduino_examples.md](arduino-examples.md), [12_micropython_examples.md](micropython-examples.md) and [13_esp_idf_examples.md](esp-idf-examples.md): the same ten scenarios, ported to **STM32 HAL** (CubeMX-generated C) with optional FreeRTOS / CMSIS-OS2.

> **These examples talk to rbAmp directly through the raw I2C register API.** They are intended for production embedded firmware on STM32 — direct control over the HAL, predictable RAM/flash footprint, integration with LwIP, FATFS, FreeRTOS and CubeMX HAL generation.
>
> **For a more convenient, structured interface — use the `rbamp` STM32 HAL library** (chapter 21 *coming soon* — pending closure of the STM32 bench validation session; see [`baton__stm32__to__root.md`](https://rbamp.com/docs/modules-basic-standard-api-reference) for status). The library will wrap the register-level details behind a single `RbAmp_HandleTypeDef` with named functions (`RbAmp_ReadVoltage()`, `RbAmp_LatchPeriod()`, `RbAmp_ReadPeriodSnapshot()`, etc.), a per-channel Wh accumulator, a multi-module manager, and bare-HAL ergonomics that need no SPEC §B.5 retry layer (STM32 HAL does not share the ESP-IDF v5 NACK pattern). Until chapter 21 lands, the raw-API recipes below are the recommended path on STM32.

## Supported families and toolchain

| Family | Validated | I2C peripheral | Notes |
|---|:---:|---|---|
| STM32F1 family | ✓ | I2C1, I2C2 | Common DIY target |
| STM32F4 (Nucleo-F411, Discovery) | ✓ | I2C1..3 | High clock, plenty of RAM |
| STM32L4 / L4+ | ✓ | I2C1..4 | Low-power, RTC backup registers, STOP / STANDBY |
| STM32G0 / G4 | ✓ | I2C1..3 | Modern peripherals |
| STM32H7 (Nucleo-H743) | ✓ | I2C1..4 | Built-in Ethernet PHY → MQTT directly via LwIP |

Toolchain: **CubeMX 6.x** + **CubeIDE** or a Makefile project (`arm-none-eabi-gcc`). Examples assume HAL version that matches your CubeMX-generated project. The new I2C HAL signature (`HAL_I2C_Mem_Read/Write`) is supported by every modern family.

## Networking note

Most STM32 chips have no built-in Wi-Fi. The MQTT-dependent examples (4, 6, 8, 9) assume **one** of the following network paths:

- **Ethernet** — STM32F4 / F7 / H7 boards with on-board PHY (LAN8742, DP83848) running LwIP + a small MQTT library (Paho Embedded C, MQTT-C, or an HAL-friendly fork). This is the smoothest path.
- **W5500 SPI Ethernet** — for STM32F1 / L4 / G4 (no on-chip MAC). Uses the WIZnet ioLibrary.
- **ESP32-AT via UART** — an ESP-AT module acts as a Wi-Fi bridge. Issue AT commands from the STM32 main loop.

The examples below show the **rbAmp logic in full detail**; the network bring-up is abstracted behind two stub functions you implement in your project:

```c
extern HAL_StatusTypeDef Net_Init(void);                  // bring up the chosen stack
extern HAL_StatusTypeDef Net_MqttPublish(const char *topic,
                                          const uint8_t *payload, uint32_t len,
                                          uint8_t qos, uint8_t retain);
```

Reference implementations for each network path are out of scope for this chapter; pick the one that fits your hardware and replace the stubs.

## Project layout

CubeMX-generated project shape:

```
example_NN/
├── example_NN.ioc                  ← CubeMX project (pins, peripherals, clock)
├── Makefile                        ← (or *.eww / *.uvprojx for IAR / STM32CubeIDE)
├── Core/
│   ├── Inc/main.h
│   ├── Inc/rbamp_io.h              ← common helpers
│   ├── Src/main.c                  ← USER CODE inside CubeMX markers
│   ├── Src/rbamp_io.c
│   └── Src/stm32xxxx_it.c          ← EXTI handlers (CubeMX-generated)
├── Drivers/
│   └── STM32xxxx_HAL_Driver/       ← HAL sources
└── Middlewares/                    ← FreeRTOS, LwIP, FATFS (optional)
```

CubeMX pin setup used in every example unless noted:

- **I2C1**: SCL = PB6, SDA = PB7 (typical STM32F1 / Nucleo-F411 defaults)
- **UART1**: TX = PA9, RX = PA10 — for `printf` debug
- **EXTI**: PA0 = DRDY (falling edge, pull-up)
- I2C speed: 100 kHz

## Examples table of contents

| # | Title | Difficulty | Output | MQTT | DRDY | Multi-module | Bidirectional | Use case |
|:---:|---|:---:|:---:|:---:|:---:|:---:|:---:|---|
| 1 | [Quick read](#example-1--quick-read-minimal) | minimal | UART printf | — | — | — | — | Smoke test |
| 2 | [60-second energy meter on OLED](#example-2--60-second-energy-meter-on-oled) | low | SSD1306 | — | — | — | — | Boxed Wh counter |
| 3 | [Multi-module monitor](#example-3--multi-module-monitor) | low | UART printf | — | — | yes (3) | — | Whole-home monitoring |
| 4 | [Per-appliance energy tracker (UI3)](#example-4--per-appliance-energy-tracker-ui3) | medium | — | yes | — | — | — | Sub-metering in HA |
| 5 | [Master-side bidirectional on a BASIC module](#example-5--master-side-bidirectional-on-a-basic-module) | medium | UART printf | — | yes | — | yes (master) | Solar home on BASIC tier |
| 6 | [Home energy balance](#example-6--home-energy-balance) | high | — | yes | — | yes (3) | yes | Full home balance |
| 7 | [Power-event detection](#example-7--power-event-detection) | medium | UART + SD | — | yes | — | — | Appliance event log |
| 8 | [MQTT publisher with HA Auto-discovery](#example-8--mqtt-publisher-with-home-assistant-auto-discovery) | medium | — | yes (+disco) | — | — | optional | Drop-in HA integration |
| 9 | [Battery-powered logger with STANDBY mode](#example-9--battery-powered-logger-with-standby-mode) | medium | — | yes | — | — | — | Off-grid / outdoor meter |
| 10 | [Time-of-use (TOU) tariff with SNTP wall-clock](#example-10--time-of-use-tou-tariff-with-sntp-wall-clock) | medium | UART printf | yes | — | — | optional | Peak / off-peak tariff metering |

---

## Common helpers (`rbamp_io.h` / `rbamp_io.c`)

Place these files in `Core/Inc` and `Core/Src` of your CubeMX project. To save space, the `#include "rbamp_io.h"` line is omitted in each example below.

### `rbamp_io.h`

```c
/**
 * @file rbamp_io.h
 * @brief Raw I2C helpers for the rbAmp slave (STM32 HAL).
 */
#ifndef RBAMP_IO_H
#define RBAMP_IO_H

#include <stdint.h>
#include <stdbool.h>
#include "main.h"        /* CubeMX-generated, brings the HAL header */

#ifdef __cplusplus
extern "C" {
#endif

/* I2C transaction timeout in milliseconds (HAL_GetTick units). */
#define RBAMP_I2C_TIMEOUT_MS  50U

/**
 * @brief  Bind the helpers to a HAL I2C handle.
 * @details Stores the pointer internally; call once at startup. The handle
 *          must have been initialised by HAL_I2C_Init() (typically by CubeMX).
 */
void RbAmpIO_Bind(I2C_HandleTypeDef *hi2c);

/**
 * @brief  Read one byte from a register of an rbAmp slave.
 * @param  addr  7-bit slave address (e.g. 0x50).
 * @param  reg   Register address inside the slave.
 * @param  out   Output byte.
 * @return HAL_OK on success.
 */
HAL_StatusTypeDef RbAmpIO_ReadU8(uint8_t addr, uint8_t reg, uint8_t *out);

/**
 * @brief  Read a little-endian float32 from four consecutive registers.
 * @details Reads four consecutive bytes. rbAmp supports READ auto-increment
 *          (a single burst-read is valid for atomicity). The per-byte form
 *          is shown for clarity. See chapter 11 for asymmetry rules.
 */
HAL_StatusTypeDef RbAmpIO_ReadFloatLE(uint8_t addr, uint8_t reg, float *out);

/**
 * @brief  Read a little-endian uint32 from four consecutive registers.
 */
HAL_StatusTypeDef RbAmpIO_ReadU32LE(uint8_t addr, uint8_t reg, uint32_t *out);

/**
 * @brief  Write a single byte to a register.
 */
HAL_StatusTypeDef RbAmpIO_WriteU8(uint8_t addr, uint8_t reg, uint8_t val);

/**
 * @brief  Broadcast CMD_LATCH_PERIOD to every rbAmp with GC reception
 *         enabled (FLEET_CONFIG.bit0 = 1; opt-in, default OFF).
 * @details Canonical 5-byte general-call frame: A5 27 group tick_lo tick_hi.
 *          Slaves reject any first byte != 0xA5. See chapter 11 §6.3.2.
 * @param  tick16  Caller's 16-bit window counter (mirrored at GC_TICK 0x59).
 * @param  group   GROUP_ID filter (0 = all-call).
 * @return HAL_OK on ACK; HAL_ERROR on NACK — fall back to per-module latch.
 */
HAL_StatusTypeDef RbAmpIO_BroadcastLatch(uint16_t tick16, uint8_t group);

/**
 * @brief  Probe whether a slave responds at the given address (1-byte read).
 * @return true if the slave acks, false otherwise.
 */
bool RbAmpIO_Probe(uint8_t addr);

/**
 * @brief  Block until DATA_VALID bit 0 of register 0xCE reads 1, or timeout.
 * @return HAL_OK if ready in time, HAL_TIMEOUT otherwise.
 */
HAL_StatusTypeDef RbAmpIO_WaitReady(uint8_t addr, uint32_t timeout_ms);

/* ===== rbAmp register addresses (subset used in examples) ===== */
#define RB_REG_STATUS           0x00U
#define RB_REG_COMMAND          0x01U
#define RB_REG_VERSION          0x03U
#define RB_REG_AC_FREQ          0x20U
#define RB_REG_PERIOD_VALID     0x07U
#define RB_REG_U_RMS            0x86U
#define RB_REG_I0_RMS           0x8EU
#define RB_REG_I1_RMS           0x92U
#define RB_REG_I2_RMS           0x96U
#define RB_REG_P0_REAL          0xA6U
#define RB_REG_PF0              0xB2U
#define RB_REG_Q0               0xD0U
#define RB_REG_DATA_VALID       0xCEU
#define RB_REG_PERIOD_AVG_P0    0xDCU
#define RB_REG_PERIOD_AVG_P1    0xC2U
#define RB_REG_PERIOD_AVG_P2    0xC6U
#define RB_REG_PERIOD_MAX_P    0xE0U
#define RB_REG_PERIOD_LATCH_MS  0xECU

/* ===== Commands ===== */
#define RB_CMD_RESET            0x01U
#define RB_CMD_SAVE_GAINS       0x26U
#define RB_CMD_LATCH_PERIOD     0x27U

#ifdef __cplusplus
}
#endif

#endif /* RBAMP_IO_H */
```

### `rbamp_io.c`

```c
#include "rbamp_io.h"
#include <string.h>

/* HAL I2C uses the 8-bit address format: left-shift the 7-bit address. */
#define ADDR8(addr7)  ((uint16_t)((addr7) << 1))

static I2C_HandleTypeDef *s_hi2c = NULL;

void RbAmpIO_Bind(I2C_HandleTypeDef *hi2c) {
    s_hi2c = hi2c;
}

HAL_StatusTypeDef RbAmpIO_ReadU8(uint8_t addr, uint8_t reg, uint8_t *out) {
    return HAL_I2C_Mem_Read(s_hi2c, ADDR8(addr), reg,
                            I2C_MEMADD_SIZE_8BIT, out, 1,
                            RBAMP_I2C_TIMEOUT_MS);
}

HAL_StatusTypeDef RbAmpIO_ReadFloatLE(uint8_t addr, uint8_t reg, float *out) {
    uint8_t buf[4];
    for (int i = 0; i < 4; i++) {
        HAL_StatusTypeDef st = HAL_I2C_Mem_Read(s_hi2c, ADDR8(addr), reg + i,
                                                 I2C_MEMADD_SIZE_8BIT,
                                                 &buf[i], 1,
                                                 RBAMP_I2C_TIMEOUT_MS);
        if (st != HAL_OK) return st;
    }
    memcpy(out, buf, sizeof(float));   /* STM32 is little-endian — host order matches */
    return HAL_OK;
}

HAL_StatusTypeDef RbAmpIO_ReadU32LE(uint8_t addr, uint8_t reg, uint32_t *out) {
    uint8_t buf[4];
    for (int i = 0; i < 4; i++) {
        HAL_StatusTypeDef st = HAL_I2C_Mem_Read(s_hi2c, ADDR8(addr), reg + i,
                                                 I2C_MEMADD_SIZE_8BIT,
                                                 &buf[i], 1,
                                                 RBAMP_I2C_TIMEOUT_MS);
        if (st != HAL_OK) return st;
    }
    *out = (uint32_t)buf[0] | (uint32_t)buf[1] << 8
         | (uint32_t)buf[2] << 16 | (uint32_t)buf[3] << 24;
    return HAL_OK;
}

HAL_StatusTypeDef RbAmpIO_WriteU8(uint8_t addr, uint8_t reg, uint8_t val) {
    return HAL_I2C_Mem_Write(s_hi2c, ADDR8(addr), reg,
                              I2C_MEMADD_SIZE_8BIT, &val, 1,
                              RBAMP_I2C_TIMEOUT_MS);
}

HAL_StatusTypeDef RbAmpIO_BroadcastLatch(uint16_t tick16, uint8_t group) {
    /* Canonical 5-byte general-call frame: A5 27 group tick_lo tick_hi.
     * HAL_I2C_Master_Transmit accepts the 8-bit address; 0x00 << 1 = 0x00. */
    uint8_t tx[5] = {
        0xA5,                              /* frame magic */
        RB_CMD_LATCH_PERIOD,               /* opcode 0x27 */
        group,                             /* group_id (0 = all-call) */
        (uint8_t)(tick16 & 0xFF),
        (uint8_t)((tick16 >> 8) & 0xFF),
    };
    return HAL_I2C_Master_Transmit(s_hi2c, ADDR8(0x00), tx, sizeof tx,
                                    RBAMP_I2C_TIMEOUT_MS);
}

bool RbAmpIO_Probe(uint8_t addr) {
    /* HAL_I2C_IsDeviceReady performs an address-only NACK probe. */
    return HAL_I2C_IsDeviceReady(s_hi2c, ADDR8(addr), /*trials=*/2,
                                  RBAMP_I2C_TIMEOUT_MS) == HAL_OK;
}

HAL_StatusTypeDef RbAmpIO_WaitReady(uint8_t addr, uint32_t timeout_ms) {
    uint32_t deadline = HAL_GetTick() + timeout_ms;
    while (HAL_GetTick() < deadline) {
        uint8_t v = 0;
        if (RbAmpIO_ReadU8(addr, RB_REG_DATA_VALID, &v) == HAL_OK && (v & 0x01U)) {
            return HAL_OK;
        }
        HAL_Delay(50);
    }
    return HAL_TIMEOUT;
}
```

### `printf` redirection

All examples call `printf()` over UART. In your `main.c` add the canonical redirection helper (works with newlib-nano):

```c
extern UART_HandleTypeDef huart1;

/**
 * @brief  Standard newlib `_write` retarget — sends bytes to UART1.
 * @details A bounded timeout (100 ms) prevents a stuck/unplugged UART from
 *          starving the IWDG (which `HAL_MAX_DELAY` would do). On timeout the
 *          chunk is dropped — preferable to a reset.
 */
int _write(int file, char *ptr, int len) {
    if (HAL_UART_Transmit(&huart1, (uint8_t*)ptr, len, 100) != HAL_OK) {
        return -1;                              /* drop chunk, do NOT block */
    }
    return len;
}
```

For Cube IDE projects with semihosting disabled and newlib-nano selected, `printf("%.1f", ...)` requires `-u _printf_float` in linker flags.

---

## Example 1 — Quick read (minimal)

**Goal**: print U, I, P, PF to UART once per second.
**Hardware**: any STM32 + one rbAmp UI1 + USB-UART for the host console.

In CubeMX:
- enable `I2C1` (Standard, 100 kHz) on PB6/PB7
- enable `USART1` (115200 8N1) on PA9/PA10
- everything else default

`main.c` user code — the `/* USER CODE BEGIN 2 */` and `/* USER CODE BEGIN WHILE */` blocks shown below sit **inside the CubeMX-generated `main()` function**, between its initialisation calls and the `while(1)` loop. **Do not paste them at file scope** — they must live inside `main()`.

Reference shape (the surrounding `int main(void) { … }` is CubeMX-generated and shown here for context only):

```c
/* USER CODE BEGIN Includes */
#include <stdio.h>
#include "rbamp_io.h"
/* USER CODE END Includes */

extern I2C_HandleTypeDef hi2c1;
extern UART_HandleTypeDef huart1;

int _write(int file, char *ptr, int len) {
    if (HAL_UART_Transmit(&huart1, (uint8_t*)ptr, len, 100) != HAL_OK) return -1;
    return len;
}

#define RB_ADDR  0x50U

int main(void)                  /* generated by CubeMX */
{
    HAL_Init();                 /* generated by CubeMX */
    SystemClock_Config();       /* generated by CubeMX */
    MX_GPIO_Init();             /* generated */
    MX_I2C1_Init();             /* generated */
    MX_USART1_UART_Init();      /* generated */

    /* USER CODE BEGIN 2 */
    RbAmpIO_Bind(&hi2c1);

    /* Wait until the first RT window has been computed. */
    if (RbAmpIO_WaitReady(RB_ADDR, 2000) != HAL_OK) {
        printf("ERROR: rbAmp not ready\r\n");
        Error_Handler();
    }

    uint8_t ver = 0;
    RbAmpIO_ReadU8(RB_ADDR, RB_REG_VERSION, &ver);
    printf("rbAmp version: 0x%02X\r\n", ver);
    /* USER CODE END 2 */

    /* USER CODE BEGIN WHILE */
    while (1) {
        float u, i_, p, pf;
        RbAmpIO_ReadFloatLE(RB_ADDR, RB_REG_U_RMS,   &u );
        RbAmpIO_ReadFloatLE(RB_ADDR, RB_REG_I0_RMS,  &i_);
        RbAmpIO_ReadFloatLE(RB_ADDR, RB_REG_P0_REAL, &p );
        RbAmpIO_ReadFloatLE(RB_ADDR, RB_REG_PF0,     &pf);
        printf("U=%.1fV  I=%.3fA  P=%+.1fW  PF=%+.3f\r\n", u, i_, p, pf);
        HAL_Delay(1000);
    /* USER CODE END WHILE */
    }
}
```

> **Convention used in this chapter**: Examples 2–10 omit the surrounding `main(void)` and CubeMX init calls for brevity, showing only the `/* USER CODE BEGIN 2 */` / `/* USER CODE BEGIN WHILE */` blocks. Every snippet starting with `RbAmpIO_Bind(&hi2c1);` is the **contents of `main()`**, not file-scope code.

Expected console output:

```
rbAmp version: 0x04
U=230.1V  I=0.262A  P=+60.2W  PF=+0.998
U=229.9V  I=0.262A  P=+60.1W  PF=+0.998
...
```

---

## Example 2 — 60-second energy meter on OLED

**Goal**: Wh counter refreshed once per minute, shown on a 128×64 SSD1306 sharing the rbAmp I2C bus.
**Hardware**: STM32 + rbAmp + SSD1306 (`0x3C`) on the same bus.
**Accounting**: unidirectional (BASIC tier).
**Library**: any STM32-HAL SSD1306 driver (e.g. `afiskon/stm32-ssd1306`).

```c
#include <stdio.h>
#include <string.h>
#include "rbamp_io.h"
#include "ssd1306.h"        /* your driver of choice */
#include "ssd1306_fonts.h"

extern I2C_HandleTypeDef hi2c1;
#define RB_ADDR  0x50U

/* USER CODE BEGIN 2 */
RbAmpIO_Bind(&hi2c1);
ssd1306_Init();
RbAmpIO_WaitReady(RB_ADDR, 2000);

/* Primer latch — discard the snapshot it produces. */
RbAmpIO_WriteU8(RB_ADDR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
uint32_t t_prev_ms = HAL_GetTick();
double total_wh = 0.0;
/* USER CODE END 2 */

while (1) {
    HAL_Delay(60UL * 1000UL);

    /* 1) Final latch — closes the period [t_prev_ms .. now]. */
    RbAmpIO_WriteU8(RB_ADDR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
    uint32_t t_now_ms = HAL_GetTick();
    HAL_Delay(50);                              /* let firmware finalise snapshot */

    /* 2) Check PERIOD_VALID. */
    uint8_t valid = 0;
    RbAmpIO_ReadU8(RB_ADDR, RB_REG_PERIOD_VALID, &valid);
    if ((valid & 0x01U) == 0) {
        printf("WARN: stale snapshot\r\n");
        t_prev_ms = HAL_GetTick();
        continue;
    }

    /* 3) Read time-averaged power, channel 0. */
    float avg_p = 0.0f;
    RbAmpIO_ReadFloatLE(RB_ADDR, RB_REG_PERIOD_AVG_P0, &avg_p);

    /* 4) Energy uses the MASTER clock. */
    float dt_s = (float)(t_now_ms - t_prev_ms) / 1000.0f;
    double e_wh = (double)avg_p * dt_s / 3600.0;
    total_wh += e_wh;
    t_prev_ms = t_now_ms;

    /* 5) Render on OLED. */
    char line[24];
    ssd1306_Fill(Black);

    snprintf(line, sizeof(line), "P:  %6.1f W", avg_p);
    ssd1306_SetCursor(0, 0);   ssd1306_WriteString(line, Font_7x10, White);

    snprintf(line, sizeof(line), "dt: %6.1f s", dt_s);
    ssd1306_SetCursor(0, 16);  ssd1306_WriteString(line, Font_7x10, White);

    snprintf(line, sizeof(line), "%.2f Wh", total_wh);
    ssd1306_SetCursor(0, 32);  ssd1306_WriteString(line, Font_11x18, White);

    ssd1306_UpdateScreen();
    printf("avg_P=%.1f  E_period=%.3f  total=%.3f\r\n", avg_p, e_wh, total_wh);
}
```

> The SSD1306 driver and rbAmp share `hi2c1` — never re-initialise the peripheral, just use the same handle for both.

---

## Example 3 — Multi-module monitor

**Goal**: poll three modules (main feed, water heater, AC) and print the total.
**Hardware**: STM32 + 3 × rbAmp UI1 at `0x50`, `0x51`, `0x52`.

```c
#include <stdio.h>
#include "rbamp_io.h"

extern I2C_HandleTypeDef hi2c1;

typedef struct {
    uint8_t  addr;
    const char *label;
} module_t;

static const module_t modules[] = {
    { 0x50, "Mains " },
    { 0x51, "Boiler" },
    { 0x52, "AC    " },
};
#define N_MODULES (sizeof(modules) / sizeof(modules[0]))

/* USER CODE BEGIN 2 */
RbAmpIO_Bind(&hi2c1);

/* Bus probe — halt if any module is missing. */
for (size_t i = 0; i < N_MODULES; i++) {
    if (!RbAmpIO_Probe(modules[i].addr)) {
        printf("ERROR: %s at 0x%02X not found\r\n",
               modules[i].label, modules[i].addr);
        Error_Handler();
    }
}
printf("All modules OK\r\n");
/* USER CODE END 2 */

while (1) {
    float total_p = 0.0f;
    printf("---\r\n");

    /* Round-robin read. */
    for (size_t i = 0; i < N_MODULES; i++) {
        float u = 0.0f, p = 0.0f;
        RbAmpIO_ReadFloatLE(modules[i].addr, RB_REG_U_RMS,   &u);
        RbAmpIO_ReadFloatLE(modules[i].addr, RB_REG_P0_REAL, &p);
        printf("[%s] U=%.1f  P=%+.1f W\r\n", modules[i].label, u, p);
        total_p += p;
    }
    printf("TOTAL: %.1f W\r\n", total_p);
    HAL_Delay(2000);
}
```

`HAL_I2C_IsDeviceReady()` (inside `RbAmpIO_Probe()`) is the STM32 equivalent of the Arduino `Wire.endTransmission()` probe.

---

## Example 4 — Per-appliance energy tracker (UI3)

**Goal**: a UI3 module with three CT clamps on three different loads on the same phase; independent Wh counters published to MQTT every minute.
**Hardware**: STM32 + 1 × rbAmp UI3 + Ethernet PHY (or W5500 / ESP32-AT bridge).
**Network stubs**: `Net_Init()` and `Net_MqttPublish()` (see the chapter intro).

```c
#include <stdio.h>
#include "rbamp_io.h"

extern I2C_HandleTypeDef hi2c1;
extern HAL_StatusTypeDef Net_Init(void);
extern HAL_StatusTypeDef Net_MqttPublish(const char *topic,
                                          const uint8_t *payload, uint32_t len,
                                          uint8_t qos, uint8_t retain);

#define RB_ADDR  0x50U
static const char *CH_NAMES[3] = { "main", "heatpump", "lights" };

/**
 * @brief  Publish one channel's data to MQTT.
 */
static void mqtt_publish_channel(const char *ch_name, float avg_p, double e_wh) {
    char topic[64], payload[64];
    int t = snprintf(topic, sizeof(topic), "rbamp/%s/state", ch_name);
    int n = snprintf(payload, sizeof(payload),
                     "{\"power\":%.1f,\"energy\":%.4f}", avg_p, e_wh);
    (void)t;
    Net_MqttPublish(topic, (const uint8_t*)payload, (uint32_t)n, /*qos*/ 0, /*retain*/ 0);
}

/* USER CODE BEGIN 2 */
RbAmpIO_Bind(&hi2c1);
Net_Init();
RbAmpIO_WaitReady(RB_ADDR, 2000);

RbAmpIO_WriteU8(RB_ADDR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);   /* primer */
uint32_t t_prev_ms = HAL_GetTick();
double total_wh[3] = { 0.0, 0.0, 0.0 };
/* USER CODE END 2 */

while (1) {
    HAL_Delay(60UL * 1000UL);

    RbAmpIO_WriteU8(RB_ADDR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
    uint32_t t_now_ms = HAL_GetTick();
    HAL_Delay(50);

    uint8_t valid = 0;
    RbAmpIO_ReadU8(RB_ADDR, RB_REG_PERIOD_VALID, &valid);
    if ((valid & 0x01U) == 0) {
        t_prev_ms = HAL_GetTick();
        continue;
    }

    /* PERIOD_AVG_P_W per channel: I0 at 0xDC, I1 at 0xC2, I2 at 0xC6. */
    float avg_p[3];
    RbAmpIO_ReadFloatLE(RB_ADDR, RB_REG_PERIOD_AVG_P0, &avg_p[0]);
    RbAmpIO_ReadFloatLE(RB_ADDR, RB_REG_PERIOD_AVG_P1, &avg_p[1]);
    RbAmpIO_ReadFloatLE(RB_ADDR, RB_REG_PERIOD_AVG_P2, &avg_p[2]);

    float dt_s = (float)(t_now_ms - t_prev_ms) / 1000.0f;
    t_prev_ms = t_now_ms;

    for (int ch = 0; ch < 3; ch++) {
        double e = (double)avg_p[ch] * dt_s / 3600.0;
        total_wh[ch] += e;
        printf("[%s] avg_P=%+.1f W  E_total=%.3f Wh\r\n",
               CH_NAMES[ch], avg_p[ch], total_wh[ch]);
        mqtt_publish_channel(CH_NAMES[ch], avg_p[ch], total_wh[ch]);
    }
}
```

---

## Example 5 — Master-side bidirectional on a BASIC module

**Goal**: implement bidirectional accounting on the master while the module is BASIC firmware. The DRDY falling edge fires an EXTI interrupt; the ISR sets a flag, and the main loop reads RT `P_real` and splits consumption / export.
**Hardware**: STM32 + rbAmp UI1; DRDY wired to PA0 (configured as `GPIO_EXTI0` falling, pull-up in CubeMX).

In CubeMX:
- `PA0` → mode `GPIO_EXTI0`, pull-up, trigger `Falling edge`
- NVIC: enable `EXTI line 0 interrupt`

`stm32xxxx_it.c` — CubeMX-generated, only the user-code block:

```c
extern volatile uint8_t g_data_ready;     /* defined in main.c */

void EXTI0_IRQHandler(void) {
    HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_0);
}

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    if (GPIO_Pin == GPIO_PIN_0) {
        g_data_ready = 1;                 /* tiny ISR — set a flag and return */
    }
}
```

`main.c` user code:

```c
#include <stdio.h>
#include <math.h>
#include "rbamp_io.h"

extern I2C_HandleTypeDef hi2c1;
volatile uint8_t g_data_ready = 0;

#define RB_ADDR  0x50U

/* USER CODE BEGIN 2 */
RbAmpIO_Bind(&hi2c1);
RbAmpIO_WaitReady(RB_ADDR, 2000);

double e_consumed_wh = 0.0;
double e_exported_wh = 0.0;
uint32_t t_last_sample_ms = HAL_GetTick();
uint32_t t_last_print_ms  = t_last_sample_ms;
printf("Bidirectional accumulator started\r\n");
/* USER CODE END 2 */

while (1) {
    if (!g_data_ready) {
        continue;                          /* main loop also free to do other work */
    }
    __disable_irq();
    g_data_ready = 0;                      /* clear flag atomically */
    __enable_irq();

    uint32_t t_now = HAL_GetTick();

    /* Signed instantaneous P_real: + consume, − export. */
    float p = 0.0f;
    RbAmpIO_ReadFloatLE(RB_ADDR, RB_REG_P0_REAL, &p);
    float dt_s = (float)(t_now - t_last_sample_ms) / 1000.0f;
    t_last_sample_ms = t_now;

    if (p > 0.0f) {
        e_consumed_wh += (double)p * dt_s / 3600.0;
    } else {
        e_exported_wh += (double)(-p) * dt_s / 3600.0;
    }

    /* Summary print every 5 s — not on every RT window. */
    if (t_now - t_last_print_ms >= 5000) {
        t_last_print_ms = t_now;
        printf("P=%+8.1fW   consumed=%.4f Wh  exported=%.4f Wh  net=%+.4f Wh\r\n",
               p, e_consumed_wh, e_exported_wh,
               e_consumed_wh - e_exported_wh);
    }
}
```

**Notes**:

- The EXTI ISR is minimal — just sets `g_data_ready`. All I2C work runs in the main loop.
- `volatile` qualifier is essential: the compiler must reload `g_data_ready` from RAM on every loop iteration.
- A short `__disable_irq() / __enable_irq()` window around the clear prevents a race with a near-simultaneous EXTI fire.

---

## Example 6 — Home energy balance

**Goal**: full balance dashboard. Three modules:
- rbAmp #1 on the main feed (STANDARD / PRO — bidirectional)
- rbAmp #2 on the solar inverter output (BASIC OK)
- rbAmp #3 UI3 on three large loads (HP, AC, EV)

Master uses `RbAmpIO_BroadcastLatch()` for synchronous snapshots and publishes a retained balance to MQTT every minute.

```c
#include <stdio.h>
#include <string.h>
#include "rbamp_io.h"

extern I2C_HandleTypeDef hi2c1;
extern HAL_StatusTypeDef Net_Init(void);
extern HAL_StatusTypeDef Net_MqttPublish(const char *topic,
                                          const uint8_t *payload, uint32_t len,
                                          uint8_t qos, uint8_t retain);

#define ADDR_MAINS  0x50U   /* bidirectional (STANDARD/PRO) */
#define ADDR_SOLAR  0x51U   /* generation only */
#define ADDR_LOADS  0x52U   /* UI3 */

/* PERIOD_AVG_P_NEG address for the mains module — see the SKU datasheet
 * of the STANDARD/PRO product. Set to 0xFF for BASIC modules. */
#define REG_PERIOD_AVG_P_NEG_MAINS  0xFFU

typedef struct {
    double mains_in_wh;
    double mains_out_wh;
    double solar_total_wh;
    double loads_wh[3];
} totals_t;

/* USER CODE BEGIN 2 */
RbAmpIO_Bind(&hi2c1);
Net_Init();
RbAmpIO_WaitReady(ADDR_MAINS, 2000);
RbAmpIO_WaitReady(ADDR_SOLAR, 2000);
RbAmpIO_WaitReady(ADDR_LOADS, 2000);

static uint16_t gc_tick = 0;
if (RbAmpIO_BroadcastLatch(gc_tick++, 0) != HAL_OK) {
    /* GC NACK — fall back to per-module sequential latch. */
    RbAmpIO_WriteU8(ADDR_MAINS, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
    RbAmpIO_WriteU8(ADDR_SOLAR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
    RbAmpIO_WriteU8(ADDR_LOADS, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
}
uint32_t t_prev_ms = HAL_GetTick();

totals_t totals = {0};
printf("Home balance started\r\n");
/* USER CODE END 2 */

while (1) {
    HAL_Delay(60UL * 1000UL);

    /* Synchronous latch — all modules snapshot at the same instant. */
    if (RbAmpIO_BroadcastLatch(gc_tick++, 0) != HAL_OK) {
        /* GC NACK — fall back to per-module latch (skew ~1 ms/module). */
        RbAmpIO_WriteU8(ADDR_MAINS, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
        RbAmpIO_WriteU8(ADDR_SOLAR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
        RbAmpIO_WriteU8(ADDR_LOADS, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
    }
    uint32_t t_now_ms = HAL_GetTick();
    HAL_Delay(50);

    float dt_s = (float)(t_now_ms - t_prev_ms) / 1000.0f;
    t_prev_ms = t_now_ms;

    /* ----- MAINS (bidirectional, STANDARD/PRO) ----- */
    uint8_t valid;
    if (RbAmpIO_ReadU8(ADDR_MAINS, RB_REG_PERIOD_VALID, &valid) == HAL_OK
        && (valid & 0x01U)) {
        float p_consume = 0.0f, p_export = 0.0f;
        RbAmpIO_ReadFloatLE(ADDR_MAINS, RB_REG_PERIOD_AVG_P0, &p_consume);
        if (REG_PERIOD_AVG_P_NEG_MAINS != 0xFFU) {
            RbAmpIO_ReadFloatLE(ADDR_MAINS, REG_PERIOD_AVG_P_NEG_MAINS, &p_export);
        }
        totals.mains_in_wh  += (double)p_consume * dt_s / 3600.0;
        totals.mains_out_wh += (double)p_export  * dt_s / 3600.0;
    }

    /* ----- SOLAR (generation only) ----- */
    if (RbAmpIO_ReadU8(ADDR_SOLAR, RB_REG_PERIOD_VALID, &valid) == HAL_OK
        && (valid & 0x01U)) {
        float p = 0.0f;
        RbAmpIO_ReadFloatLE(ADDR_SOLAR, RB_REG_PERIOD_AVG_P0, &p);
        totals.solar_total_wh += (double)p * dt_s / 3600.0;
    }

    /* ----- LOADS (UI3, three channels) ----- */
    if (RbAmpIO_ReadU8(ADDR_LOADS, RB_REG_PERIOD_VALID, &valid) == HAL_OK
        && (valid & 0x01U)) {
        float p[3] = {0};
        RbAmpIO_ReadFloatLE(ADDR_LOADS, RB_REG_PERIOD_AVG_P0, &p[0]);  /* I0 */
        RbAmpIO_ReadFloatLE(ADDR_LOADS, RB_REG_PERIOD_AVG_P1, &p[1]);  /* I1 */
        RbAmpIO_ReadFloatLE(ADDR_LOADS, RB_REG_PERIOD_AVG_P2, &p[2]);  /* I2 */
        for (int i = 0; i < 3; i++) {
            totals.loads_wh[i] += (double)p[i] * dt_s / 3600.0;
        }
    }

    double total_consumed  = totals.mains_in_wh + totals.solar_total_wh
                           - totals.mains_out_wh;
    double solar_self_used = totals.solar_total_wh - totals.mains_out_wh;
    if (solar_self_used < 0.0) solar_self_used = 0.0;

    printf("MAINS  in=%.2f  out=%.2f\r\n", totals.mains_in_wh, totals.mains_out_wh);
    printf("SOLAR  gen=%.2f  self-used=%.2f  exported=%.2f\r\n",
           totals.solar_total_wh, solar_self_used, totals.mains_out_wh);
    printf("LOADS  HP=%.2f  AC=%.2f  EV=%.2f\r\n",
           totals.loads_wh[0], totals.loads_wh[1], totals.loads_wh[2]);
    printf("TOTAL  household consumed=%.2f Wh\r\n", total_consumed);

    char payload[512];
    int n = snprintf(payload, sizeof(payload),
        "{\"mains_in\":%.3f,\"mains_out\":%.3f,\"solar\":%.3f,"
        "\"self_used\":%.3f,\"total_consumed\":%.3f,"
        "\"hp\":%.3f,\"ac\":%.3f,\"ev\":%.3f}",
        totals.mains_in_wh, totals.mains_out_wh, totals.solar_total_wh,
        solar_self_used, total_consumed,
        totals.loads_wh[0], totals.loads_wh[1], totals.loads_wh[2]);
    Net_MqttPublish("home/energy/balance", (const uint8_t*)payload,
                     (uint32_t)n, /*qos*/ 1, /*retain*/ 1);
}
```

---

## Example 7 — Power-event detection

**Goal**: on every DRDY edge, compare instantaneous P to an EMA and log significant deviations to an SD card via SDIO + FATFS.
**Hardware**: STM32F4 / L4 / H7 + rbAmp + SD-card on SDIO interface (CubeMX: enable `FATFS` middleware + `SDIO`).

CubeMX setup:
- `SDIO` mode = "SD 4 bits Wide bus" or "SD 1 bit"
- `FATFS` middleware enabled, mount point default
- `PA0` = `GPIO_EXTI0`, falling, pull-up (DRDY)

```c
#include <stdio.h>
#include <math.h>
#include <string.h>
#include "rbamp_io.h"
#include "fatfs.h"

extern I2C_HandleTypeDef hi2c1;
extern FATFS  SDFatFS;
extern char   SDPath[4];
extern volatile uint8_t g_data_ready;  /* set by EXTI0 callback */

#define RB_ADDR  0x50U
#define EMA_ALPHA         0.05f
#define EVENT_THRESHOLD_W 200.0f

static FIL  s_logfile;
static bool s_log_open = false;       /* guard — f_write on closed FIL is UB */

/* USER CODE BEGIN 2 */
RbAmpIO_Bind(&hi2c1);
RbAmpIO_WaitReady(RB_ADDR, 2000);

/* Mount the SD card (CubeMX has already set up SDFatFS / SDPath).
 * Both mount and open MUST succeed before we touch s_logfile in the loop. */
if (f_mount(&SDFatFS, SDPath, 1) == FR_OK) {
    if (f_open(&s_logfile, "events.log",
               FA_OPEN_APPEND | FA_WRITE) == FR_OK) {
        s_log_open = true;
        printf("SD mounted; logging to events.log\r\n");
    } else {
        printf("WARN: f_open(events.log) failed — file logging disabled\r\n");
    }
} else {
    printf("WARN: f_mount failed — file logging disabled\r\n");
}

/* Seed the EMA with current power so we do not log a spurious startup event. */
float p_ema = 0.0f;
RbAmpIO_ReadFloatLE(RB_ADDR, RB_REG_P0_REAL, &p_ema);
printf("Event detector started\r\n");
/* USER CODE END 2 */

while (1) {
    if (!g_data_ready) continue;
    __disable_irq(); g_data_ready = 0; __enable_irq();

    float p = 0.0f;
    RbAmpIO_ReadFloatLE(RB_ADDR, RB_REG_P0_REAL, &p);

    float delta = p - p_ema;
    p_ema = (1.0f - EMA_ALPHA) * p_ema + EMA_ALPHA * p;

    if (fabsf(delta) > EVENT_THRESHOLD_W) {
        const char *type = (delta > 0.0f) ? "TURN_ON" : "TURN_OFF";
        char line[128];
        int n = snprintf(line, sizeof(line),
                         "%lu  %s  delta=%+.1f W  P=%.1f W  EMA=%.1f W\r\n",
                         (unsigned long)HAL_GetTick(),
                         type, delta, p, p_ema);
        printf("%s", line);
        if (s_log_open) {                       /* never touch a closed FIL */
            UINT bw;
            f_write(&s_logfile, line, n, &bw);
            f_sync(&s_logfile);                 /* flush to the card */
        }
    }
}
```

---

## Example 8 — MQTT publisher with Home Assistant Auto-discovery

**Goal**: drop-in HA integration. The STM32 publishes HA Auto-discovery configs (retained) and emits a JSON state every minute.
**Hardware**: STM32 with Ethernet (H7 / F7 / F4 + LAN8742) or any STM32 + Ethernet bridge.

```c
#include <stdio.h>
#include <string.h>
#include "rbamp_io.h"

extern I2C_HandleTypeDef hi2c1;
extern HAL_StatusTypeDef Net_Init(void);
extern HAL_StatusTypeDef Net_MqttPublish(const char *topic,
                                          const uint8_t *payload, uint32_t len,
                                          uint8_t qos, uint8_t retain);

#define RB_ADDR     0x50U
#define DEVICE_ID   "rbamp_main"
#define DEVICE_NAME "Mains rbAmp"

/**
 * @brief  Publish one HA discovery config (retained).
 */
static void publish_discovery_sensor(const char *key, const char *friendly,
                                     const char *unit, const char *dev_class,
                                     const char *state_class) {
    char topic[128], payload[512];
    snprintf(topic, sizeof(topic),
             "homeassistant/sensor/%s/%s/config", DEVICE_ID, key);

    int n = snprintf(payload, sizeof(payload),
        "{"
          "\"name\":\"%s %s\","
          "\"unique_id\":\"%s_%s\","
          "\"state_topic\":\"rbamp/%s/state\","
          "\"value_template\":\"{{ value_json.%s }}\","
          "\"state_class\":\"%s\","
          "\"device\":{"
            "\"identifiers\":[\"%s\"],"
            "\"name\":\"%s\","
            "\"manufacturer\":\"rbAmp\","
            "\"model\":\"rbAmp UI*\""
          "}",
        DEVICE_NAME, friendly, DEVICE_ID, key,
        DEVICE_ID, key, state_class,
        DEVICE_ID, DEVICE_NAME);
    if (unit)      n += snprintf(payload + n, sizeof(payload) - n,
                                  ",\"unit_of_measurement\":\"%s\"", unit);
    if (dev_class) n += snprintf(payload + n, sizeof(payload) - n,
                                  ",\"device_class\":\"%s\"", dev_class);
    n += snprintf(payload + n, sizeof(payload) - n, "}");

    Net_MqttPublish(topic, (const uint8_t*)payload, (uint32_t)n,
                    /*qos*/ 1, /*retain*/ 1);
}

/**
 * @brief  Publish the full set of HA discovery configs for one rbAmp.
 */
static void publish_discovery_all(void) {
    publish_discovery_sensor("voltage",        "Voltage",        "V",   "voltage",        "measurement");
    publish_discovery_sensor("current",        "Current",        "A",   "current",        "measurement");
    publish_discovery_sensor("power",          "Power",          "W",   "power",          "measurement");
    publish_discovery_sensor("energy",         "Energy",         "Wh",  "energy",         "total_increasing");
    publish_discovery_sensor("frequency",      "Frequency",      "Hz",  "frequency",      "measurement");
    publish_discovery_sensor("power_factor",   "Power Factor",   NULL,  "power_factor",   "measurement");
    publish_discovery_sensor("apparent_power", "Apparent Power", "VA",  "apparent_power", "measurement");
    publish_discovery_sensor("reactive_power", "Reactive Power", "var", "reactive_power", "measurement");
}

/* USER CODE BEGIN 2 */
RbAmpIO_Bind(&hi2c1);
Net_Init();
publish_discovery_all();

RbAmpIO_WaitReady(RB_ADDR, 2000);
RbAmpIO_WriteU8(RB_ADDR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);  /* primer */
uint32_t t_prev_ms = HAL_GetTick();
double total_wh = 0.0;
/* USER CODE END 2 */

while (1) {
    HAL_Delay(60UL * 1000UL);

    RbAmpIO_WriteU8(RB_ADDR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
    uint32_t t_now_ms = HAL_GetTick();
    HAL_Delay(50);

    uint8_t valid = 0;
    RbAmpIO_ReadU8(RB_ADDR, RB_REG_PERIOD_VALID, &valid);
    if ((valid & 0x01U) == 0) {
        t_prev_ms = HAL_GetTick();
        continue;
    }
    float avg_p = 0.0f;
    RbAmpIO_ReadFloatLE(RB_ADDR, RB_REG_PERIOD_AVG_P0, &avg_p);
    float dt_s = (float)(t_now_ms - t_prev_ms) / 1000.0f;
    total_wh += (double)avg_p * dt_s / 3600.0;
    t_prev_ms = t_now_ms;

    /* Live RT values for the state payload. */
    float u = 0, i_ = 0, pf = 0, q = 0;
    uint8_t freq = 0;
    RbAmpIO_ReadFloatLE(RB_ADDR, RB_REG_U_RMS,  &u);
    RbAmpIO_ReadFloatLE(RB_ADDR, RB_REG_I0_RMS, &i_);
    RbAmpIO_ReadFloatLE(RB_ADDR, RB_REG_PF0,    &pf);
    RbAmpIO_ReadFloatLE(RB_ADDR, RB_REG_Q0,     &q);
    RbAmpIO_ReadU8     (RB_ADDR, RB_REG_AC_FREQ, &freq);

    char payload[384];
    int n = snprintf(payload, sizeof(payload),
        "{"
          "\"voltage\":%.1f,"
          "\"current\":%.3f,"
          "\"power\":%.1f,"
          "\"energy\":%.3f,"
          "\"frequency\":%u,"
          "\"power_factor\":%.3f,"
          "\"apparent_power\":%.1f,"
          "\"reactive_power\":%.1f"
        "}",
        u, i_, avg_p, total_wh,
        (unsigned)freq, pf, u * i_, q);

    char topic[64];
    snprintf(topic, sizeof(topic), "rbamp/%s/state", DEVICE_ID);
    Net_MqttPublish(topic, (const uint8_t*)payload, (uint32_t)n,
                    /*qos*/ 0, /*retain*/ 0);
    printf("Published: %s\r\n", payload);
}
```

---

## Example 9 — Battery-powered logger with STANDBY mode

**Goal**: an outdoor / off-grid logger that wakes from STANDBY every 10 minutes via the RTC alarm, latches a period, publishes via MQTT, then re-enters STANDBY. Current draw is dominated by the rare wake — STANDBY is single-digit µA on most STM32 families.
**Hardware**: STM32L4 / L5 (best low-power), or F4/F7 with reduced features. rbAmp may be powered through a high-side switch on a GPIO so it consumes nothing during STANDBY.
**Persistence**: RTC backup registers `RTC->BKP0R..BKPxR` (varies by family) survive STANDBY and VBAT.

```c
#include <stdio.h>
#include <string.h>
#include "rbamp_io.h"
/* CubeMX-generated: RTC + Wakeup timer + Backup registers in PWR */

extern I2C_HandleTypeDef hi2c1;
extern RTC_HandleTypeDef hrtc;
extern HAL_StatusTypeDef Net_Init(void);
extern HAL_StatusTypeDef Net_MqttPublish(const char *topic,
                                          const uint8_t *payload, uint32_t len,
                                          uint8_t qos, uint8_t retain);

#define RB_ADDR  0x50U
#define WAKE_SECONDS  (10U * 60U)       /* 10 minutes */

/* Backup-register layout (uint32 per slot). All survive STANDBY and VBAT.
 * BKP0..1: total_wh (double, split into two uint32 halves)
 * BKP2:    wake_count
 * BKP3:    primer_done (0/1) + flags
 * BKP4..5: t_prev_ms_ticks (uint64 split)
 */
#define BKP_TOTAL_WH_LO  RTC_BKP_DR0
#define BKP_TOTAL_WH_HI  RTC_BKP_DR1
#define BKP_WAKE_COUNT   RTC_BKP_DR2
#define BKP_PRIMER       RTC_BKP_DR3
#define BKP_T_PREV_LO    RTC_BKP_DR4
#define BKP_T_PREV_HI    RTC_BKP_DR5

static void bkp_load(double *total_wh, uint32_t *wakes,
                     uint8_t *primer, uint64_t *t_prev_ms) {
    uint32_t lo = HAL_RTCEx_BKUPRead(&hrtc, BKP_TOTAL_WH_LO);
    uint32_t hi = HAL_RTCEx_BKUPRead(&hrtc, BKP_TOTAL_WH_HI);
    uint64_t bits = ((uint64_t)hi << 32) | lo;
    memcpy(total_wh, &bits, sizeof(double));
    *wakes  = HAL_RTCEx_BKUPRead(&hrtc, BKP_WAKE_COUNT);
    *primer = (uint8_t)HAL_RTCEx_BKUPRead(&hrtc, BKP_PRIMER);
    uint32_t plo = HAL_RTCEx_BKUPRead(&hrtc, BKP_T_PREV_LO);
    uint32_t phi = HAL_RTCEx_BKUPRead(&hrtc, BKP_T_PREV_HI);
    *t_prev_ms = ((uint64_t)phi << 32) | plo;
}

static void bkp_save(double total_wh, uint32_t wakes,
                     uint8_t primer, uint64_t t_prev_ms) {
    HAL_PWR_EnableBkUpAccess();
    uint64_t bits;
    memcpy(&bits, &total_wh, sizeof(double));
    HAL_RTCEx_BKUPWrite(&hrtc, BKP_TOTAL_WH_LO, (uint32_t)(bits & 0xFFFFFFFFU));
    HAL_RTCEx_BKUPWrite(&hrtc, BKP_TOTAL_WH_HI, (uint32_t)(bits >> 32));
    HAL_RTCEx_BKUPWrite(&hrtc, BKP_WAKE_COUNT,  wakes);
    HAL_RTCEx_BKUPWrite(&hrtc, BKP_PRIMER,      primer);
    HAL_RTCEx_BKUPWrite(&hrtc, BKP_T_PREV_LO,   (uint32_t)(t_prev_ms & 0xFFFFFFFFU));
    HAL_RTCEx_BKUPWrite(&hrtc, BKP_T_PREV_HI,   (uint32_t)(t_prev_ms >> 32));
}

/**
 * @brief  Arm the RTC wake-up timer and enter STANDBY.
 * @details On the next wake (or RESET via NRST), execution restarts at the
 *          reset vector — exactly like a fresh boot.
 */
static void enter_standby(uint32_t seconds) {
    HAL_RTCEx_SetWakeUpTimer_IT(&hrtc, seconds,
                                 RTC_WAKEUPCLOCK_CK_SPRE_16BITS);
    HAL_PWR_EnterSTANDBYMode();
    /* never returns */
}

/* USER CODE BEGIN 2 */
double   total_wh;
uint32_t wake_count;
uint8_t  primer_done;
uint64_t t_prev_unused;            /* reserved BKP slot; see note below */
bkp_load(&total_wh, &wake_count, &primer_done, &t_prev_unused);
(void)t_prev_unused;               /* HAL_GetTick() resets after STANDBY — */
                                    /* the dt is approximated by WAKE_SECONDS  */
                                    /* instead. Slot kept for forward compat;  */
                                    /* upgrade to HAL_RTC_GetTime/Date if you  */
                                    /* need true wall-clock dt across sleeps.  */
wake_count++;

RbAmpIO_Bind(&hi2c1);
if (RbAmpIO_WaitReady(RB_ADDR, 2000) != HAL_OK) {
    bkp_save(total_wh, wake_count, primer_done, /*t_prev=*/0);
    enter_standby(WAKE_SECONDS);
}

/* First wake-up after a cold boot: primer latch only. */
if (!primer_done) {
    RbAmpIO_WriteU8(RB_ADDR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
    primer_done = 1;
    bkp_save(total_wh, wake_count, primer_done, /*t_prev=*/0);
    printf("Primer done — entering STANDBY\r\n");
    enter_standby(WAKE_SECONDS);
}

/* Subsequent wake-ups: close the period and read its average power. */
RbAmpIO_WriteU8(RB_ADDR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
HAL_Delay(50);

uint8_t valid = 0;
RbAmpIO_ReadU8(RB_ADDR, RB_REG_PERIOD_VALID, &valid);
if ((valid & 0x01U) == 0) {
    bkp_save(total_wh, wake_count, primer_done, /*t_prev=*/0);
    enter_standby(WAKE_SECONDS);
}

float avg_p = 0.0f;
RbAmpIO_ReadFloatLE(RB_ADDR, RB_REG_PERIOD_AVG_P0, &avg_p);
/* dt approximation: STANDBY resets HAL_GetTick(), and the WAKE_SECONDS-driven
 * RTC wake-timer is the wall-clock interval to single-digit-% accuracy with
 * an LSE crystal. For tighter accounting, read HAL_RTC_GetTime/Date before
 * STANDBY and compute dt against the stored epoch. */
float dt_s = (float)WAKE_SECONDS;
total_wh += (double)avg_p * dt_s / 3600.0;

/* Publish via Wi-Fi / Ethernet (graceful fallback). */
if (Net_Init() == HAL_OK) {
    char payload[256];
    int n = snprintf(payload, sizeof(payload),
        "{\"wake\":%lu,\"dt_s\":%.1f,\"avg_p\":%.1f,\"energy_wh\":%.3f}",
        (unsigned long)wake_count, dt_s, avg_p, total_wh);
    Net_MqttPublish("rbamp/remote/state",
                    (const uint8_t*)payload, (uint32_t)n,
                    /*qos*/ 1, /*retain*/ 1);
    printf("Published: %s\r\n", payload);
}

bkp_save(total_wh, wake_count, primer_done, /*t_prev=*/0);
enter_standby(WAKE_SECONDS);
/* USER CODE END 2 */
```

**Notes**:

- Backup registers (`BKP_DR*`) survive **STANDBY** and **VBAT** but **not** a full power loss without a VBAT backup battery. For absolute persistence, mirror the total to internal flash periodically.
- `HAL_GetTick()` is reset by every STANDBY → use the RTC calendar (`HAL_RTC_GetTime/Date`) for absolute wall-clock dt if your application needs it.
- The CubeMX RTC must be configured with **LSE** (32.768 kHz crystal) for accurate long-term timing; LSI drift can be 5–10 % at temperature.
- For STM32F1 / F4 the wake-up clock source enum may differ slightly; check the family's HAL reference for `RTC_WAKEUPCLOCK_*` values.

---

## Example 10 — Time-of-use (TOU) tariff with SNTP wall-clock

**Goal**: bill energy under a time-of-use schedule. The STM32 keeps a real wall-clock via SNTP (LwIP `lwip/apps/sntp.h`), splits each latched period into the right tariff bucket, resets daily totals at midnight, and publishes payloads with ISO timestamps.
**Hardware**: STM32 with Ethernet (STM32H7 / F7 / F4 + LAN8742) or any STM32 + Ethernet bridge.

```c
#include <stdio.h>
#include <string.h>
#include <time.h>
#include "rbamp_io.h"
#include "lwip/apps/sntp.h"

extern I2C_HandleTypeDef hi2c1;
extern RTC_HandleTypeDef hrtc;
extern HAL_StatusTypeDef Net_Init(void);
extern HAL_StatusTypeDef Net_MqttPublish(const char *topic,
                                          const uint8_t *payload, uint32_t len,
                                          uint8_t qos, uint8_t retain);

#define RB_ADDR  0x50U

#define TZ_OFFSET_HOURS  2          /* CEST = UTC+2; recompute on roll-over for DST */
#define PEAK_START_HOUR  8
#define PEAK_END_HOUR    22

/* IMPORTANT: LwIP's SNTP populates the system clock via the SNTP_SET_SYSTEM_TIME
 * macro that you MUST define in lwipopts.h. Without it, time(NULL) stays at 0
 * forever and sntp_sync() will always time out. Minimal hook:
 *
 *     #include <sys/time.h>
 *     #define SNTP_SET_SYSTEM_TIME(sec) \
 *         do { struct timeval tv = { (time_t)(sec), 0 }; \
 *              settimeofday(&tv, NULL); } while (0)
 *
 * Also requires _gettimeofday() / time() to be retargeted to the same source
 * (newlib's default works once settimeofday is wired). */

typedef struct {
    double peak_wh;
    double off_peak_wh;
    int    day_of_year;             /* tm_yday */
} day_totals_t;
typedef struct {
    double peak_wh;
    double off_peak_wh;
} lifetime_totals_t;

static day_totals_t      s_today    = { .day_of_year = -1 };
static lifetime_totals_t s_lifetime = { 0 };

/**
 * @brief  Start SNTP and wait until the system clock looks plausible (year > 2023).
 */
static HAL_StatusTypeDef sntp_sync(uint32_t timeout_ms) {
    sntp_setoperatingmode(SNTP_OPMODE_POLL);
    sntp_setservername(0, "pool.ntp.org");
    sntp_setservername(1, "time.google.com");
    sntp_init();

    uint32_t deadline = HAL_GetTick() + timeout_ms;
    while (HAL_GetTick() < deadline) {
        time_t now = time(NULL);
        struct tm lt;
        gmtime_r(&now, &lt);
        if (lt.tm_year + 1900 >= 2023) {
            printf("SNTP sync: %04d-%02d-%02d %02d:%02d:%02d UTC\r\n",
                   lt.tm_year + 1900, lt.tm_mon + 1, lt.tm_mday,
                   lt.tm_hour, lt.tm_min, lt.tm_sec);
            return HAL_OK;
        }
        HAL_Delay(500);
    }
    return HAL_TIMEOUT;
}

static struct tm local_time(void) {
    time_t now = time(NULL) + (TZ_OFFSET_HOURS * 3600);
    struct tm lt;
    gmtime_r(&now, &lt);
    return lt;
}

static bool is_peak_hour(int hour) {
    /* For schedules that cross midnight, swap to
     *    return (hour >= PEAK_START_HOUR) || (hour < PEAK_END_HOUR);
     */
    return (hour >= PEAK_START_HOUR) && (hour < PEAK_END_HOUR);
}

static void iso_now(char *out, size_t out_len) {
    struct tm lt = local_time();
    const char sign = (TZ_OFFSET_HOURS >= 0) ? '+' : '-';
    snprintf(out, out_len, "%04d-%02d-%02dT%02d:%02d:%02d%c%02d00",
             lt.tm_year + 1900, lt.tm_mon + 1, lt.tm_mday,
             lt.tm_hour, lt.tm_min, lt.tm_sec,
             sign, (TZ_OFFSET_HOURS < 0) ? -TZ_OFFSET_HOURS : TZ_OFFSET_HOURS);
}

/**
 * @brief  Add one period's energy to the right tariff bucket and roll over at midnight.
 */
static void accumulate(double e_wh, const struct tm *tm_now) {
    if (s_today.day_of_year != tm_now->tm_yday) {
        if (s_today.day_of_year >= 0) {
            char payload[192];
            int n = snprintf(payload, sizeof(payload),
                "{\"yday\":%d,\"peak_wh\":%.3f,\"off_peak_wh\":%.3f,"
                 "\"total_wh\":%.3f}",
                s_today.day_of_year, s_today.peak_wh, s_today.off_peak_wh,
                s_today.peak_wh + s_today.off_peak_wh);
            Net_MqttPublish("rbamp/tou/day_close",
                            (const uint8_t*)payload, (uint32_t)n,
                            /*qos*/ 1, /*retain*/ 1);
            printf("Day closed: %s\r\n", payload);
        }
        s_today.peak_wh = 0.0;
        s_today.off_peak_wh = 0.0;
        s_today.day_of_year = tm_now->tm_yday;
    }
    if (is_peak_hour(tm_now->tm_hour)) {
        s_today.peak_wh    += e_wh;
        s_lifetime.peak_wh += e_wh;
    } else {
        s_today.off_peak_wh    += e_wh;
        s_lifetime.off_peak_wh += e_wh;
    }
}

/* USER CODE BEGIN 2 */
RbAmpIO_Bind(&hi2c1);
Net_Init();
sntp_sync(15000);

RbAmpIO_WaitReady(RB_ADDR, 2000);
RbAmpIO_WriteU8(RB_ADDR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);   /* primer */
uint32_t t_prev_ms = HAL_GetTick();
printf("TOU started: peak %02d:00..%02d:00 local\r\n",
       PEAK_START_HOUR, PEAK_END_HOUR);
/* USER CODE END 2 */

while (1) {
    HAL_Delay(60UL * 1000UL);

    RbAmpIO_WriteU8(RB_ADDR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
    uint32_t t_now_ms = HAL_GetTick();
    HAL_Delay(50);

    uint8_t valid = 0;
    RbAmpIO_ReadU8(RB_ADDR, RB_REG_PERIOD_VALID, &valid);
    if ((valid & 0x01U) == 0) {
        t_prev_ms = HAL_GetTick();
        continue;
    }
    float avg_p = 0.0f;
    RbAmpIO_ReadFloatLE(RB_ADDR, RB_REG_PERIOD_AVG_P0, &avg_p);
    float dt_s = (float)(t_now_ms - t_prev_ms) / 1000.0f;
    double e_wh = (double)avg_p * dt_s / 3600.0;
    t_prev_ms = t_now_ms;

    /* Guard against fresh boot before SNTP succeeds. */
    struct tm lt = local_time();
    if (lt.tm_year + 1900 < 2023) {
        printf("WARN: clock not yet synced, retrying SNTP\r\n");
        sntp_sync(15000);
        continue;
    }
    accumulate(e_wh, &lt);
    bool peak = is_peak_hour(lt.tm_hour);

    char iso[40];
    iso_now(iso, sizeof(iso));

    char payload[384];
    int n = snprintf(payload, sizeof(payload),
        "{"
          "\"ts\":\"%s\","
          "\"tariff\":\"%s\","
          "\"avg_p\":%.1f,"
          "\"period_wh\":%.4f,"
          "\"today_peak_wh\":%.3f,"
          "\"today_off_peak_wh\":%.3f,"
          "\"life_peak_wh\":%.3f,"
          "\"life_off_peak_wh\":%.3f"
        "}",
        iso, peak ? "peak" : "off_peak",
        avg_p, e_wh,
        s_today.peak_wh, s_today.off_peak_wh,
        s_lifetime.peak_wh, s_lifetime.off_peak_wh);
    Net_MqttPublish("rbamp/tou/state", (const uint8_t*)payload,
                    (uint32_t)n, /*qos*/ 0, /*retain*/ 0);
    printf("%s\r\n", payload);
}
```

**Notes**:

- LwIP's SNTP client populates the C library `time(NULL)` source via the LwIP `SYS_ARCH_RAND` / `sys_now` hooks; on STM32 ports `gmtime_r()` then returns sensible values.
- For full DST handling, store the TZ offset in a small lookup table and recompute on the daily roll-over, or pipe the time through `tzset()` with a POSIX TZ string if your newlib build supports it (most do).
- For STANDARD / PRO bidirectional metering, call `accumulate()` twice — once for the consumption side and once for the export side — and publish both peak/off-peak buckets.

---

## Example comparison

| # | Complexity | Output | MQTT | DRDY | Multi-module | Bidirectional | Persistence | Use case |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|---|
| 1 | minimal | UART printf | — | — | — | — | — | smoke test |
| 2 | low | OLED | — | — | — | — | RAM | boxed meter |
| 3 | low | UART printf | — | — | yes (3) | — | — | home monitoring |
| 4 | medium | — | yes | — | — | — | RAM | per-appliance in HA |
| 5 | medium | UART printf | — | yes | — | yes (master) | RAM | solar home on BASIC tier |
| 6 | high | — | yes | — | yes (3) | yes | RAM | full home balance |
| 7 | medium | UART + SD (FATFS) | — | yes | — | — | SD card | event detection |
| 8 | medium | — | yes (+disco) | — | — | optional | RAM + MQTT | HA Auto-discovery |
| 9 | medium | — | yes | — | — | — | RTC BKP regs | off-grid / outdoor logger |
| 10 | medium | UART printf | yes | — | — | optional | RAM + MQTT | TOU peak/off-peak with SNTP |

## Best practices

- **Always check HAL return codes.** I2C transactions can transiently fail on long buses; production code should retry rather than ignore. Wrap retries around `RbAmpIO_*` calls if needed.
- **Keep ISRs tiny.** EXTI / SysTick / DMA-complete callbacks should set a `volatile uint8_t` flag and return. All I2C / printf work belongs in the main loop.
- **One HAL I2C driver per peripheral.** Both rbAmp and OLED share `hi2c1` — never re-initialise the peripheral, just use the shared handle.
- **Use `HAL_GetTick()` for the master clock.** Millisecond precision, monotonic, survives most HAL operations.
- **`PERIOD_VALID` (`0x07`) must be checked after every `CMD_LATCH_PERIOD`.** Stale snapshots happen if the master polls faster than the firmware integrates.
- **Wait ≥ 50 ms after a latch** before reading `0xDC..0xEF` — the firmware finalises the snapshot inside its main loop.
- **Use `HAL_I2C_IsDeviceReady()` for bus scans** — STM32 equivalent of the Arduino `endTransmission()` ping.
- **For battery applications**, enter STOP or STANDBY between samples and arm the RTC wake-up timer. STANDBY restarts execution at the reset vector — use backup registers for state that must survive.
- **Persist Wh totals to internal flash** if you need them to survive a full power loss (battery removal with no VBAT). RTC backup registers only survive when VBAT or VDD is present.
- **`printf` float support** requires `-u _printf_float` in CubeIDE linker flags (newlib-nano build).

## What next

After working through these ten examples:

- For Arduino-style ESP32 development, see [10_arduino_examples.md](arduino-examples.md) (raw) or [17_arduino_library.md](https://rbamp.com/docs/modules-basic-standard-library-arduino-overview) (library).
- For MicroPython / CircuitPython, see [12_micropython_examples.md](micropython-examples.md) (raw) or [18_micropython_library.md](https://rbamp.com/docs/modules-basic-standard-micropython-examples) (library).
- For native ESP-IDF, see [13_esp_idf_examples.md](esp-idf-examples.md) (raw) or [19_esp_idf_library.md](https://rbamp.com/docs/modules-basic-standard-esp-idf-overview) (library).
- For Raspberry Pi Pico C SDK, see [15_pico_sdk_examples.md](pico-sdk-examples.md).
- For ESPHome integration (declarative YAML), see [`tools/esphome-rbamp/docs/en/`](https://rbamp.com/docs/modules-basic-standard-esphome-overview).
- The formal I2C register specification used here lives in [11_api_reference.md](api-reference.md).
