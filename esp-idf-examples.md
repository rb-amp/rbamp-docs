# 13 · ESP-IDF Examples

This chapter is the ESP-IDF equivalent of [10_arduino_examples.md](arduino-examples.md) and [12_micropython_examples.md](micropython-examples.md): the same ten scenarios, ported to **ESP-IDF 5.x** native C with FreeRTOS, the new I2C master driver, esp-mqtt, esp-netif and the deep-sleep / RTC-memory APIs.

> **These examples talk to rbAmp directly through the raw I2C register API.** They are intended for production-grade native ESP32 firmware where direct control, full FreeRTOS task design, OTA-friendly project layout and minimal memory footprint matter.
>
> **For a more convenient, structured interface — use the `rbamp` ESP-IDF component** (see [chapter 19 · ESP-IDF Library](esp-idf-overview.md)). The component wraps the register-level details behind an opaque `rbamp_handle_t` with named functions (`rbamp_read_voltage()`, `rbamp_latch_period()`, `rbamp_read_period_snapshot()`, etc.), a per-channel Wh accumulator, multi-module helpers, full `idf.py menuconfig` integration via Kconfig (`CONFIG_RBAMP_NACK_RETRY_ATTEMPTS`, `CONFIG_RBAMP_DEFAULT_I2C_FREQ_HZ`, etc.), and the application-level NACK retry + sanity-check discipline mandatory for ESP-IDF v5 `i2c_master` (see the bus-timing and retry guidance in [chapter 3 · API Reference](api-reference.md)). Most user-facing projects should start from the component and fall back to the raw API only when needed.

## ESP-IDF version

Examples are written for **ESP-IDF 5.2+** using the new I2C-master driver in `driver/i2c_master.h`. The legacy `driver/i2c.h` API still works and the helpers are easy to port — see the inline note in the common header.

Targets validated in this chapter: **ESP32**, **ESP32-S3**, **ESP32-C3**. Other targets (ESP32-C6, H2) require no code changes.

## Project layout

All examples share the same minimal project skeleton (one ESP-IDF project per example):

```
example_NN/
├── CMakeLists.txt
├── sdkconfig.defaults
└── main/
    ├── CMakeLists.txt
    ├── rbamp_io.h            ← common helpers (header-only or .c)
    ├── rbamp_io.c
    └── example_NN.c          ← the example itself
```

Top-level `CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.16)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(example_NN)
```

`main/CMakeLists.txt`:

```cmake
idf_component_register(SRCS "example_NN.c" "rbamp_io.c"
                       INCLUDE_DIRS ".")
```

A minimal `sdkconfig.defaults` is shown per example where non-default Kconfig values are required (Wi-Fi credentials, MQTT broker, NVS-encryption etc.). All examples assume the default `CONFIG_FREERTOS_HZ=100` tick rate (10 ms granularity).

Build & flash:

```
idf.py set-target esp32
idf.py menuconfig            # set Wi-Fi creds, MQTT host etc.
idf.py build flash monitor
```

## Examples table of contents

| # | Title | Difficulty | Output | MQTT | DRDY | Multi-module | Bidirectional | Use case |
|:---:|---|:---:|:---:|:---:|:---:|:---:|:---:|---|
| 1 | [Quick read](#example-1--quick-read-minimal) | minimal | UART log | — | — | — | — | Smoke test |
| 2 | [60-second energy meter on OLED](#example-2--60-second-energy-meter-on-oled) | low | SSD1306 | — | — | — | — | Boxed Wh counter |
| 3 | [Multi-module monitor](#example-3--multi-module-monitor) | low | UART log | — | — | yes (3) | — | Whole-home monitoring |
| 4 | [Per-appliance energy tracker (UI3)](#example-4--per-appliance-energy-tracker-ui3) | medium | — | yes | — | — | — | Sub-metering in HA |
| 5 | [Master-side bidirectional on a BASIC module](#example-5--master-side-bidirectional-on-a-basic-module) | medium | UART log | — | yes | — | yes (master) | Solar home on BASIC tier |
| 6 | [Home energy balance](#example-6--home-energy-balance) | high | — | yes | — | yes (3) | yes | Full home balance |
| 7 | [Power-event detection](#example-7--power-event-detection) | medium | UART + SD | — | yes | — | — | Appliance event log |
| 8 | [MQTT publisher with HA Auto-discovery](#example-8--mqtt-publisher-with-home-assistant-auto-discovery) | medium | — | yes (+disco) | — | — | optional | Drop-in HA integration |
| 9 | [Battery-powered remote logger with deep sleep](#example-9--battery-powered-remote-logger-with-deep-sleep) | medium | — | yes | — | — | — | Off-grid / outdoor meter |
| 10 | [Time-of-use (TOU) tariff with SNTP wall-clock](#example-10--time-of-use-tou-tariff-with-sntp-wall-clock) | medium | UART log | yes | — | — | optional | Peak / off-peak tariff metering |

---

## Common helpers (`rbamp_io.h` / `rbamp_io.c`)

These helpers are included by every example. To save space in the rest of this chapter, the `#include "rbamp_io.h"` line and any `rbamp_io_*` calls are shown without re-defining the helpers.

### `rbamp_io.h`

```c
/**
 * @file rbamp_io.h
 * @brief Raw I2C helpers for the rbAmp slave (ESP-IDF 5.x master driver).
 */
#pragma once

#include <stdint.h>
#include <stdbool.h>
#include "esp_err.h"
#include "driver/i2c_master.h"

#ifdef __cplusplus
extern "C" {
#endif

/**
 * @brief  Initialise an I2C master bus on the given pins.
 * @param  port      I2C port number (I2C_NUM_0 or I2C_NUM_1).
 * @param  sda_gpio  SDA pin.
 * @param  scl_gpio  SCL pin.
 * @param  out_bus   Output handle of the created bus.
 * @return ESP_OK on success, an esp_err_t otherwise.
 */
esp_err_t rbamp_io_init_bus(i2c_port_num_t port,
                            int sda_gpio, int scl_gpio,
                            i2c_master_bus_handle_t *out_bus);

/**
 * @brief  Attach an rbAmp slave device to a bus.
 * @param  bus       Bus handle from rbamp_io_init_bus().
 * @param  addr      7-bit slave address (default 0x50).
 * @param  out_dev   Output handle of the device.
 * @return ESP_OK on success.
 */
esp_err_t rbamp_io_add_device(i2c_master_bus_handle_t bus, uint8_t addr,
                              i2c_master_dev_handle_t *out_dev);

/**
 * @brief  Read one byte from a register.
 * @details Per-byte helper. WRITE transactions on rbAmp do not auto-increment
 *          (multi-byte writes must address each byte explicitly). READ
 *          transactions DO auto-increment — burst-read is also valid; see
 *          chapter 11 for the asymmetry rules.
 */
esp_err_t rbamp_io_read_u8(i2c_master_dev_handle_t dev,
                           uint8_t reg, uint8_t *out);

/**
 * @brief  Read a little-endian float32 from four consecutive registers.
 *         Performs four separate read transactions.
 */
esp_err_t rbamp_io_read_float_le(i2c_master_dev_handle_t dev,
                                 uint8_t reg, float *out);

/**
 * @brief  Read a little-endian uint32 from four consecutive registers.
 */
esp_err_t rbamp_io_read_u32_le(i2c_master_dev_handle_t dev,
                               uint8_t reg, uint32_t *out);

/**
 * @brief  Write a single byte to a register.
 */
esp_err_t rbamp_io_write_u8(i2c_master_dev_handle_t dev,
                            uint8_t reg, uint8_t val);

/**
 * @brief  Broadcast CMD_LATCH_PERIOD to every rbAmp with GC reception
 *         enabled (FLEET_CONFIG.bit0 = 1; opt-in, default OFF).
 * @details Canonical 5-byte general-call frame: A5 27 group tick_lo tick_hi.
 *          Slaves reject any first byte != 0xA5. See chapter 11 §6.3.2.
 * @param  bus     Bus handle (broadcast — no per-device handle).
 * @param  tick16  Caller's 16-bit window counter (mirrored at GC_TICK 0x59).
 * @param  group   GROUP_ID filter (0 = all-call).
 * @return ESP_OK on ACK (at least one slave accepted).
 *         ESP_ERR_NOT_FOUND on bus-level NACK — fall back to per-module latch.
 */
esp_err_t rbamp_io_broadcast_latch(i2c_master_bus_handle_t bus,
                                   uint16_t tick16, uint8_t group);

/**
 * @brief  Block (with timeout) until DATA_VALID bit 0 of register 0xCE reads 1.
 * @param  dev          Device handle.
 * @param  timeout_ms   Maximum time to wait.
 * @return ESP_OK if ready in time; ESP_ERR_TIMEOUT otherwise.
 */
esp_err_t rbamp_io_wait_ready(i2c_master_dev_handle_t dev, uint32_t timeout_ms);

/* ===== rbAmp register addresses (subset used in examples) ===== */
#define RB_REG_STATUS           0x00
#define RB_REG_COMMAND          0x01
#define RB_REG_VERSION          0x03
#define RB_REG_AC_FREQ          0x20
#define RB_REG_PERIOD_VALID     0x07
#define RB_REG_U_RMS            0x86
#define RB_REG_I0_RMS           0x8E
#define RB_REG_I1_RMS           0x92
#define RB_REG_I2_RMS           0x96
#define RB_REG_P0_REAL          0xA6
#define RB_REG_PF0              0xB2
#define RB_REG_Q0               0xD0
#define RB_REG_DATA_VALID       0xCE
#define RB_REG_PERIOD_AVG_P0    0xDC
#define RB_REG_PERIOD_AVG_P1    0xC2
#define RB_REG_PERIOD_AVG_P2    0xC6
#define RB_REG_PERIOD_MAX_P     0xE0
#define RB_REG_PERIOD_LATCH_MS  0xEC

/* ===== Commands ===== */
#define RB_CMD_RESET            0x01
#define RB_CMD_SAVE_GAINS       0x26
#define RB_CMD_LATCH_PERIOD     0x27

#ifdef __cplusplus
}
#endif
```

### `rbamp_io.c`

```c
#include "rbamp_io.h"
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"

static const char *TAG = "rbamp_io";

/* I2C transaction timeout. rbAmp processes all reads in ISR/main-loop within
 * a few hundred microseconds, so 50 ms is plenty. */
#define RB_I2C_TIMEOUT_MS  50
#define RB_I2C_SPEED_HZ    50000

esp_err_t rbamp_io_init_bus(i2c_port_num_t port,
                            int sda_gpio, int scl_gpio,
                            i2c_master_bus_handle_t *out_bus) {
    i2c_master_bus_config_t bus_cfg = {
        .i2c_port           = port,
        .sda_io_num         = sda_gpio,
        .scl_io_num         = scl_gpio,
        .clk_source         = I2C_CLK_SRC_DEFAULT,
        .glitch_ignore_cnt  = 7,
        .flags.enable_internal_pullup = false, // pull-ups live on the rbAmp board
    };
    return i2c_new_master_bus(&bus_cfg, out_bus);
}

esp_err_t rbamp_io_add_device(i2c_master_bus_handle_t bus, uint8_t addr,
                              i2c_master_dev_handle_t *out_dev) {
    i2c_device_config_t dev_cfg = {
        .dev_addr_length = I2C_ADDR_BIT_LEN_7,
        .device_address  = addr,
        .scl_speed_hz    = RB_I2C_SPEED_HZ,
    };
    return i2c_master_bus_add_device(bus, &dev_cfg, out_dev);
}

esp_err_t rbamp_io_read_u8(i2c_master_dev_handle_t dev,
                           uint8_t reg, uint8_t *out) {
    return i2c_master_transmit_receive(dev, &reg, 1, out, 1, RB_I2C_TIMEOUT_MS);
}

esp_err_t rbamp_io_read_float_le(i2c_master_dev_handle_t dev,
                                 uint8_t reg, float *out) {
    uint8_t buf[4];
    for (int i = 0; i < 4; i++) {
        uint8_t r = reg + i;
        esp_err_t err = i2c_master_transmit_receive(dev, &r, 1, &buf[i], 1,
                                                     RB_I2C_TIMEOUT_MS);
        if (err != ESP_OK) return err;
    }
    memcpy(out, buf, sizeof(float));  // host is little-endian (xtensa, RISC-V)
    return ESP_OK;
}

esp_err_t rbamp_io_read_u32_le(i2c_master_dev_handle_t dev,
                               uint8_t reg, uint32_t *out) {
    uint8_t buf[4];
    for (int i = 0; i < 4; i++) {
        uint8_t r = reg + i;
        esp_err_t err = i2c_master_transmit_receive(dev, &r, 1, &buf[i], 1,
                                                     RB_I2C_TIMEOUT_MS);
        if (err != ESP_OK) return err;
    }
    *out = (uint32_t)buf[0] | (uint32_t)buf[1] << 8
         | (uint32_t)buf[2] << 16 | (uint32_t)buf[3] << 24;
    return ESP_OK;
}

esp_err_t rbamp_io_write_u8(i2c_master_dev_handle_t dev,
                            uint8_t reg, uint8_t val) {
    uint8_t tx[2] = { reg, val };
    return i2c_master_transmit(dev, tx, 2, RB_I2C_TIMEOUT_MS);
}

esp_err_t rbamp_io_broadcast_latch(i2c_master_bus_handle_t bus,
                                   uint16_t tick16, uint8_t group) {
    /* Canonical 5-byte general-call frame: A5 27 group tick_lo tick_hi.
     * Attach a transient device at address 0x00 for the broadcast write. */
    i2c_master_dev_handle_t gc;
    i2c_device_config_t cfg = {
        .dev_addr_length = I2C_ADDR_BIT_LEN_7,
        .device_address  = 0x00,
        .scl_speed_hz    = RB_I2C_SPEED_HZ,
    };
    esp_err_t err = i2c_master_bus_add_device(bus, &cfg, &gc);
    if (err != ESP_OK) return err;
    uint8_t tx[5] = {
        0xA5,                      // frame magic
        RB_CMD_LATCH_PERIOD,       // opcode 0x27
        group,                     // group_id (0 = all-call)
        (uint8_t)(tick16 & 0xFF),
        (uint8_t)((tick16 >> 8) & 0xFF),
    };
    err = i2c_master_transmit(gc, tx, sizeof tx, RB_I2C_TIMEOUT_MS);
    i2c_master_bus_rm_device(gc);
    /* On bus-level NACK i2c_master_transmit returns ESP_ERR_INVALID_RESPONSE
     * or ESP_FAIL — translate to ESP_ERR_NOT_FOUND so the caller falls back
     * to per-module sequential latch. */
    return (err == ESP_OK) ? ESP_OK : ESP_ERR_NOT_FOUND;
}

esp_err_t rbamp_io_wait_ready(i2c_master_dev_handle_t dev, uint32_t timeout_ms) {
    TickType_t deadline = xTaskGetTickCount() + pdMS_TO_TICKS(timeout_ms);
    while (xTaskGetTickCount() < deadline) {
        uint8_t v = 0;
        if (rbamp_io_read_u8(dev, RB_REG_DATA_VALID, &v) == ESP_OK
            && (v & 0x01)) {
            return ESP_OK;
        }
        vTaskDelay(pdMS_TO_TICKS(50));
    }
    ESP_LOGW(TAG, "rbAmp not ready after %lu ms", (unsigned long)timeout_ms);
    return ESP_ERR_TIMEOUT;
}
```

### Porting to the legacy I2C driver (`driver/i2c.h`)

For projects pinned to ESP-IDF ≤ 5.1, replace the bus / device handles with the classic port-level API:

- `i2c_param_config()` + `i2c_driver_install()` instead of `i2c_new_master_bus`
- `i2c_master_write_read_device()` instead of `i2c_master_transmit_receive`
- `i2c_master_write_to_device()` instead of `i2c_master_transmit`

The function signatures of the helpers stay the same; only their bodies change.

---

## Example 1 — Quick read (minimal)

**Goal**: print U, I, P, PF over UART once per second.
**Hardware**: ESP32 + one rbAmp UI1, I2C on GPIO 21 (SDA) / GPIO 22 (SCL).

`main/example_01.c`:

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "rbamp_io.h"

static const char *TAG = "rbamp_quickread";

#define I2C_PORT   I2C_NUM_0
#define PIN_SDA    21
#define PIN_SCL    22
#define RB_ADDR    0x50

void app_main(void) {
    // 1) Bring up the I2C bus and attach the slave.
    i2c_master_bus_handle_t bus;
    i2c_master_dev_handle_t dev;
    ESP_ERROR_CHECK(rbamp_io_init_bus(I2C_PORT, PIN_SDA, PIN_SCL, &bus));
    ESP_ERROR_CHECK(rbamp_io_add_device(bus, RB_ADDR, &dev));

    // 2) Wait until the first RT window has been computed.
    ESP_ERROR_CHECK(rbamp_io_wait_ready(dev, 2000));

    uint8_t ver = 0;
    // VERSION read is one-shot at startup; ESP_ERROR_CHECK is acceptable here.
    if (rbamp_io_read_u8(dev, RB_REG_VERSION, &ver) == ESP_OK) {
        ESP_LOGI(TAG, "rbAmp version: 0x%02X", ver);
    }

    // 3) Read loop. Do NOT wrap per-sample I2C reads in ESP_ERROR_CHECK —
    // a transient NACK on a long bus would abort() and reboot the chip.
    while (1) {
        float u = 0, i_ = 0, p = 0, pf = 0;
        esp_err_t e = ESP_OK;
        e |= rbamp_io_read_float_le(dev, RB_REG_U_RMS,   &u );
        e |= rbamp_io_read_float_le(dev, RB_REG_I0_RMS,  &i_);
        e |= rbamp_io_read_float_le(dev, RB_REG_P0_REAL, &p );
        e |= rbamp_io_read_float_le(dev, RB_REG_PF0,     &pf);
        if (e == ESP_OK) {
            ESP_LOGI(TAG, "U=%.1fV  I=%.3fA  P=%+.1fW  PF=%+.3f", u, i_, p, pf);
        } else {
            ESP_LOGW(TAG, "I2C read failed (e=0x%x) — will retry", e);
        }
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

Expected log output (60 W incandescent on 230 V):

```
I (1234) rbamp_quickread: rbAmp version: 0x04
I (2236) rbamp_quickread: U=230.1V  I=0.262A  P=+60.2W  PF=+0.998
I (3238) rbamp_quickread: U=229.9V  I=0.262A  P=+60.1W  PF=+0.998
...
```

---

## Example 2 — 60-second energy meter on OLED

**Goal**: Wh counter refreshed once per minute, shown on a 128×64 SSD1306 sharing the rbAmp I2C bus.
**Hardware**: ESP32 + rbAmp + SSD1306 OLED (at `0x3C`) on the same bus.
**Accounting**: unidirectional (BASIC tier).
**Library**: any IDF SSD1306 component (e.g. `nopnop2002/esp-idf-ssd1306`). For brevity the OLED API below is symbolic — replace with your component's calls.

```c
#include <stdio.h>
#include <inttypes.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "esp_timer.h"
#include "rbamp_io.h"
#include "ssd1306.h"  // your driver of choice

static const char *TAG = "rbamp_oled";

#define RB_ADDR  0x50

void app_main(void) {
    i2c_master_bus_handle_t bus;
    i2c_master_dev_handle_t dev;
    ESP_ERROR_CHECK(rbamp_io_init_bus(I2C_NUM_0, 21, 22, &bus));
    ESP_ERROR_CHECK(rbamp_io_add_device(bus, RB_ADDR, &dev));

    // OLED handle — initialised on the same bus.
    SSD1306_t oled = {0};
    i2c_master_init_external(&oled, /* bus = */ I2C_NUM_0, /* addr = */ 0x3C);
    ssd1306_init(&oled, 128, 64);

    ESP_ERROR_CHECK(rbamp_io_wait_ready(dev, 2000));

    // Primer latch — the snapshot it produces is discarded.
    ESP_ERROR_CHECK(rbamp_io_write_u8(dev, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD));
    int64_t t_prev_us = esp_timer_get_time();
    double total_wh = 0.0;

    while (1) {
        vTaskDelay(pdMS_TO_TICKS(60 * 1000));

        // 1) Final latch — closes the just-elapsed period.
        ESP_ERROR_CHECK(rbamp_io_write_u8(dev, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD));
        int64_t t_now_us = esp_timer_get_time();
        vTaskDelay(pdMS_TO_TICKS(50));  // let firmware finalise the snapshot

        // 2) Check PERIOD_VALID before trusting the data.
        uint8_t valid = 0;
        ESP_ERROR_CHECK(rbamp_io_read_u8(dev, RB_REG_PERIOD_VALID, &valid));
        if ((valid & 0x01) == 0) {
            ESP_LOGW(TAG, "Stale snapshot — skipping");
            t_prev_us = esp_timer_get_time();
            continue;
        }

        // 3) Read time-averaged P for channel 0.
        float avg_p = 0.0f;
        ESP_ERROR_CHECK(rbamp_io_read_float_le(dev, RB_REG_PERIOD_AVG_P0, &avg_p));

        // 4) Energy uses the MASTER clock.
        float dt_s = (float)(t_now_us - t_prev_us) / 1000000.0f;
        double e_wh = (double)avg_p * dt_s / 3600.0;
        total_wh += e_wh;
        t_prev_us = t_now_us;

        // 5) Render on OLED.
        char line[32];
        ssd1306_clear_screen(&oled, false);
        snprintf(line, sizeof(line), "P:  %6.1f W", avg_p);   ssd1306_display_text(&oled, 0, line, strlen(line), false);
        snprintf(line, sizeof(line), "dt: %6.1f s", dt_s);    ssd1306_display_text(&oled, 2, line, strlen(line), false);
        snprintf(line, sizeof(line), "%.2f Wh", total_wh);    ssd1306_display_text_x3(&oled, 4, line, strlen(line), false);

        ESP_LOGI(TAG, "avg_P=%.1f  E_period=%.3f  total=%.3f", avg_p, e_wh, total_wh);
    }
}
```

> If your SSD1306 component owns its own I2C device, point it at the **same `i2c_master_bus_handle_t`** so both rbAmp and OLED share a single bus driver — never install two drivers on the same port.

---

## Example 3 — Multi-module monitor

**Goal**: poll three modules (main feed, water heater, AC) and print the total.
**Hardware**: ESP32 + 3 × rbAmp UI1 at `0x50`, `0x51`, `0x52`.

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "rbamp_io.h"

static const char *TAG = "rbamp_multi";

typedef struct {
    uint8_t addr;
    const char *label;
    i2c_master_dev_handle_t dev;
} module_t;

static module_t s_modules[] = {
    { 0x50, "Mains ", NULL },
    { 0x51, "Boiler", NULL },
    { 0x52, "AC    ", NULL },
};
static const int N_MODULES = sizeof(s_modules) / sizeof(s_modules[0]);

/**
 * @brief  Probe a slave with a 1-byte read to confirm presence.
 */
static bool module_probe(i2c_master_bus_handle_t bus, uint8_t addr) {
    return i2c_master_probe(bus, addr, 100) == ESP_OK;
}

void app_main(void) {
    i2c_master_bus_handle_t bus;
    ESP_ERROR_CHECK(rbamp_io_init_bus(I2C_NUM_0, 21, 22, &bus));

    // Verify every module is present and attach a device handle.
    for (int i = 0; i < N_MODULES; i++) {
        if (!module_probe(bus, s_modules[i].addr)) {
            ESP_LOGE(TAG, "%s at 0x%02X not found — halting",
                     s_modules[i].label, s_modules[i].addr);
            while (1) vTaskDelay(portMAX_DELAY);
        }
        ESP_ERROR_CHECK(rbamp_io_add_device(bus, s_modules[i].addr,
                                            &s_modules[i].dev));
    }
    ESP_LOGI(TAG, "All modules OK");

    while (1) {
        float total_p = 0.0f;
        ESP_LOGI(TAG, "---");
        for (int i = 0; i < N_MODULES; i++) {
            float u = 0.0f, p = 0.0f;
            rbamp_io_read_float_le(s_modules[i].dev, RB_REG_U_RMS,   &u);
            rbamp_io_read_float_le(s_modules[i].dev, RB_REG_P0_REAL, &p);
            ESP_LOGI(TAG, "[%s] U=%.1f  P=%+.1f W", s_modules[i].label, u, p);
            total_p += p;
        }
        ESP_LOGI(TAG, "TOTAL: %.1f W", total_p);
        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}
```

`i2c_master_probe()` is the IDF 5.x equivalent of an Arduino `Wire.beginTransmission()` / `endTransmission()` ping — perfect for a startup bus scan.

---

## Example 4 — Per-appliance energy tracker (UI3)

**Goal**: a UI3 module with three CT clamps on three different loads on the same phase; independent Wh counters published to MQTT every minute.
**Hardware**: ESP32 + 1 × rbAmp UI3 + Wi-Fi + MQTT broker.
**Components**: `esp_wifi`, `esp_event`, `esp_netif`, `mqtt_client` (built-in `esp-mqtt`).

`sdkconfig.defaults` (relevant):

```
CONFIG_ESP_WIFI_SSID="ssid"
CONFIG_ESP_WIFI_PASSWORD="password"
CONFIG_BROKER_URL="mqtt://192.168.1.10"
```

`main/example_04.c`:

```c
#include <inttypes.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "esp_timer.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "nvs_flash.h"
#include "mqtt_client.h"
#include "rbamp_io.h"

static const char *TAG = "rbamp_ui3";
#define RB_ADDR 0x50
static const char *CH_NAMES[3] = { "main", "heatpump", "lights" };

/* Shared MQTT client handle. */
static esp_mqtt_client_handle_t s_mqtt;

/* ---------- Wi-Fi station bring-up (minimal) ---------- */
static EventGroupHandle_t s_wifi_evt;
#define WIFI_BIT_CONNECTED BIT0

static void wifi_evt_handler(void *arg, esp_event_base_t base,
                             int32_t id, void *data) {
    if (base == WIFI_EVENT && id == WIFI_EVENT_STA_DISCONNECTED) {
        esp_wifi_connect();
    } else if (base == IP_EVENT && id == IP_EVENT_STA_GOT_IP) {
        xEventGroupSetBits(s_wifi_evt, WIFI_BIT_CONNECTED);
    }
}

static void wifi_init_sta(void) {
    // First-boot or partition mismatch can return ESP_ERR_NVS_NO_FREE_PAGES /
    // ESP_ERR_NVS_NEW_VERSION_FOUND — recover by erasing and re-initialising.
    esp_err_t err = nvs_flash_init();
    if (err == ESP_ERR_NVS_NO_FREE_PAGES ||
        err == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        err = nvs_flash_init();
    }
    ESP_ERROR_CHECK(err);
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_netif_create_default_wifi_sta();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    s_wifi_evt = xEventGroupCreate();
    ESP_ERROR_CHECK(esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID,
                                                wifi_evt_handler, NULL));
    ESP_ERROR_CHECK(esp_event_handler_register(IP_EVENT, IP_EVENT_STA_GOT_IP,
                                                wifi_evt_handler, NULL));

    wifi_config_t wcfg = {
        .sta = {
            .ssid     = CONFIG_ESP_WIFI_SSID,
            .password = CONFIG_ESP_WIFI_PASSWORD,
        },
    };
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wcfg));
    ESP_ERROR_CHECK(esp_wifi_start());
    xEventGroupWaitBits(s_wifi_evt, WIFI_BIT_CONNECTED, pdFALSE, pdTRUE,
                        portMAX_DELAY);
    ESP_LOGI(TAG, "Wi-Fi connected");
}

/* ---------- MQTT bring-up ---------- */
static void mqtt_init(void) {
    esp_mqtt_client_config_t cfg = {
        .broker.address.uri = CONFIG_BROKER_URL,
    };
    s_mqtt = esp_mqtt_client_init(&cfg);
    ESP_ERROR_CHECK(esp_mqtt_client_start(s_mqtt));
}

/**
 * @brief  Publish one channel's state to MQTT.
 */
static void mqtt_publish_channel(const char *name, float avg_p, double e_wh) {
    char topic[64], payload[64];
    snprintf(topic,   sizeof(topic),   "rbamp/%s/state", name);
    snprintf(payload, sizeof(payload), "{\"power\":%.1f,\"energy\":%.4f}",
             avg_p, e_wh);
    esp_mqtt_client_publish(s_mqtt, topic, payload, 0, /*qos*/ 0, /*retain*/ 0);
}

void app_main(void) {
    wifi_init_sta();
    mqtt_init();

    i2c_master_bus_handle_t bus;
    i2c_master_dev_handle_t dev;
    ESP_ERROR_CHECK(rbamp_io_init_bus(I2C_NUM_0, 21, 22, &bus));
    ESP_ERROR_CHECK(rbamp_io_add_device(bus, RB_ADDR, &dev));
    ESP_ERROR_CHECK(rbamp_io_wait_ready(dev, 2000));

    ESP_ERROR_CHECK(rbamp_io_write_u8(dev, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD));
    int64_t t_prev_us = esp_timer_get_time();
    double total_wh[3] = {0.0, 0.0, 0.0};

    while (1) {
        vTaskDelay(pdMS_TO_TICKS(60 * 1000));

        ESP_ERROR_CHECK(rbamp_io_write_u8(dev, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD));
        int64_t t_now_us = esp_timer_get_time();
        vTaskDelay(pdMS_TO_TICKS(50));

        uint8_t valid = 0;
        rbamp_io_read_u8(dev, RB_REG_PERIOD_VALID, &valid);
        if ((valid & 0x01) == 0) {
            ESP_LOGW(TAG, "stale snapshot");
            t_prev_us = esp_timer_get_time();
            continue;
        }

        // PERIOD_AVG_P_W per channel: I0 at 0xDC, I1 at 0xC2, I2 at 0xC6.
        float avg_p[3];
        rbamp_io_read_float_le(dev, RB_REG_PERIOD_AVG_P0, &avg_p[0]);
        rbamp_io_read_float_le(dev, RB_REG_PERIOD_AVG_P1, &avg_p[1]);
        rbamp_io_read_float_le(dev, RB_REG_PERIOD_AVG_P2, &avg_p[2]);

        float dt_s = (float)(t_now_us - t_prev_us) / 1000000.0f;
        t_prev_us = t_now_us;

        for (int ch = 0; ch < 3; ch++) {
            double e = (double)avg_p[ch] * dt_s / 3600.0;
            total_wh[ch] += e;
            ESP_LOGI(TAG, "[%s] avg_P=%+.1f W  E_total=%.3f Wh",
                     CH_NAMES[ch], avg_p[ch], total_wh[ch]);
            mqtt_publish_channel(CH_NAMES[ch], avg_p[ch], total_wh[ch]);
        }
    }
}
```

---

## Example 5 — Master-side bidirectional on a BASIC module

**Goal**: implement bidirectional accounting on the master while the module is BASIC firmware (period accumulator clamps negative samples). The ESP32 reacts to the DRDY falling edge via a GPIO ISR and splits consumption / export.
**Hardware**: ESP32 + rbAmp UI1 with DRDY wired to GPIO 15.

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "driver/gpio.h"
#include "esp_log.h"
#include "esp_timer.h"
#include "rbamp_io.h"

static const char *TAG = "rbamp_bidir";

#define RB_ADDR      0x50
#define PIN_DRDY     15

static QueueHandle_t s_drdy_q;

/**
 * @brief  GPIO ISR for DRDY falling edge.
 * @details Pushes a sentinel into a queue. All real work happens in the task
 *          (no I2C inside an ISR).
 */
static void IRAM_ATTR drdy_isr(void *arg) {
    uint8_t sentinel = 1;
    BaseType_t hp_woken = pdFALSE;
    xQueueSendFromISR(s_drdy_q, &sentinel, &hp_woken);
    if (hp_woken) portYIELD_FROM_ISR();
}

/**
 * @brief  Worker task that integrates RT P_real into consumption / export buckets.
 */
static void accumulator_task(void *arg) {
    i2c_master_dev_handle_t dev = (i2c_master_dev_handle_t)arg;
    double e_consumed_wh = 0.0;
    double e_exported_wh = 0.0;
    int64_t t_last_us = esp_timer_get_time();
    int64_t t_last_print_us = t_last_us;

    while (1) {
        uint8_t sentinel = 0;
        if (xQueueReceive(s_drdy_q, &sentinel, portMAX_DELAY) != pdTRUE) continue;

        int64_t t_now_us = esp_timer_get_time();
        float p = 0.0f;
        rbamp_io_read_float_le(dev, RB_REG_P0_REAL, &p);    // signed P
        float dt_s = (float)(t_now_us - t_last_us) / 1000000.0f;
        t_last_us = t_now_us;

        if (p > 0) {
            e_consumed_wh += p * dt_s / 3600.0;
        } else {
            e_exported_wh += -p * dt_s / 3600.0;
        }

        // Print summary every 5 s, not on every RT window.
        if (t_now_us - t_last_print_us >= 5LL * 1000000LL) {
            t_last_print_us = t_now_us;
            ESP_LOGI(TAG,
                "P=%+8.1fW   consumed=%.4f Wh  exported=%.4f Wh  net=%+.4f Wh",
                p, e_consumed_wh, e_exported_wh,
                e_consumed_wh - e_exported_wh);
        }
    }
}

void app_main(void) {
    i2c_master_bus_handle_t bus;
    i2c_master_dev_handle_t dev;
    ESP_ERROR_CHECK(rbamp_io_init_bus(I2C_NUM_0, 21, 22, &bus));
    ESP_ERROR_CHECK(rbamp_io_add_device(bus, RB_ADDR, &dev));
    ESP_ERROR_CHECK(rbamp_io_wait_ready(dev, 2000));

    // DRDY input with internal pull-up; falling-edge ISR.
    gpio_config_t io = {
        .pin_bit_mask = 1ULL << PIN_DRDY,
        .mode         = GPIO_MODE_INPUT,
        .pull_up_en   = GPIO_PULLUP_ENABLE,
        .intr_type    = GPIO_INTR_NEGEDGE,
    };
    ESP_ERROR_CHECK(gpio_config(&io));

    s_drdy_q = xQueueCreate(8, sizeof(uint8_t));
    ESP_ERROR_CHECK(gpio_install_isr_service(ESP_INTR_FLAG_IRAM));
    ESP_ERROR_CHECK(gpio_isr_handler_add(PIN_DRDY, drdy_isr, NULL));

    xTaskCreate(accumulator_task, "rb_acc", 4096, dev, 5, NULL);

    ESP_LOGI(TAG, "Bidirectional accumulator started");
    vTaskSuspend(NULL);   // app_main idle
}
```

**Notes**:

- The ISR only pushes a sentinel into a queue (`xQueueSendFromISR`). All I2C work happens in the worker task — safer and avoids holding locks in ISR context.
- For ESP-IDF 5.x prefer `gpio_install_isr_service()` once + `gpio_isr_handler_add()` per pin (instead of a single global handler).

---

## Example 6 — Home energy balance

**Goal**: full balance dashboard. Three modules:
- rbAmp #1 on the main feed (STANDARD / PRO — bidirectional)
- rbAmp #2 on the solar inverter output (BASIC ok)
- rbAmp #3 UI3 on three large loads (HP, AC, EV)

Master uses `rbamp_io_broadcast_latch(bus)` for synchronous snapshots and publishes a single retained balance to MQTT.

```c
#include <inttypes.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "esp_timer.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "nvs_flash.h"
#include "mqtt_client.h"
#include "rbamp_io.h"

static const char *TAG = "rbamp_balance";

#define ADDR_MAINS  0x50    // bidirectional (STANDARD / PRO)
#define ADDR_SOLAR  0x51    // generation only
#define ADDR_LOADS  0x52    // UI3

/* PERIOD_AVG_P_NEG address for the mains module — see the SKU datasheet
 * of the STANDARD / PRO product. Set to 0xFF on BASIC modules. */
#define REG_PERIOD_AVG_P_NEG_MAINS  0xFF

extern void wifi_init_sta(void);              // reuse Ex. 4 boilerplate
extern esp_mqtt_client_handle_t mqtt_start(void);

void app_main(void) {
    wifi_init_sta();
    esp_mqtt_client_handle_t mqtt = mqtt_start();

    i2c_master_bus_handle_t bus;
    ESP_ERROR_CHECK(rbamp_io_init_bus(I2C_NUM_0, 21, 22, &bus));

    i2c_master_dev_handle_t mains, solar, loads;
    ESP_ERROR_CHECK(rbamp_io_add_device(bus, ADDR_MAINS, &mains));
    ESP_ERROR_CHECK(rbamp_io_add_device(bus, ADDR_SOLAR, &solar));
    ESP_ERROR_CHECK(rbamp_io_add_device(bus, ADDR_LOADS, &loads));

    ESP_ERROR_CHECK(rbamp_io_wait_ready(mains, 2000));
    ESP_ERROR_CHECK(rbamp_io_wait_ready(solar, 2000));
    ESP_ERROR_CHECK(rbamp_io_wait_ready(loads, 2000));

    // Primer latch on all modules (general call). GC reception is opt-in
    // (FLEET_CONFIG.bit0, default OFF) — fall back to per-module on NACK.
    uint16_t gc_tick = 0;
    if (rbamp_io_broadcast_latch(bus, gc_tick++, 0) != ESP_OK) {
        rbamp_io_write_u8(mains, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
        rbamp_io_write_u8(solar, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
        rbamp_io_write_u8(loads, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
    }
    int64_t t_prev_us = esp_timer_get_time();

    struct {
        double mains_in_wh, mains_out_wh, solar_total_wh;
        double loads_wh[3];
    } totals = {0};

    ESP_LOGI(TAG, "Home balance started");

    while (1) {
        vTaskDelay(pdMS_TO_TICKS(60 * 1000));

        // Synchronous latch — all three modules snapshot at the same instant.
        if (rbamp_io_broadcast_latch(bus, gc_tick++, 0) != ESP_OK) {
            rbamp_io_write_u8(mains, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
            rbamp_io_write_u8(solar, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
            rbamp_io_write_u8(loads, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
        }
        int64_t t_now_us = esp_timer_get_time();
        vTaskDelay(pdMS_TO_TICKS(50));

        float dt_s = (float)(t_now_us - t_prev_us) / 1000000.0f;
        t_prev_us = t_now_us;

        /* ----- MAINS (bidirectional, STANDARD/PRO) ----- */
        uint8_t valid;
        if (rbamp_io_read_u8(mains, RB_REG_PERIOD_VALID, &valid) == ESP_OK
            && (valid & 0x01)) {
            float p_consume = 0.0f, p_export = 0.0f;
            rbamp_io_read_float_le(mains, RB_REG_PERIOD_AVG_P0, &p_consume);
            if (REG_PERIOD_AVG_P_NEG_MAINS != 0xFF) {
                rbamp_io_read_float_le(mains, REG_PERIOD_AVG_P_NEG_MAINS, &p_export);
            }
            totals.mains_in_wh  += p_consume * dt_s / 3600.0;
            totals.mains_out_wh += p_export  * dt_s / 3600.0;
        }

        /* ----- SOLAR (generation only) -----
         * CT arrow points toward the home → P > 0 = inverter exporting. */
        if (rbamp_io_read_u8(solar, RB_REG_PERIOD_VALID, &valid) == ESP_OK
            && (valid & 0x01)) {
            float p = 0.0f;
            rbamp_io_read_float_le(solar, RB_REG_PERIOD_AVG_P0, &p);
            totals.solar_total_wh += p * dt_s / 3600.0;
        }

        /* ----- LOADS (UI3, three channels) ----- */
        if (rbamp_io_read_u8(loads, RB_REG_PERIOD_VALID, &valid) == ESP_OK
            && (valid & 0x01)) {
            float p[3] = {0};
            rbamp_io_read_float_le(loads, RB_REG_PERIOD_AVG_P0, &p[0]); // I0
            rbamp_io_read_float_le(loads, RB_REG_PERIOD_AVG_P1, &p[1]); // I1
            rbamp_io_read_float_le(loads, RB_REG_PERIOD_AVG_P2, &p[2]); // I2
            for (int i = 0; i < 3; i++) {
                totals.loads_wh[i] += p[i] * dt_s / 3600.0;
            }
        }

        double total_consumed = totals.mains_in_wh + totals.solar_total_wh
                              - totals.mains_out_wh;
        double solar_self_used = totals.solar_total_wh - totals.mains_out_wh;
        if (solar_self_used < 0.0) solar_self_used = 0.0;

        ESP_LOGI(TAG, "MAINS  in=%.2f  out=%.2f",
                 totals.mains_in_wh, totals.mains_out_wh);
        ESP_LOGI(TAG, "SOLAR  gen=%.2f  self-used=%.2f  exported=%.2f",
                 totals.solar_total_wh, solar_self_used, totals.mains_out_wh);
        ESP_LOGI(TAG, "LOADS  HP=%.2f  AC=%.2f  EV=%.2f",
                 totals.loads_wh[0], totals.loads_wh[1], totals.loads_wh[2]);
        ESP_LOGI(TAG, "TOTAL  household consumed=%.2f Wh", total_consumed);

        char payload[512];
        snprintf(payload, sizeof(payload),
            "{\"mains_in\":%.3f,\"mains_out\":%.3f,\"solar\":%.3f,"
            "\"self_used\":%.3f,\"total_consumed\":%.3f,"
            "\"hp\":%.3f,\"ac\":%.3f,\"ev\":%.3f}",
            totals.mains_in_wh, totals.mains_out_wh, totals.solar_total_wh,
            solar_self_used, total_consumed,
            totals.loads_wh[0], totals.loads_wh[1], totals.loads_wh[2]);
        esp_mqtt_client_publish(mqtt, "home/energy/balance",
                                payload, 0, /*qos*/ 1, /*retain*/ 1);
    }
}
```

> `wifi_init_sta()` and `mqtt_start()` are the same boilerplate as Example 4 — kept as `extern` to avoid duplication. In a real project place them in `net_helpers.c` and link via `idf_component_register(... PRIV_REQUIRES esp_wifi esp_event nvs_flash mqtt esp_netif ...)`.

---

## Example 7 — Power-event detection

**Goal**: on every DRDY edge, compare instantaneous P to an EMA and log significant deviations to an SD card mounted via SPI.
**Hardware**: ESP32 + rbAmp + SPI SD-card module (FATFS via `vfs_fat_sdmmc`).

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "driver/gpio.h"
#include "driver/spi_common.h"
#include "esp_log.h"
#include "esp_timer.h"
#include "esp_vfs_fat.h"
#include "sdmmc_cmd.h"
#include "rbamp_io.h"

static const char *TAG = "rbamp_event";
#define RB_ADDR    0x50
#define PIN_DRDY   15
#define MOUNT_POINT "/sdcard"

#define EMA_ALPHA           0.05f   // ~4 s time constant at 5 Hz updates
#define EVENT_THRESHOLD_W   200.0f

static QueueHandle_t s_drdy_q;
static FILE *s_logfile = NULL;

static void IRAM_ATTR drdy_isr(void *arg) {
    uint8_t s = 1;
    BaseType_t hp = pdFALSE;
    xQueueSendFromISR(s_drdy_q, &s, &hp);
    if (hp) portYIELD_FROM_ISR();
}

/**
 * @brief  Mount an SPI SD card under MOUNT_POINT using FATFS.
 */
static esp_err_t mount_sd_card(void) {
    sdmmc_host_t host = SDSPI_HOST_DEFAULT();
    spi_bus_config_t buscfg = {
        .mosi_io_num = 23, .miso_io_num = 19, .sclk_io_num = 18,
        .quadwp_io_num = -1, .quadhd_io_num = -1, .max_transfer_sz = 4000,
    };
    ESP_ERROR_CHECK(spi_bus_initialize(host.slot, &buscfg, SDSPI_DEFAULT_DMA));

    sdspi_device_config_t slot = SDSPI_DEVICE_CONFIG_DEFAULT();
    slot.gpio_cs = 5;
    slot.host_id = host.slot;

    esp_vfs_fat_sdmmc_mount_config_t mount_cfg = {
        .format_if_mount_failed = false, .max_files = 4, .allocation_unit_size = 16 * 1024,
    };
    sdmmc_card_t *card;
    return esp_vfs_fat_sdspi_mount(MOUNT_POINT, &host, &slot, &mount_cfg, &card);
}

static void event_task(void *arg) {
    i2c_master_dev_handle_t dev = (i2c_master_dev_handle_t)arg;

    // Seed the EMA with the current power so we don't log a spurious startup event.
    float p_ema = 0.0f;
    rbamp_io_read_float_le(dev, RB_REG_P0_REAL, &p_ema);

    while (1) {
        uint8_t s; xQueueReceive(s_drdy_q, &s, portMAX_DELAY);
        float p = 0.0f;
        rbamp_io_read_float_le(dev, RB_REG_P0_REAL, &p);

        float delta = p - p_ema;
        p_ema = (1.0f - EMA_ALPHA) * p_ema + EMA_ALPHA * p;

        if (fabsf(delta) > EVENT_THRESHOLD_W) {
            const char *type = (delta > 0) ? "TURN_ON" : "TURN_OFF";
            ESP_LOGI(TAG, "%s  delta=%+.1f W  P=%.1f W  EMA=%.1f W",
                     type, delta, p, p_ema);
            if (s_logfile) {
                fprintf(s_logfile,
                        "%" PRId64 "  %s  delta=%+.1f W  P=%.1f W  EMA=%.1f W\n",
                        esp_timer_get_time() / 1000, type, delta, p, p_ema);
                fflush(s_logfile);   // ensure the line hits the card
            }
        }
    }
}

void app_main(void) {
    i2c_master_bus_handle_t bus;
    i2c_master_dev_handle_t dev;
    ESP_ERROR_CHECK(rbamp_io_init_bus(I2C_NUM_0, 21, 22, &bus));
    ESP_ERROR_CHECK(rbamp_io_add_device(bus, RB_ADDR, &dev));
    ESP_ERROR_CHECK(rbamp_io_wait_ready(dev, 2000));

    if (mount_sd_card() == ESP_OK) {
        s_logfile = fopen(MOUNT_POINT "/events.log", "a");
        if (s_logfile) ESP_LOGI(TAG, "Logging to %s/events.log", MOUNT_POINT);
    }

    gpio_config_t io = {
        .pin_bit_mask = 1ULL << PIN_DRDY,
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_ENABLE,
        .intr_type = GPIO_INTR_NEGEDGE,
    };
    ESP_ERROR_CHECK(gpio_config(&io));
    s_drdy_q = xQueueCreate(16, sizeof(uint8_t));
    ESP_ERROR_CHECK(gpio_install_isr_service(ESP_INTR_FLAG_IRAM));
    ESP_ERROR_CHECK(gpio_isr_handler_add(PIN_DRDY, drdy_isr, NULL));

    xTaskCreate(event_task, "rb_evt", 6144, dev, 5, NULL);
    ESP_LOGI(TAG, "Event detector started");
    vTaskSuspend(NULL);
}
```

---

## Example 8 — MQTT publisher with Home Assistant Auto-discovery

**Goal**: a drop-in HA integration. The ESP32 connects to Wi-Fi and an MQTT broker, publishes HA Auto-discovery configs (retained) and emits a JSON state every minute.
**Hardware**: ESP32 + one rbAmp UI1 + Wi-Fi + MQTT broker.

```c
#include <inttypes.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "esp_timer.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "nvs_flash.h"
#include "mqtt_client.h"
#include "rbamp_io.h"

static const char *TAG = "rbamp_ha";
#define RB_ADDR     0x50
#define DEVICE_ID   "rbamp_main"
#define DEVICE_NAME "Mains rbAmp"

extern void wifi_init_sta(void);          // reuse Ex. 4 boilerplate

static esp_mqtt_client_handle_t s_mqtt;
static volatile bool s_mqtt_connected = false;

static void mqtt_event_handler(void *arg, esp_event_base_t base,
                               int32_t id, void *data) {
    if (id == MQTT_EVENT_CONNECTED)    s_mqtt_connected = true;
    else if (id == MQTT_EVENT_DISCONNECTED) s_mqtt_connected = false;
}

/**
 * @brief  Publish one HA discovery config (retained).
 */
static void publish_discovery_sensor(const char *key, const char *friendly,
                                     const char *unit, const char *dev_class,
                                     const char *state_class) {
    char topic[128], payload[512];
    snprintf(topic, sizeof(topic),
             "homeassistant/sensor/" DEVICE_ID "/%s/config", key);

    int n = snprintf(payload, sizeof(payload),
        "{"
          "\"name\":\"" DEVICE_NAME " %s\","
          "\"unique_id\":\"" DEVICE_ID "_%s\","
          "\"state_topic\":\"rbamp/" DEVICE_ID "/state\","
          "\"value_template\":\"{{ value_json.%s }}\","
          "\"state_class\":\"%s\","
          "\"device\":{"
            "\"identifiers\":[\"" DEVICE_ID "\"],"
            "\"name\":\"" DEVICE_NAME "\","
            "\"manufacturer\":\"rbAmp\","
            "\"model\":\"rbAmp UI*\""
          "}",
        friendly, key, key, state_class);
    if (unit)      n += snprintf(payload + n, sizeof(payload) - n,
                                  ",\"unit_of_measurement\":\"%s\"", unit);
    if (dev_class) n += snprintf(payload + n, sizeof(payload) - n,
                                  ",\"device_class\":\"%s\"", dev_class);
    snprintf(payload + n, sizeof(payload) - n, "}");

    esp_mqtt_client_publish(s_mqtt, topic, payload, 0, /*qos*/ 1, /*retain*/ 1);
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

void app_main(void) {
    wifi_init_sta();

    // MQTT client.
    esp_mqtt_client_config_t cfg = {
        .broker.address.uri = CONFIG_BROKER_URL,
    };
    s_mqtt = esp_mqtt_client_init(&cfg);
    esp_mqtt_client_register_event(s_mqtt, ESP_EVENT_ANY_ID,
                                    mqtt_event_handler, NULL);
    esp_mqtt_client_start(s_mqtt);

    // Wait for the first CONNECTED event, then publish discovery once.
    while (!s_mqtt_connected) vTaskDelay(pdMS_TO_TICKS(100));
    publish_discovery_all();

    // rbAmp bring-up.
    i2c_master_bus_handle_t bus;
    i2c_master_dev_handle_t dev;
    ESP_ERROR_CHECK(rbamp_io_init_bus(I2C_NUM_0, 21, 22, &bus));
    ESP_ERROR_CHECK(rbamp_io_add_device(bus, RB_ADDR, &dev));
    ESP_ERROR_CHECK(rbamp_io_wait_ready(dev, 2000));

    rbamp_io_write_u8(dev, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);   // primer
    int64_t t_prev_us = esp_timer_get_time();
    double total_wh = 0.0;

    while (1) {
        vTaskDelay(pdMS_TO_TICKS(60 * 1000));

        rbamp_io_write_u8(dev, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
        int64_t t_now_us = esp_timer_get_time();
        vTaskDelay(pdMS_TO_TICKS(50));

        uint8_t valid = 0;
        rbamp_io_read_u8(dev, RB_REG_PERIOD_VALID, &valid);
        if ((valid & 0x01) == 0) {
            t_prev_us = esp_timer_get_time();
            continue;
        }
        float avg_p = 0.0f;
        rbamp_io_read_float_le(dev, RB_REG_PERIOD_AVG_P0, &avg_p);
        float dt_s = (float)(t_now_us - t_prev_us) / 1000000.0f;
        total_wh += (double)avg_p * dt_s / 3600.0;
        t_prev_us = t_now_us;

        // Live RT values for the state payload.
        float u = 0, i_ = 0, pf = 0, q = 0;
        uint8_t freq = 0;
        rbamp_io_read_float_le(dev, RB_REG_U_RMS,  &u);
        rbamp_io_read_float_le(dev, RB_REG_I0_RMS, &i_);
        rbamp_io_read_float_le(dev, RB_REG_PF0,    &pf);
        rbamp_io_read_float_le(dev, RB_REG_Q0,     &q);
        rbamp_io_read_u8     (dev, RB_REG_AC_FREQ, &freq);

        char payload[384];
        snprintf(payload, sizeof(payload),
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
            u, i_, avg_p, total_wh, (unsigned)freq, pf, u * i_, q);

        esp_mqtt_client_publish(s_mqtt, "rbamp/" DEVICE_ID "/state",
                                payload, 0, /*qos*/ 0, /*retain*/ 0);
        ESP_LOGI(TAG, "Published: %s", payload);
    }
}
```

---

## Example 9 — Battery-powered remote logger with deep sleep

**Goal**: an outdoor / off-grid logger that wakes every 10 minutes, latches a period, publishes via MQTT, then re-enters deep sleep. Average current draw a few mA — a single Li-ion cell lasts months.
**Hardware**: ESP32 (or ESP32-S3) + one rbAmp UI1 + Li-ion. The rbAmp can be power-gated through a MOSFET on GPIO 4.
**Persistence**: `RTC_DATA_ATTR` survives deep sleep; `nvs_flash` survives full power loss.

```c
#include <math.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"
#include "esp_timer.h"
#include "esp_sleep.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "nvs_flash.h"
#include "mqtt_client.h"
#include "rbamp_io.h"

static const char *TAG = "rbamp_remote";
#define RB_ADDR    0x50
#define PIN_RBAMP_POWER 4
#define SLEEP_US  (10ULL * 60ULL * 1000ULL * 1000ULL)   // 10-minute wake interval

/* Variables that survive deep sleep. RTC slow memory is preserved while at
 * least one RTC domain is powered.
 * IMPORTANT: after a brown-out, watchdog reset, or any non-deep-sleep boot,
 * RTC slow memory may contain garbage. A magic-marker sentinel lets us
 * distinguish a fresh cold boot from a deep-sleep wake. */
RTC_DATA_ATTR static uint32_t s_rtc_magic   = 0;        /* set to RTC_MAGIC on init */
RTC_DATA_ATTR static double   s_total_wh    = 0.0;
RTC_DATA_ATTR static uint32_t s_wake_count  = 0;
RTC_DATA_ATTR static bool     s_primer_done = false;
RTC_DATA_ATTR static int64_t  s_t_prev_us   = 0;

#define RTC_MAGIC  0xCAFEFEEDu

/**
 * @brief  Power-gate the rbAmp module.
 * @param  on  true = enable, false = disable.
 */
static void rbamp_power(bool on) {
    gpio_set_direction(PIN_RBAMP_POWER, GPIO_MODE_OUTPUT);
    gpio_set_level(PIN_RBAMP_POWER, on ? 1 : 0);
    if (on) vTaskDelay(pdMS_TO_TICKS(300));   // give the LDO + MCU time to boot
}

extern void wifi_init_sta(void);              // standard boilerplate
extern esp_mqtt_client_handle_t mqtt_start_with_timeout(uint32_t timeout_ms);

void app_main(void) {
    // Cold-boot detection: any non-deep-sleep reset (power-on, brown-out,
    // watchdog) may leave RTC slow memory uninitialised. Gate every retained
    // variable through the magic-marker sentinel.
    if (s_rtc_magic != RTC_MAGIC) {
        ESP_LOGI(TAG, "Cold boot — initialising RTC state");
        s_total_wh    = 0.0;
        s_wake_count  = 0;
        s_primer_done = false;
        s_t_prev_us   = 0;
        s_rtc_magic   = RTC_MAGIC;
    }
    s_wake_count++;

    // ----- Power up the rbAmp and bring up I2C -----
    rbamp_power(true);
    i2c_master_bus_handle_t bus;
    i2c_master_dev_handle_t dev;
    ESP_ERROR_CHECK(rbamp_io_init_bus(I2C_NUM_0, 21, 22, &bus));
    ESP_ERROR_CHECK(rbamp_io_add_device(bus, RB_ADDR, &dev));
    if (rbamp_io_wait_ready(dev, 2000) != ESP_OK) {
        ESP_LOGE(TAG, "rbAmp not ready — sleeping");
        rbamp_power(false);
        esp_sleep_enable_timer_wakeup(SLEEP_US);
        esp_deep_sleep_start();
    }

    // ----- First wake after cold boot: primer latch only -----
    if (!s_primer_done) {
        rbamp_io_write_u8(dev, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
        s_t_prev_us  = esp_timer_get_time();
        s_primer_done = true;
        ESP_LOGI(TAG, "Primer done — sleeping until first real period");
        rbamp_power(false);
        esp_sleep_enable_timer_wakeup(SLEEP_US);
        esp_deep_sleep_start();
    }

    // ----- Subsequent wakes: close the period and read its average power -----
    rbamp_io_write_u8(dev, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
    int64_t t_now_us = esp_timer_get_time();
    vTaskDelay(pdMS_TO_TICKS(50));

    uint8_t valid = 0;
    rbamp_io_read_u8(dev, RB_REG_PERIOD_VALID, &valid);
    if ((valid & 0x01) == 0) {
        ESP_LOGW(TAG, "Stale snapshot — skipping this cycle");
        s_t_prev_us = t_now_us;
        rbamp_power(false);
        esp_sleep_enable_timer_wakeup(SLEEP_US);
        esp_deep_sleep_start();
    }

    float avg_p = 0.0f;
    rbamp_io_read_float_le(dev, RB_REG_PERIOD_AVG_P0, &avg_p);
    float dt_s = (float)(t_now_us - s_t_prev_us) / 1000000.0f;
    s_t_prev_us = t_now_us;
    s_total_wh += (double)avg_p * dt_s / 3600.0;

    // ----- Publish via Wi-Fi + MQTT (graceful fallback on failure) -----
    wifi_init_sta();
    esp_mqtt_client_handle_t mqtt = mqtt_start_with_timeout(8000);
    if (mqtt) {
        char payload[256];
        snprintf(payload, sizeof(payload),
            "{\"wake\":%" PRIu32 ",\"dt_s\":%.1f,"
             "\"avg_p\":%.1f,\"energy_wh\":%.3f}",
            s_wake_count, dt_s, avg_p, s_total_wh);
        esp_mqtt_client_publish(mqtt, "rbamp/remote/state",
                                payload, 0, /*qos*/ 1, /*retain*/ 1);
        ESP_LOGI(TAG, "Published: %s", payload);
        esp_mqtt_client_stop(mqtt);
        esp_mqtt_client_destroy(mqtt);
    } else {
        ESP_LOGW(TAG, "Wi-Fi/MQTT failed — value buffered in RTC, retry next wake");
    }
    esp_wifi_stop();

    rbamp_power(false);
    esp_sleep_enable_timer_wakeup(SLEEP_US);
    ESP_LOGI(TAG, "Deep sleep");
    esp_deep_sleep_start();
}
```

**Power budget (typical, ESP32-WROOM)**:

- Awake duration per wake: ~3 s (Wi-Fi + MQTT + I2C).
- Wake current: ~80 mA average → ~0.07 mAh per wake.
- Sleep current: ~10 µA (RTC + retained domains).
- One wake every 10 min → ~10 mAh/day → a 2000 mAh Li-ion cell lasts ~6 months.

**Notes**:

- `RTC_DATA_ATTR` survives **deep sleep** but is **lost on full power loss** (battery disconnect). For absolute persistence, store the running total to NVS at every wake.
- The primer-latch step is gated by `s_primer_done` — the first sample after a cold boot is correctly discarded.
- `mqtt_start_with_timeout()` is a thin helper that calls `esp_mqtt_client_start()` and waits for `MQTT_EVENT_CONNECTED` with a deadline. Implement as a tiny wrapper around an event handler; not shown here for brevity.

---

## Example 10 — Time-of-use (TOU) tariff with SNTP wall-clock

**Goal**: bill energy under a time-of-use schedule (different rates for peak and off-peak hours). The ESP32 keeps a real wall-clock via SNTP, splits each latched period into the right tariff bucket, resets daily totals at midnight, and publishes payloads with ISO timestamps.
**Hardware**: ESP32 + one rbAmp UI1 + Wi-Fi.

```c
#include <time.h>
#include <inttypes.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "esp_timer.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "esp_netif_sntp.h"
#include "esp_sntp.h"
#include "nvs_flash.h"
#include "mqtt_client.h"
#include "rbamp_io.h"

static const char *TAG = "rbamp_tou";
#define RB_ADDR  0x50

/* POSIX TZ string. Examples:
 *   Central Europe: "CET-1CEST,M3.5.0/2,M10.5.0/3"
 *   US Eastern:     "EST5EDT,M3.2.0/2,M11.1.0/2"
 *   UTC fixed:      "UTC0"
 */
#define TZ_INFO  "CET-1CEST,M3.5.0/2,M10.5.0/3"

#define PEAK_START_HOUR 8     // inclusive
#define PEAK_END_HOUR   22    // exclusive

extern void wifi_init_sta(void);
extern esp_mqtt_client_handle_t mqtt_start(void);

/* Tariff buckets. */
typedef struct {
    double peak_wh, off_peak_wh;
    int    day_of_year;       // tm_yday to detect a day roll-over
} day_totals_t;
typedef struct {
    double peak_wh, off_peak_wh;
} lifetime_totals_t;

static day_totals_t      s_today    = { .day_of_year = -1 };
static lifetime_totals_t s_lifetime = { 0 };
static esp_mqtt_client_handle_t s_mqtt;

/**
 * @brief  Synchronise the system clock with SNTP and apply the local timezone.
 * @details Blocks until the first valid time is received (timeout 15 s).
 */
static esp_err_t sntp_sync(void) {
    esp_sntp_config_t cfg = ESP_NETIF_SNTP_DEFAULT_CONFIG_MULTIPLE(2,
        ESP_SNTP_SERVER_LIST("pool.ntp.org", "time.google.com"));
    cfg.start = true;
    ESP_ERROR_CHECK(esp_netif_sntp_init(&cfg));

    if (esp_netif_sntp_sync_wait(pdMS_TO_TICKS(15000)) != ESP_OK) {
        ESP_LOGW(TAG, "SNTP sync timed out");
        return ESP_ERR_TIMEOUT;
    }
    setenv("TZ", TZ_INFO, 1);
    tzset();

    time_t now; time(&now);
    struct tm lt; localtime_r(&now, &lt);
    ESP_LOGI(TAG, "SNTP sync: %04d-%02d-%02d %02d:%02d:%02d %s",
             lt.tm_year + 1900, lt.tm_mon + 1, lt.tm_mday,
             lt.tm_hour, lt.tm_min, lt.tm_sec, lt.tm_zone);
    return ESP_OK;
}

/**
 * @brief  Decide whether a given hour falls inside the peak tariff window.
 * @details For schedules that cross midnight, change to
 *          `(hour >= PEAK_START_HOUR || hour < PEAK_END_HOUR)`.
 */
static inline bool is_peak_hour(int hour) {
    return (hour >= PEAK_START_HOUR && hour < PEAK_END_HOUR);
}

/**
 * @brief  Add one period's energy to the right bucket and handle midnight roll-over.
 */
static void accumulate(double e_wh, const struct tm *tm_now) {
    if (s_today.day_of_year != tm_now->tm_yday) {
        if (s_today.day_of_year >= 0) {
            char payload[192];
            snprintf(payload, sizeof(payload),
                "{\"yday\":%d,\"peak_wh\":%.3f,\"off_peak_wh\":%.3f,"
                 "\"total_wh\":%.3f}",
                s_today.day_of_year, s_today.peak_wh, s_today.off_peak_wh,
                s_today.peak_wh + s_today.off_peak_wh);
            esp_mqtt_client_publish(s_mqtt, "rbamp/tou/day_close",
                                    payload, 0, /*qos*/ 1, /*retain*/ 1);
            ESP_LOGI(TAG, "Day closed: %s", payload);
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

void app_main(void) {
    wifi_init_sta();
    sntp_sync();
    s_mqtt = mqtt_start();

    i2c_master_bus_handle_t bus;
    i2c_master_dev_handle_t dev;
    ESP_ERROR_CHECK(rbamp_io_init_bus(I2C_NUM_0, 21, 22, &bus));
    ESP_ERROR_CHECK(rbamp_io_add_device(bus, RB_ADDR, &dev));
    ESP_ERROR_CHECK(rbamp_io_wait_ready(dev, 2000));

    rbamp_io_write_u8(dev, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);   // primer
    int64_t t_prev_us = esp_timer_get_time();
    ESP_LOGI(TAG, "TOU started: peak %02d:00..%02d:00 local",
             PEAK_START_HOUR, PEAK_END_HOUR);

    while (1) {
        vTaskDelay(pdMS_TO_TICKS(60 * 1000));

        rbamp_io_write_u8(dev, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
        int64_t t_now_us = esp_timer_get_time();
        vTaskDelay(pdMS_TO_TICKS(50));

        uint8_t valid = 0;
        rbamp_io_read_u8(dev, RB_REG_PERIOD_VALID, &valid);
        if ((valid & 0x01) == 0) {
            t_prev_us = esp_timer_get_time();
            continue;
        }
        float avg_p = 0.0f;
        rbamp_io_read_float_le(dev, RB_REG_PERIOD_AVG_P0, &avg_p);
        float dt_s = (float)(t_now_us - t_prev_us) / 1000000.0f;
        double e_wh = (double)avg_p * dt_s / 3600.0;
        t_prev_us = t_now_us;

        // Tag with the current local time.
        time_t now_s; time(&now_s);
        struct tm tm_now; localtime_r(&now_s, &tm_now);
        if (tm_now.tm_year + 1900 < 2023) {
            ESP_LOGW(TAG, "Clock not yet synced — retrying SNTP");
            sntp_sync();
            // Reset t_prev_us so the NEXT valid period does not get an
            // inflated dt_s spanning the SNTP retry interval.
            t_prev_us = esp_timer_get_time();
            continue;
        }

        accumulate(e_wh, &tm_now);
        bool peak = is_peak_hour(tm_now.tm_hour);

        char iso[40];
        strftime(iso, sizeof(iso), "%Y-%m-%dT%H:%M:%S%z", &tm_now);

        char payload[384];
        snprintf(payload, sizeof(payload),
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
        esp_mqtt_client_publish(s_mqtt, "rbamp/tou/state",
                                payload, 0, /*qos*/ 0, /*retain*/ 0);
        ESP_LOGI(TAG, "%s", payload);
    }
}
```

**Notes**:

- The IDF SNTP API in 5.x is `esp_netif_sntp_init` / `esp_netif_sntp_sync_wait` (replaces the older `sntp_setservername` / `sntp_init` from LwIP). It returns once the time is plausible.
- `setenv("TZ", ...)` followed by `tzset()` enables `localtime_r()` to apply DST rules automatically — no manual offset handling required.
- For STANDARD / PRO bidirectional metering, call `accumulate()` twice — once for consumption and once for export — and publish both peak/off-peak consumption and peak/off-peak export.
- The daily roll-over publishes a retained payload on `rbamp/tou/day_close`; HA or Grafana can subscribe to record daily totals.

---

## Example comparison

| # | Complexity | Output | MQTT | DRDY | Multi-module | Bidirectional | Persistence | Use case |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|---|
| 1 | minimal | UART log | — | — | — | — | — | smoke test |
| 2 | low | OLED | — | — | — | — | RAM | boxed meter |
| 3 | low | UART log | — | — | yes (3) | — | — | home monitoring |
| 4 | medium | — | yes | — | — | — | RAM | per-appliance in HA |
| 5 | medium | UART log | — | yes | — | yes (master) | RAM | solar home on BASIC tier |
| 6 | high | — | yes | — | yes (3) | yes | RAM | full home balance |
| 7 | medium | UART + SD | — | yes | — | — | SD card | event detection |
| 8 | medium | — | yes (+disco) | — | — | optional | RAM + MQTT | HA Auto-discovery |
| 9 | medium | — | yes | — | — | — | RTC memory | off-grid / outdoor logger |
| 10 | medium | UART log | yes | — | — | optional | RAM + MQTT | TOU peak/off-peak with SNTP |

## Best practices

- **Always check `ESP_ERROR_CHECK()` return paths.** I2C transactions can transiently fail on long buses; production code should retry rather than abort.
- **Keep ISRs tiny.** GPIO handlers should push a sentinel into a queue and let a FreeRTOS task do the I2C work.
- **Share one I2C driver per port.** Multiple slaves (rbAmp + OLED + EEPROM) attach to the same `i2c_master_bus_handle_t` — never install two drivers on the same port.
- **Use `esp_timer_get_time()` for the master clock.** Microsecond precision, monotonic, survives `vTaskDelay` jitter.
- **`PERIOD_VALID` (`0x07`) must be checked after every `CMD_LATCH_PERIOD`.** Stale snapshots happen if the master polls faster than the firmware integrates.
- **Wait ≥ 50 ms after a latch** before reading `0xDC..0xEF` — the firmware finalises the snapshot inside its main loop.
- **Use `i2c_master_probe()` for bus scans** — it is the IDF 5.x equivalent of the Arduino `endTransmission()` ping.
- **For battery applications, gate the rbAmp power** through a MOSFET and call `esp_sleep_enable_timer_wakeup()` + `esp_deep_sleep_start()` between samples. Use `RTC_DATA_ATTR` for state that must survive sleep.
- **Persist Wh totals to NVS** if you need them to survive a full power loss (battery removal). RTC slow memory only survives deep sleep.

## What next

After working through these ten examples:

- For the high-level `rbamp` ESP-IDF component, see [19_esp_idf_library.md](esp-idf-overview.md).
- For Arduino-style ESP32 development, see [10_arduino_examples.md](arduino-examples.md) (raw) or [17_arduino_library.md](library-arduino-overview.md) (library).
- For MicroPython / CircuitPython, see [12_micropython_examples.md](micropython-examples.md) (raw) or [18_micropython_library.md](micropython-examples.md) (library).
- For ESPHome integration (declarative YAML), see [`tools/esphome-rbamp/docs/en/`](esphome-overview.md).
- The formal I2C register specification used here lives in [11_api_reference.md](api-reference.md).
- For Linux SBC (CPython) integration, see [16_python_sbc_examples.md](python-sbc-examples.md) (raw) or [20_python_sbc_library.md](python-overview.md) (library).
