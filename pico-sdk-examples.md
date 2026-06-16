# 15 · Raspberry Pi Pico SDK Examples

This chapter is the Raspberry Pi Pico equivalent of [10_arduino_examples.md](arduino-examples.md), [12_micropython_examples.md](micropython-examples.md), [13_esp_idf_examples.md](esp-idf-examples.md) and [14_stm32_hal_examples.md](stm32-hal-examples.md): the same ten scenarios, ported to **Pico SDK** (`pico-sdk` 1.5+ / 2.x) native C with LwIP-based networking.

> **These examples talk to rbAmp directly through the raw I2C register API.** They are intended for production embedded firmware on RP2040 / RP2350 — direct control of the dual-core Cortex-M0+ / M33, integration with the LwIP MQTT client, FATFS, and the SDK's sleep / persistence primitives.
>
> **No dedicated Pico SDK library is shipped in v1** — the five v1 client libraries cover Arduino (chapter 17, supports STM32duino + arduino-pico for RP2040 via the Arduino Wire API), MicroPython (chapter 18, runs on the RP2040 MicroPython port), ESP-IDF (chapter 19), CPython on Linux SBC (chapter 20), and STM32 HAL (chapter 21, deferred). For native Pico SDK firmware in C, follow the raw-API recipes below. If you want a higher-level API on RP2040, the easiest path is the arduino-pico core + the rbAmp Arduino library — see [chapter 17 · Arduino Library](https://rbamp.com/docs/modules-basic-standard-library-arduino-overview). A native Pico SDK port may land in v2 if there is pilot demand.

## Supported targets and toolchain

| Target | Validated | Wi-Fi | Notes |
|---|:---:|:---:|---|
| Raspberry Pi Pico (RP2040) | ✓ | no | I2C-only examples 1–3, 5, 7 |
| Raspberry Pi Pico W (RP2040 + CYW43) | ✓ | yes | All examples; uses `pico-cyw43-arch` + LwIP |
| Raspberry Pi Pico 2 (RP2350) | ✓ | no | Cortex-M33 |
| Raspberry Pi Pico 2 W (RP2350 + CYW43) | ✓ | yes | All examples |

Toolchain: **`pico-sdk` 1.5.1+** (or **2.x** for RP2350), CMake 3.13+, `arm-none-eabi-gcc` 12+, the official VS Code "Raspberry Pi Pico" extension is recommended.

## Project layout

Every example uses the standard `pico-sdk` project structure:

```
example_NN/
├── CMakeLists.txt
├── pico_sdk_import.cmake
└── src/
    ├── example_NN.c
    ├── rbamp_io.h
    ├── rbamp_io.c
    └── lwipopts.h         (only for Wi-Fi examples)
```

Top-level `CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.13)

# Pull in the SDK before project() — see pico-sdk template.
include(pico_sdk_import.cmake)

# For Pico W, set BOARD before project():
# set(PICO_BOARD pico_w)

project(example_NN C CXX ASM)
set(CMAKE_C_STANDARD 11)
pico_sdk_init()

add_executable(example_NN
    src/example_NN.c
    src/rbamp_io.c)

target_include_directories(example_NN PRIVATE src)

# Always-required libraries.
target_link_libraries(example_NN
    pico_stdlib
    hardware_i2c)

# Wi-Fi examples add CYW43 + LwIP + mqtt:
# target_link_libraries(example_NN
#     pico_cyw43_arch_lwip_threadsafe_background
#     pico_lwip_mqtt
#     pico_lwip_sntp)
# target_compile_definitions(example_NN PRIVATE WIFI_SSID=\"...\" WIFI_PASSWORD=\"...\")

# Route stdio to USB CDC (for Pico's serial console).
pico_enable_stdio_usb(example_NN 1)
pico_enable_stdio_uart(example_NN 0)

pico_add_extra_outputs(example_NN)
```

Build and flash:

```
mkdir build && cd build
cmake ..
make -j
# Drag-and-drop the .uf2 onto the RPI-RP2 / RP2350 USB mass-storage device.
```

## Examples table of contents

| # | Title | Difficulty | Output | MQTT | DRDY | Multi-module | Bidirectional | Use case |
|:---:|---|:---:|:---:|:---:|:---:|:---:|:---:|---|
| 1 | [Quick read](#example-1--quick-read-minimal) | minimal | USB stdio | — | — | — | — | Smoke test |
| 2 | [60-second energy meter on OLED](#example-2--60-second-energy-meter-on-oled) | low | SSD1306 | — | — | — | — | Boxed Wh counter |
| 3 | [Multi-module monitor](#example-3--multi-module-monitor) | low | USB stdio | — | — | yes (3) | — | Whole-home monitoring |
| 4 | [Per-appliance energy tracker (UI3)](#example-4--per-appliance-energy-tracker-ui3) | medium | — | yes | — | — | — | Sub-metering in HA |
| 5 | [Master-side bidirectional on a BASIC module](#example-5--master-side-bidirectional-on-a-basic-module) | medium | USB stdio | — | yes | — | yes (master) | Solar home on BASIC tier |
| 6 | [Home energy balance](#example-6--home-energy-balance) | high | — | yes | — | yes (3) | yes | Full home balance |
| 7 | [Power-event detection](#example-7--power-event-detection) | medium | USB + SD | — | yes | — | — | Appliance event log |
| 8 | [MQTT publisher with HA Auto-discovery](#example-8--mqtt-publisher-with-home-assistant-auto-discovery) | medium | — | yes (+disco) | — | — | optional | Drop-in HA integration |
| 9 | [Battery-powered logger with dormant sleep](#example-9--battery-powered-logger-with-dormant-sleep) | medium | — | yes | — | — | — | Off-grid / outdoor meter |
| 10 | [Time-of-use (TOU) tariff with SNTP wall-clock](#example-10--time-of-use-tou-tariff-with-sntp-wall-clock) | medium | USB stdio | yes | — | — | optional | Peak / off-peak tariff metering |

> Wi-Fi examples (4, 6, 8, 9, 10) require **Pico W** or **Pico 2 W**.

---

## Common helpers (`rbamp_io.h` / `rbamp_io.c`)

Place these in your project's `src/` directory. To save space, the `#include "rbamp_io.h"` line is omitted in each example below.

### `rbamp_io.h`

```c
/**
 * @file rbamp_io.h
 * @brief Raw I2C helpers for the rbAmp slave (Raspberry Pi Pico SDK).
 */
#pragma once

#include <stdint.h>
#include <stdbool.h>
#include "pico/stdlib.h"
#include "hardware/i2c.h"

#ifdef __cplusplus
extern "C" {
#endif

#define RBAMP_I2C_TIMEOUT_US  50000U     /* 50 ms */

/**
 * @brief  Initialise the I2C peripheral and pin functions.
 * @param  i2c_inst   i2c0 or i2c1.
 * @param  sda_gpio   SDA GPIO number (e.g. 4 for default i2c0).
 * @param  scl_gpio   SCL GPIO number (e.g. 5 for default i2c0).
 * @param  baudrate   Bus speed (use 100000 for 100 kHz).
 */
void rbamp_io_init(i2c_inst_t *i2c_inst, uint sda_gpio, uint scl_gpio,
                   uint baudrate);

/**
 * @brief  Read one byte from a register of an rbAmp slave.
 * @return PICO_OK on success, an error code otherwise.
 */
int rbamp_io_read_u8(uint8_t addr, uint8_t reg, uint8_t *out);

/**
 * @brief  Read a little-endian float32 from four consecutive registers.
 * @details Reads four consecutive bytes. rbAmp supports READ auto-increment
 *          (a single burst-read is valid for atomicity). The per-byte form
 *          is shown for clarity. See chapter 11 for asymmetry rules.
 */
int rbamp_io_read_float_le(uint8_t addr, uint8_t reg, float *out);

/**
 * @brief  Read a little-endian uint32 from four consecutive registers.
 */
int rbamp_io_read_u32_le(uint8_t addr, uint8_t reg, uint32_t *out);

/**
 * @brief  Write a single byte to a register.
 */
int rbamp_io_write_u8(uint8_t addr, uint8_t reg, uint8_t val);

/**
 * @brief  Broadcast CMD_LATCH_PERIOD to every rbAmp with GC reception
 *         enabled (FLEET_CONFIG.bit0 = 1; opt-in, default OFF).
 * @details Canonical 5-byte general-call frame: A5 27 group tick_lo tick_hi.
 *          Slaves reject any first byte != 0xA5. See chapter 11 §6.3.2.
 * @param  tick16  Caller's 16-bit window counter (mirrored at GC_TICK 0x59).
 * @param  group   GROUP_ID filter (0 = all-call).
 * @return PICO_OK on ACK; PICO_ERROR_GENERIC on NACK — fall back to per-module.
 */
int rbamp_io_broadcast_latch(uint16_t tick16, uint8_t group);

/**
 * @brief  Probe whether a slave responds at the given address.
 * @return true if the slave acks, false otherwise.
 */
bool rbamp_io_probe(uint8_t addr);

/**
 * @brief  Block until DATA_VALID bit 0 of register 0xCE reads 1, or timeout.
 * @return PICO_OK on success, PICO_ERROR_TIMEOUT otherwise.
 */
int rbamp_io_wait_ready(uint8_t addr, uint32_t timeout_ms);

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
#define RB_REG_PERIOD_MAX_P     0xE0U
#define RB_REG_PERIOD_LATCH_MS  0xECU

/* ===== Commands ===== */
#define RB_CMD_RESET            0x01U
#define RB_CMD_SAVE_GAINS       0x26U
#define RB_CMD_LATCH_PERIOD     0x27U

#ifdef __cplusplus
}
#endif
```

### `rbamp_io.c`

```c
#include "rbamp_io.h"
#include <string.h>
#include "pico/error.h"

static i2c_inst_t *s_i2c = NULL;

void rbamp_io_init(i2c_inst_t *i2c_inst, uint sda_gpio, uint scl_gpio,
                   uint baudrate) {
    s_i2c = i2c_inst;
    i2c_init(s_i2c, baudrate);
    gpio_set_function(sda_gpio, GPIO_FUNC_I2C);
    gpio_set_function(scl_gpio, GPIO_FUNC_I2C);
    /* The rbAmp board carries its own pull-ups; do NOT enable the internal
     * pulls unless yours are physically removed. */
}

/**
 * @brief  Internal helper — one register read.
 * @details Pattern: write reg-address (nostop=true) + read 1 byte.
 */
static int read_one(uint8_t addr, uint8_t reg, uint8_t *out) {
    int n = i2c_write_timeout_us(s_i2c, addr, &reg, 1,
                                  /*nostop=*/true, RBAMP_I2C_TIMEOUT_US);
    if (n != 1) return PICO_ERROR_GENERIC;
    n = i2c_read_timeout_us(s_i2c, addr, out, 1,
                             /*nostop=*/false, RBAMP_I2C_TIMEOUT_US);
    return (n == 1) ? PICO_OK : PICO_ERROR_GENERIC;
}

int rbamp_io_read_u8(uint8_t addr, uint8_t reg, uint8_t *out) {
    return read_one(addr, reg, out);
}

int rbamp_io_read_float_le(uint8_t addr, uint8_t reg, float *out) {
    uint8_t buf[4];
    for (int i = 0; i < 4; i++) {
        int rc = read_one(addr, (uint8_t)(reg + i), &buf[i]);
        if (rc != PICO_OK) return rc;
    }
    memcpy(out, buf, sizeof(float));   /* RP2040 / RP2350 are little-endian */
    return PICO_OK;
}

int rbamp_io_read_u32_le(uint8_t addr, uint8_t reg, uint32_t *out) {
    uint8_t buf[4];
    for (int i = 0; i < 4; i++) {
        int rc = read_one(addr, (uint8_t)(reg + i), &buf[i]);
        if (rc != PICO_OK) return rc;
    }
    *out = (uint32_t)buf[0] | (uint32_t)buf[1] << 8
         | (uint32_t)buf[2] << 16 | (uint32_t)buf[3] << 24;
    return PICO_OK;
}

int rbamp_io_write_u8(uint8_t addr, uint8_t reg, uint8_t val) {
    uint8_t tx[2] = { reg, val };
    int n = i2c_write_timeout_us(s_i2c, addr, tx, 2,
                                  /*nostop=*/false, RBAMP_I2C_TIMEOUT_US);
    return (n == 2) ? PICO_OK : PICO_ERROR_GENERIC;
}

int rbamp_io_broadcast_latch(uint16_t tick16, uint8_t group) {
    /* Canonical 5-byte general-call frame: A5 27 group tick_lo tick_hi. */
    uint8_t tx[5] = {
        0xA5,                          /* frame magic */
        RB_CMD_LATCH_PERIOD,           /* opcode 0x27 */
        group,                         /* group_id (0 = all-call) */
        (uint8_t)(tick16 & 0xFF),
        (uint8_t)((tick16 >> 8) & 0xFF),
    };
    int n = i2c_write_timeout_us(s_i2c, /*addr=*/0x00U, tx, sizeof tx,
                                  /*nostop=*/false, RBAMP_I2C_TIMEOUT_US);
    return (n == (int)sizeof tx) ? PICO_OK : PICO_ERROR_GENERIC;
}

bool rbamp_io_probe(uint8_t addr) {
    uint8_t dummy;
    /* A 1-byte read is the simplest ACK probe — read the STATUS register. */
    return read_one(addr, RB_REG_STATUS, &dummy) == PICO_OK;
}

int rbamp_io_wait_ready(uint8_t addr, uint32_t timeout_ms) {
    absolute_time_t deadline = make_timeout_time_ms(timeout_ms);
    while (!time_reached(deadline)) {
        uint8_t v = 0;
        if (rbamp_io_read_u8(addr, RB_REG_DATA_VALID, &v) == PICO_OK
            && (v & 0x01U)) {
            return PICO_OK;
        }
        sleep_ms(50);
    }
    return PICO_ERROR_TIMEOUT;
}
```

---

## Example 1 — Quick read (minimal)

**Goal**: print U, I, P, PF over USB CDC once per second.
**Hardware**: Raspberry Pi Pico + one rbAmp UI1. I2C on GPIO 4 (SDA) / GPIO 5 (SCL) — default `i2c0` pins.

`src/example_01.c`:

```c
#include <stdio.h>
#include "pico/stdlib.h"
#include "hardware/i2c.h"
#include "rbamp_io.h"

#define RB_ADDR  0x50U

int main(void) {
    stdio_init_all();                       /* USB CDC stdio */
    sleep_ms(2000);                         /* let the USB host enumerate */

    /* I2C0 on GPIO 4 / 5 at 100 kHz. */
    rbamp_io_init(i2c0, 4, 5, 100000);

    /* Wait until the first RT window has been computed. */
    if (rbamp_io_wait_ready(RB_ADDR, 2000) != PICO_OK) {
        printf("ERROR: rbAmp not ready\n");
        while (1) tight_loop_contents();
    }

    uint8_t ver = 0;
    rbamp_io_read_u8(RB_ADDR, RB_REG_VERSION, &ver);
    printf("rbAmp version: 0x%02X\n", ver);

    while (true) {
        float u, i_, p, pf;
        rbamp_io_read_float_le(RB_ADDR, RB_REG_U_RMS,   &u );
        rbamp_io_read_float_le(RB_ADDR, RB_REG_I0_RMS,  &i_);
        rbamp_io_read_float_le(RB_ADDR, RB_REG_P0_REAL, &p );
        rbamp_io_read_float_le(RB_ADDR, RB_REG_PF0,     &pf);
        printf("U=%.1fV  I=%.3fA  P=%+.1fW  PF=%+.3f\n", u, i_, p, pf);
        sleep_ms(1000);
    }
}
```

Expected serial console output:

```
rbAmp version: 0x04
U=230.1V  I=0.262A  P=+60.2W  PF=+0.998
U=229.9V  I=0.262A  P=+60.1W  PF=+0.998
...
```

> Connect to the serial console with `minicom -D /dev/ttyACM0 -b 115200` (Linux), `screen /dev/cu.usbmodem*` (macOS), or `PuTTY` (Windows).

---

## Example 2 — 60-second energy meter on OLED

**Goal**: Wh counter refreshed once per minute, shown on a 128×64 SSD1306 sharing the rbAmp I2C bus.
**Hardware**: Pico + rbAmp + SSD1306 (`0x3C`) on the same bus.
**Library**: a stock Pico-SDK SSD1306 driver (e.g. `daschr/pico-ssd1306`). The OLED API below is symbolic — substitute your driver's calls.

```c
#include <stdio.h>
#include <string.h>
#include "pico/stdlib.h"
#include "rbamp_io.h"
#include "ssd1306.h"

#define RB_ADDR  0x50U

int main(void) {
    stdio_init_all();
    sleep_ms(2000);
    rbamp_io_init(i2c0, 4, 5, 100000);

    /* OLED shares the same i2c0. */
    ssd1306_t oled;
    ssd1306_init(&oled, 128, 64, 0x3C, i2c0);

    rbamp_io_wait_ready(RB_ADDR, 2000);

    /* Primer latch — discard its snapshot. */
    rbamp_io_write_u8(RB_ADDR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
    uint32_t t_prev_ms = to_ms_since_boot(get_absolute_time());
    double total_wh = 0.0;

    while (true) {
        sleep_ms(60 * 1000);

        /* 1) Final latch. */
        rbamp_io_write_u8(RB_ADDR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
        uint32_t t_now_ms = to_ms_since_boot(get_absolute_time());
        sleep_ms(50);                       /* let firmware finalise snapshot */

        /* 2) Check PERIOD_VALID. */
        uint8_t valid = 0;
        rbamp_io_read_u8(RB_ADDR, RB_REG_PERIOD_VALID, &valid);
        if ((valid & 0x01U) == 0) {
            printf("WARN: stale snapshot\n");
            t_prev_ms = to_ms_since_boot(get_absolute_time());
            continue;
        }

        /* 3) Read time-averaged P. */
        float avg_p = 0.0f;
        rbamp_io_read_float_le(RB_ADDR, RB_REG_PERIOD_AVG_P0, &avg_p);

        /* 4) Energy uses the MASTER clock. */
        float dt_s = (float)(t_now_ms - t_prev_ms) / 1000.0f;
        double e_wh = (double)avg_p * dt_s / 3600.0;
        total_wh += e_wh;
        t_prev_ms = t_now_ms;

        /* 5) Render. */
        char line[32];
        ssd1306_clear(&oled);
        snprintf(line, sizeof(line), "P:  %6.1f W", avg_p);
        ssd1306_draw_string(&oled, 0, 0,  1, line);
        snprintf(line, sizeof(line), "dt: %6.1f s", dt_s);
        ssd1306_draw_string(&oled, 0, 16, 1, line);
        snprintf(line, sizeof(line), "%.2f Wh", total_wh);
        ssd1306_draw_string(&oled, 0, 32, 2, line);
        ssd1306_show(&oled);

        printf("avg_P=%.1f  E_period=%.3f  total=%.3f\n", avg_p, e_wh, total_wh);
    }
}
```

> The OLED driver and rbAmp share `i2c0` — never call `i2c_init()` twice; the helper's `rbamp_io_init()` already did it.

---

## Example 3 — Multi-module monitor

**Goal**: poll three modules (main feed, water heater, AC) and print the total.
**Hardware**: Pico + 3 × rbAmp UI1 at `0x50`, `0x51`, `0x52`.

```c
#include <stdio.h>
#include <stddef.h>
#include "pico/stdlib.h"
#include "rbamp_io.h"

typedef struct {
    uint8_t addr;
    const char *label;
} module_t;

static const module_t modules[] = {
    { 0x50, "Mains " },
    { 0x51, "Boiler" },
    { 0x52, "AC    " },
};
#define N_MODULES (sizeof(modules) / sizeof(modules[0]))

int main(void) {
    stdio_init_all();
    sleep_ms(2000);
    rbamp_io_init(i2c0, 4, 5, 100000);

    /* Probe every module before entering the read loop. */
    for (size_t i = 0; i < N_MODULES; i++) {
        if (!rbamp_io_probe(modules[i].addr)) {
            printf("ERROR: %s at 0x%02X not found\n",
                   modules[i].label, modules[i].addr);
            while (1) tight_loop_contents();
        }
    }
    printf("All modules OK\n");

    while (true) {
        float total_p = 0.0f;
        printf("---\n");

        /* Round-robin read. */
        for (size_t i = 0; i < N_MODULES; i++) {
            float u = 0.0f, p = 0.0f;
            rbamp_io_read_float_le(modules[i].addr, RB_REG_U_RMS,   &u);
            rbamp_io_read_float_le(modules[i].addr, RB_REG_P0_REAL, &p);
            printf("[%s] U=%.1f  P=%+.1f W\n", modules[i].label, u, p);
            total_p += p;
        }
        printf("TOTAL: %.1f W\n", total_p);
        sleep_ms(2000);
    }
}
```

---

## Example 4 — Per-appliance energy tracker (UI3)

**Goal**: a UI3 module with three CT clamps on three different loads on the same phase; independent Wh counters published to MQTT every minute.
**Hardware**: **Pico W** (or Pico 2 W) + 1 × rbAmp UI3 + Wi-Fi network with an MQTT broker.
**SDK pieces**: `pico_cyw43_arch_lwip_threadsafe_background`, `pico_lwip_mqtt`.

`CMakeLists.txt` additions:

```cmake
set(PICO_BOARD pico_w)

target_link_libraries(example_04
    pico_stdlib
    hardware_i2c
    pico_cyw43_arch_lwip_threadsafe_background
    pico_lwip_mqtt)

target_compile_definitions(example_04 PRIVATE
    WIFI_SSID=\"ssid\"
    WIFI_PASSWORD=\"password\"
    MQTT_BROKER_IP=\"192.168.1.10\")
```

`src/lwipopts.h` (minimal — extend from `pico-examples/pico_w/wifi/lwipopts_examples_common.h`):

```c
#pragma once
#include "lwip/apps/mqtt.h"
#define LWIP_MQTT 1
#define MEMP_NUM_SYS_TIMEOUT 8
/* Use the SDK's standard threadsafe-background opts. */
```

`src/example_04.c`:

```c
#include <stdio.h>
#include <string.h>
#include "pico/stdlib.h"
#include "pico/cyw43_arch.h"
#include "lwip/apps/mqtt.h"
#include "lwip/dns.h"
#include "rbamp_io.h"

#define RB_ADDR 0x50U
static const char *CH_NAMES[3] = { "main", "heatpump", "lights" };

static mqtt_client_t *s_mqtt;
static ip_addr_t s_broker_ip;

/**
 * @brief  MQTT connection-status callback (logs result; could trigger retry).
 */
static void mqtt_conn_cb(mqtt_client_t *client, void *arg,
                         mqtt_connection_status_t status) {
    printf("MQTT connect status=%d\n", (int)status);
}

/**
 * @brief  Bring up Wi-Fi (CYW43) and connect to the MQTT broker.
 */
static void net_init(void) {
    if (cyw43_arch_init()) {
        printf("CYW43 init failed\n");
        return;
    }
    cyw43_arch_enable_sta_mode();
    if (cyw43_arch_wifi_connect_timeout_ms(WIFI_SSID, WIFI_PASSWORD,
                                            CYW43_AUTH_WPA2_AES_PSK, 30000)) {
        printf("Wi-Fi connect failed\n");
        return;
    }
    printf("Wi-Fi up\n");

    ipaddr_aton(MQTT_BROKER_IP, &s_broker_ip);
    s_mqtt = mqtt_client_new();
    struct mqtt_connect_client_info_t ci = { .client_id = "rbamp-ui3" };
    mqtt_client_connect(s_mqtt, &s_broker_ip, 1883, mqtt_conn_cb, NULL, &ci);
}

/**
 * @brief  Publish one channel's state to MQTT.
 */
static void mqtt_publish_channel(const char *ch_name, float avg_p, double e_wh) {
    char topic[64], payload[64];
    snprintf(topic,   sizeof(topic),   "rbamp/%s/state", ch_name);
    int n = snprintf(payload, sizeof(payload),
                     "{\"power\":%.1f,\"energy\":%.4f}", avg_p, e_wh);
    mqtt_publish(s_mqtt, topic, payload, n, /*qos*/ 0, /*retain*/ 0,
                 NULL, NULL);
}

int main(void) {
    stdio_init_all();
    sleep_ms(2000);
    net_init();
    rbamp_io_init(i2c0, 4, 5, 100000);
    rbamp_io_wait_ready(RB_ADDR, 2000);

    rbamp_io_write_u8(RB_ADDR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);  /* primer */
    uint32_t t_prev_ms = to_ms_since_boot(get_absolute_time());
    double total_wh[3] = { 0.0, 0.0, 0.0 };

    while (true) {
        sleep_ms(60 * 1000);

        rbamp_io_write_u8(RB_ADDR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
        uint32_t t_now_ms = to_ms_since_boot(get_absolute_time());
        sleep_ms(50);

        uint8_t valid = 0;
        rbamp_io_read_u8(RB_ADDR, RB_REG_PERIOD_VALID, &valid);
        if ((valid & 0x01U) == 0) {
            t_prev_ms = to_ms_since_boot(get_absolute_time());
            continue;
        }

        /* PERIOD_AVG_P_W per channel: I0 at 0xDC, I1 at 0xC2, I2 at 0xC6. */
        float avg_p[3];
        rbamp_io_read_float_le(RB_ADDR, RB_REG_PERIOD_AVG_P0, &avg_p[0]);
        rbamp_io_read_float_le(RB_ADDR, RB_REG_PERIOD_AVG_P1, &avg_p[1]);
        rbamp_io_read_float_le(RB_ADDR, RB_REG_PERIOD_AVG_P2, &avg_p[2]);

        float dt_s = (float)(t_now_ms - t_prev_ms) / 1000.0f;
        t_prev_ms = t_now_ms;

        for (int ch = 0; ch < 3; ch++) {
            double e = (double)avg_p[ch] * dt_s / 3600.0;
            total_wh[ch] += e;
            printf("[%s] avg_P=%+.1f W  E_total=%.3f Wh\n",
                   CH_NAMES[ch], avg_p[ch], total_wh[ch]);
            mqtt_publish_channel(CH_NAMES[ch], avg_p[ch], total_wh[ch]);
        }
    }
}
```

---

## Example 5 — Master-side bidirectional on a BASIC module

**Goal**: implement bidirectional accounting on the master while the module is BASIC firmware. The Pico reacts to the DRDY falling edge via a GPIO IRQ and splits consumption / export.
**Hardware**: Pico + rbAmp UI1; DRDY wired to GPIO 15.

```c
#include <stdio.h>
#include <math.h>
#include "pico/stdlib.h"
#include "pico/critical_section.h"
#include "hardware/gpio.h"
#include "rbamp_io.h"

#define RB_ADDR    0x50U
#define PIN_DRDY   15

static volatile bool g_data_ready = false;

/**
 * @brief  Shared GPIO IRQ callback for the Pico SDK.
 * @details The SDK uses a single callback for all GPIO IRQs — we filter by pin.
 *          Keep the callback short (no I2C, no printf inside an IRQ).
 */
static void gpio_callback(uint gpio, uint32_t events) {
    if (gpio == PIN_DRDY && (events & GPIO_IRQ_EDGE_FALL)) {
        g_data_ready = true;
    }
}

int main(void) {
    stdio_init_all();
    sleep_ms(2000);
    rbamp_io_init(i2c0, 4, 5, 100000);
    rbamp_io_wait_ready(RB_ADDR, 2000);

    /* DRDY input with internal pull-up; falling-edge IRQ. */
    gpio_init(PIN_DRDY);
    gpio_set_dir(PIN_DRDY, GPIO_IN);
    gpio_pull_up(PIN_DRDY);
    gpio_set_irq_enabled_with_callback(PIN_DRDY, GPIO_IRQ_EDGE_FALL,
                                        true, &gpio_callback);

    double e_consumed_wh = 0.0;
    double e_exported_wh = 0.0;
    // Use uint64_t for microsecond counters — to_us_since_boot() returns u64;
    // truncating to u32 overflows every ~71.6 minutes and produces a multi-kWh
    // dt_s spike on the wrap.
    uint64_t t_last_us = to_us_since_boot(get_absolute_time());
    uint64_t t_last_print_us = t_last_us;
    printf("Bidirectional accumulator started\n");

    while (true) {
        if (!g_data_ready) {
            tight_loop_contents();
            continue;
        }
        g_data_ready = false;

        uint32_t t_now_us = to_us_since_boot(get_absolute_time());

        /* Signed instantaneous P_real: + consume, − export. */
        float p = 0.0f;
        rbamp_io_read_float_le(RB_ADDR, RB_REG_P0_REAL, &p);
        float dt_s = (float)(t_now_us - t_last_us) / 1000000.0f;
        t_last_us = t_now_us;

        if (p > 0.0f) e_consumed_wh += (double)p * dt_s / 3600.0;
        else          e_exported_wh += (double)(-p) * dt_s / 3600.0;

        /* Summary print every 5 s — not on every RT window. */
        if (t_now_us - t_last_print_us >= 5000000UL) {
            t_last_print_us = t_now_us;
            printf("P=%+8.1fW   consumed=%.4f Wh  exported=%.4f Wh  net=%+.4f Wh\n",
                   p, e_consumed_wh, e_exported_wh,
                   e_consumed_wh - e_exported_wh);
        }
    }
}
```

**Notes**:

- The Pico SDK's `gpio_set_irq_enabled_with_callback()` registers a **shared** callback for all GPIO IRQs; later `gpio_set_irq_enabled()` calls reuse it.
- `volatile bool` is enough for single-flag synchronisation — RP2040 is single-issue and the boolean is read atomically.

---

## Example 6 — Home energy balance

**Goal**: full balance dashboard. Three modules:
- rbAmp #1 on the main feed (STANDARD / PRO — bidirectional)
- rbAmp #2 on the solar inverter output (BASIC OK)
- rbAmp #3 UI3 on three large loads (HP, AC, EV)

Master uses `rbamp_io_broadcast_latch()` for synchronous snapshots and publishes a retained balance to MQTT every minute.

```c
#include <stdio.h>
#include <string.h>
#include "pico/stdlib.h"
#include "pico/cyw43_arch.h"
#include "lwip/apps/mqtt.h"
#include "rbamp_io.h"

#define ADDR_MAINS  0x50U
#define ADDR_SOLAR  0x51U
#define ADDR_LOADS  0x52U
#define REG_PERIOD_AVG_P_NEG_MAINS  0xFFU   /* set per SKU datasheet on STANDARD/PRO */

extern void net_init(void);                  /* same boilerplate as Ex. 4 */
extern mqtt_client_t *s_mqtt;

typedef struct {
    double mains_in_wh;
    double mains_out_wh;
    double solar_total_wh;
    double loads_wh[3];
} totals_t;

int main(void) {
    stdio_init_all();
    sleep_ms(2000);
    net_init();
    rbamp_io_init(i2c0, 4, 5, 100000);
    rbamp_io_wait_ready(ADDR_MAINS, 2000);
    rbamp_io_wait_ready(ADDR_SOLAR, 2000);
    rbamp_io_wait_ready(ADDR_LOADS, 2000);

    /* Primer latch on all modules (general call). GC is opt-in
     * (FLEET_CONFIG.bit0, default OFF) — fall back to per-module on NACK. */
    uint16_t gc_tick = 0;
    if (rbamp_io_broadcast_latch(gc_tick++, 0) != PICO_OK) {
        rbamp_io_write_u8(ADDR_MAINS, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
        rbamp_io_write_u8(ADDR_SOLAR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
        rbamp_io_write_u8(ADDR_LOADS, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
    }
    uint32_t t_prev_ms = to_ms_since_boot(get_absolute_time());

    totals_t totals = {0};
    printf("Home balance started\n");

    while (true) {
        sleep_ms(60 * 1000);

        /* Synchronous latch on all modules. */
        if (rbamp_io_broadcast_latch(gc_tick++, 0) != PICO_OK) {
            rbamp_io_write_u8(ADDR_MAINS, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
            rbamp_io_write_u8(ADDR_SOLAR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
            rbamp_io_write_u8(ADDR_LOADS, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
        }
        uint32_t t_now_ms = to_ms_since_boot(get_absolute_time());
        sleep_ms(50);

        float dt_s = (float)(t_now_ms - t_prev_ms) / 1000.0f;
        t_prev_ms = t_now_ms;

        /* ----- MAINS (bidirectional, STANDARD/PRO) ----- */
        uint8_t valid;
        if (rbamp_io_read_u8(ADDR_MAINS, RB_REG_PERIOD_VALID, &valid) == PICO_OK
            && (valid & 0x01U)) {
            float p_consume = 0.0f, p_export = 0.0f;
            rbamp_io_read_float_le(ADDR_MAINS, RB_REG_PERIOD_AVG_P0, &p_consume);
            if (REG_PERIOD_AVG_P_NEG_MAINS != 0xFFU) {
                rbamp_io_read_float_le(ADDR_MAINS, REG_PERIOD_AVG_P_NEG_MAINS,
                                       &p_export);
            }
            totals.mains_in_wh  += (double)p_consume * dt_s / 3600.0;
            totals.mains_out_wh += (double)p_export  * dt_s / 3600.0;
        }

        /* ----- SOLAR (generation only) ----- */
        if (rbamp_io_read_u8(ADDR_SOLAR, RB_REG_PERIOD_VALID, &valid) == PICO_OK
            && (valid & 0x01U)) {
            float p = 0.0f;
            rbamp_io_read_float_le(ADDR_SOLAR, RB_REG_PERIOD_AVG_P0, &p);
            totals.solar_total_wh += (double)p * dt_s / 3600.0;
        }

        /* ----- LOADS (UI3, three channels) ----- */
        if (rbamp_io_read_u8(ADDR_LOADS, RB_REG_PERIOD_VALID, &valid) == PICO_OK
            && (valid & 0x01U)) {
            float p[3] = {0};
            rbamp_io_read_float_le(ADDR_LOADS, RB_REG_PERIOD_AVG_P0, &p[0]); /* I0 */
            rbamp_io_read_float_le(ADDR_LOADS, RB_REG_PERIOD_AVG_P1, &p[1]); /* I1 */
            rbamp_io_read_float_le(ADDR_LOADS, RB_REG_PERIOD_AVG_P2, &p[2]); /* I2 */
            for (int i = 0; i < 3; i++) {
                totals.loads_wh[i] += (double)p[i] * dt_s / 3600.0;
            }
        }

        double total_consumed = totals.mains_in_wh + totals.solar_total_wh
                              - totals.mains_out_wh;
        double solar_self_used = totals.solar_total_wh - totals.mains_out_wh;
        if (solar_self_used < 0.0) solar_self_used = 0.0;

        printf("MAINS  in=%.2f  out=%.2f\n", totals.mains_in_wh, totals.mains_out_wh);
        printf("SOLAR  gen=%.2f  self-used=%.2f  exported=%.2f\n",
               totals.solar_total_wh, solar_self_used, totals.mains_out_wh);
        printf("LOADS  HP=%.2f  AC=%.2f  EV=%.2f\n",
               totals.loads_wh[0], totals.loads_wh[1], totals.loads_wh[2]);
        printf("TOTAL  household consumed=%.2f Wh\n", total_consumed);

        char payload[512];
        int n = snprintf(payload, sizeof(payload),
            "{\"mains_in\":%.3f,\"mains_out\":%.3f,\"solar\":%.3f,"
            "\"self_used\":%.3f,\"total_consumed\":%.3f,"
            "\"hp\":%.3f,\"ac\":%.3f,\"ev\":%.3f}",
            totals.mains_in_wh, totals.mains_out_wh, totals.solar_total_wh,
            solar_self_used, total_consumed,
            totals.loads_wh[0], totals.loads_wh[1], totals.loads_wh[2]);
        mqtt_publish(s_mqtt, "home/energy/balance", payload, n,
                     /*qos*/ 1, /*retain*/ 1, NULL, NULL);
    }
}
```

---

## Example 7 — Power-event detection

**Goal**: on every DRDY edge, compare instantaneous P to an EMA and log significant deviations to an SD card mounted via SPI + FATFS.
**Hardware**: Pico + rbAmp + SPI SD-card module.
**Library**: `no-OS-FatFS-SD-SPI-RPi-Pico` or a similar SDK-friendly FATFS port.

```c
#include <stdio.h>
#include <math.h>
#include <string.h>
#include "pico/stdlib.h"
#include "hardware/gpio.h"
#include "ff.h"                          /* FATFS API */
#include "rbamp_io.h"

#define RB_ADDR    0x50U
#define PIN_DRDY   15

#define EMA_ALPHA          0.05f
#define EVENT_THRESHOLD_W  200.0f

static volatile bool g_data_ready = false;
static FATFS s_fs;
static FIL   s_logfile;

static void gpio_callback(uint gpio, uint32_t events) {
    if (gpio == PIN_DRDY && (events & GPIO_IRQ_EDGE_FALL)) {
        g_data_ready = true;
    }
}

/**
 * @brief  Mount the SD card and open events.log in append mode.
 */
static bool sd_setup(void) {
    if (f_mount(&s_fs, "", 1) != FR_OK) return false;
    if (f_open(&s_logfile, "events.log",
               FA_OPEN_APPEND | FA_WRITE) != FR_OK) return false;
    return true;
}

int main(void) {
    stdio_init_all();
    sleep_ms(2000);
    rbamp_io_init(i2c0, 4, 5, 100000);
    rbamp_io_wait_ready(RB_ADDR, 2000);

    if (sd_setup()) printf("SD mounted; logging to events.log\n");
    else            printf("WARN: SD card not available\n");

    gpio_init(PIN_DRDY);
    gpio_set_dir(PIN_DRDY, GPIO_IN);
    gpio_pull_up(PIN_DRDY);
    gpio_set_irq_enabled_with_callback(PIN_DRDY, GPIO_IRQ_EDGE_FALL,
                                        true, &gpio_callback);

    /* Seed the EMA with current power so we do not log a spurious startup event. */
    float p_ema = 0.0f;
    rbamp_io_read_float_le(RB_ADDR, RB_REG_P0_REAL, &p_ema);
    printf("Event detector started\n");

    while (true) {
        if (!g_data_ready) { tight_loop_contents(); continue; }
        g_data_ready = false;

        float p = 0.0f;
        rbamp_io_read_float_le(RB_ADDR, RB_REG_P0_REAL, &p);

        float delta = p - p_ema;
        p_ema = (1.0f - EMA_ALPHA) * p_ema + EMA_ALPHA * p;

        if (fabsf(delta) > EVENT_THRESHOLD_W) {
            const char *type = (delta > 0.0f) ? "TURN_ON" : "TURN_OFF";
            char line[128];
            int n = snprintf(line, sizeof(line),
                             "%lu  %s  delta=%+.1f W  P=%.1f W  EMA=%.1f W\n",
                             (unsigned long)to_ms_since_boot(get_absolute_time()),
                             type, delta, p, p_ema);
            printf("%s", line);
            UINT bw;
            f_write(&s_logfile, line, n, &bw);
            f_sync(&s_logfile);            /* flush to the card */
        }
    }
}
```

---

## Example 8 — MQTT publisher with Home Assistant Auto-discovery

**Goal**: a drop-in HA integration. The Pico W connects to Wi-Fi and an MQTT broker, publishes HA Auto-discovery configs (retained) and emits a JSON state every minute.
**Hardware**: Pico W (or Pico 2 W) + one rbAmp UI1 + Wi-Fi + MQTT broker.

```c
#include <stdio.h>
#include <string.h>
#include "pico/stdlib.h"
#include "pico/cyw43_arch.h"
#include "lwip/apps/mqtt.h"
#include "rbamp_io.h"

#define RB_ADDR     0x50U
#define DEVICE_ID   "rbamp_main"
#define DEVICE_NAME "Mains rbAmp"

extern void net_init(void);                /* Wi-Fi + MQTT bring-up (Ex. 4) */
extern mqtt_client_t *s_mqtt;

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

    mqtt_publish(s_mqtt, topic, payload, n,
                 /*qos*/ 1, /*retain*/ 1, NULL, NULL);
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

/* Set to true by Ex.4's mqtt_conn_cb on MQTT_CONNECT_ACCEPTED.
 * (Add `extern volatile bool s_mqtt_connected;` to the net_init translation unit.) */
extern volatile bool s_mqtt_connected;

int main(void) {
    stdio_init_all();
    sleep_ms(2000);
    net_init();

    /* MQTT connect is asynchronous — publishing discovery before the CONNECT
     * callback fires causes mqtt_publish() to return ERR_CONN and the configs
     * to silently disappear. Wait (bounded) for the connection. */
    uint32_t mqtt_deadline = to_ms_since_boot(get_absolute_time()) + 15000;
    while (!s_mqtt_connected) {
        if (to_ms_since_boot(get_absolute_time()) > mqtt_deadline) {
            printf("WARN: MQTT not connected after 15 s — discovery will be retried on reconnect\n");
            break;
        }
        sleep_ms(100);
    }
    if (s_mqtt_connected) publish_discovery_all();

    rbamp_io_init(i2c0, 4, 5, 100000);
    rbamp_io_wait_ready(RB_ADDR, 2000);

    rbamp_io_write_u8(RB_ADDR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);  /* primer */
    uint32_t t_prev_ms = to_ms_since_boot(get_absolute_time());
    double total_wh = 0.0;

    while (true) {
        sleep_ms(60 * 1000);

        rbamp_io_write_u8(RB_ADDR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
        uint32_t t_now_ms = to_ms_since_boot(get_absolute_time());
        sleep_ms(50);

        uint8_t valid = 0;
        rbamp_io_read_u8(RB_ADDR, RB_REG_PERIOD_VALID, &valid);
        if ((valid & 0x01U) == 0) {
            t_prev_ms = to_ms_since_boot(get_absolute_time());
            continue;
        }
        float avg_p = 0.0f;
        rbamp_io_read_float_le(RB_ADDR, RB_REG_PERIOD_AVG_P0, &avg_p);
        float dt_s = (float)(t_now_ms - t_prev_ms) / 1000.0f;
        total_wh += (double)avg_p * dt_s / 3600.0;
        t_prev_ms = t_now_ms;

        /* Live RT values for the state payload. */
        float u = 0, i_ = 0, pf = 0, q = 0;
        uint8_t freq = 0;
        rbamp_io_read_float_le(RB_ADDR, RB_REG_U_RMS,   &u);
        rbamp_io_read_float_le(RB_ADDR, RB_REG_I0_RMS,  &i_);
        rbamp_io_read_float_le(RB_ADDR, RB_REG_PF0,     &pf);
        rbamp_io_read_float_le(RB_ADDR, RB_REG_Q0,      &q);
        rbamp_io_read_u8     (RB_ADDR, RB_REG_AC_FREQ,  &freq);

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
        mqtt_publish(s_mqtt, "rbamp/" DEVICE_ID "/state", payload, n,
                     /*qos*/ 0, /*retain*/ 0, NULL, NULL);
        printf("Published: %s\n", payload);
    }
}
```

---

## Example 9 — Battery-powered logger with dormant sleep

**Goal**: an outdoor / off-grid logger that wakes from dormant mode every 10 minutes via the RTC alarm, latches a period, publishes via MQTT, then re-enters dormant. Pico W reaches a few hundred µA in dormant sleep — good enough for many months on a Li-ion cell.
**Hardware**: Pico W + one rbAmp UI1 + Li-ion. The rbAmp can be power-gated through a high-side switch on GPIO 16.
**Persistence**: `pico/sleep.h` + the on-chip RTC + a small NOINIT region (or external NVM) for state that must survive sleep.

```c
#include <stdio.h>
#include <string.h>
#include "pico/stdlib.h"
#include "pico/cyw43_arch.h"
#include "pico/sleep.h"
#include "hardware/rtc.h"
#include "hardware/gpio.h"
#include "lwip/apps/mqtt.h"
#include "rbamp_io.h"

#define RB_ADDR        0x50U
#define PIN_RBAMP_POWER 16
#define WAKE_SECONDS    (10 * 60)

extern void net_init(void);
extern mqtt_client_t *s_mqtt;

/* "NOINIT" region — variables that are not zeroed by the startup code.
 * They survive dormant sleep on RP2040 because RAM stays powered. */
__attribute__((section(".uninitialized_data")))
static double   s_total_wh;
__attribute__((section(".uninitialized_data")))
static uint32_t s_wake_count;
__attribute__((section(".uninitialized_data")))
static bool     s_primer_done;
__attribute__((section(".uninitialized_data")))
static uint32_t s_magic;            /* 0xCAFEFEED when state is initialised */

/**
 * @brief  Power-gate the rbAmp module.
 */
static void rbamp_power(bool on) {
    gpio_init(PIN_RBAMP_POWER);
    gpio_set_dir(PIN_RBAMP_POWER, GPIO_OUT);
    gpio_put(PIN_RBAMP_POWER, on);
    if (on) sleep_ms(300);          /* let the LDO + MCU boot */
}

/**
 * @brief  Arm the RTC alarm `seconds` ahead (with day-wrap) and enter dormant sleep.
 * @details Execution resumes at the reset vector via watchdog. On Pico W,
 *          deinit CYW43 BEFORE calling this — leaving the radio powered keeps
 *          the chip at ~10 mA instead of ~180 µA in dormant.
 */
static void enter_dormant(uint32_t seconds) {
    datetime_t now;
    if (!rtc_get_datetime(&now)) {
        /* RTC not running — fall back to a brute reboot via watchdog. */
        watchdog_reboot(0, 0, seconds * 1000U);
        while (1) tight_loop_contents();
    }

    /* Convert "now" + `seconds` to absolute calendar time WITH day-wrap.
     * Without this, alarms in the last 60 s of a day land at 00:0x today —
     * a time in the past — and the RTC never matches until 24 h later. */
    int total_sec = now.hour * 3600 + now.min * 60 + now.sec + (int)seconds;
    int day_offset = total_sec / 86400;
    total_sec %= 86400;

    datetime_t alarm = now;
    alarm.hour = total_sec / 3600;
    alarm.min  = (total_sec / 60) % 60;
    alarm.sec  =  total_sec % 60;
    if (day_offset > 0) {
        /* Bump the day; for periods ≤ 24 h this is at most one day forward. */
        alarm.day += day_offset;
        /* NOTE: this does not handle month/year roll-over; for periods ≤ 24 h
         * and well-set RTC the simplest fix is to step the day field; for a
         * production logger use a proper time_t arithmetic via mktime. */
    }

    sleep_run_from_xosc();
    sleep_goto_sleep_until(&alarm, NULL);
    /* Force a clean reset so main() re-runs from the top (with NOINIT state). */
    watchdog_reboot(0, 0, 1);
    while (1) tight_loop_contents();
}

int main(void) {
    stdio_init_all();
    sleep_ms(1000);

    /* Initialise persistent state on cold boot. Write `s_magic` LAST so a
     * crash mid-init leaves the marker unset and forces re-initialisation. */
    if (s_magic != 0xCAFEFEEDU) {
        s_total_wh    = 0.0;
        s_wake_count  = 0;
        s_primer_done = false;
        __dmb();                    /* publish other fields before the marker */
        s_magic       = 0xCAFEFEEDU;

        /* Seed the RTC with an arbitrary epoch so rtc_get_datetime() returns
         * a valid struct from the first call. Without this, the alarm math in
         * enter_dormant() reads garbage and sleep duration is undefined. */
        rtc_init();
        datetime_t epoch = { .year=2024, .month=1, .day=1,
                              .dotw=1,  .hour=0,  .min=0, .sec=0 };
        rtc_set_datetime(&epoch);
        sleep_us(64);               /* RTC needs a few cycles to latch */
    } else {
        rtc_init();
    }
    s_wake_count++;

    /* Power up rbAmp + I2C. */
    rbamp_power(true);
    rbamp_io_init(i2c0, 4, 5, 100000);
    if (rbamp_io_wait_ready(RB_ADDR, 2000) != PICO_OK) {
        printf("rbAmp not ready — sleeping\n");
        rbamp_power(false);
        enter_dormant(WAKE_SECONDS);
    }

    /* First wake-up after a cold boot: primer latch only. */
    if (!s_primer_done) {
        rbamp_io_write_u8(RB_ADDR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
        s_primer_done = true;
        printf("Primer done — entering dormant\n");
        rbamp_power(false);
        enter_dormant(WAKE_SECONDS);
    }

    /* Subsequent wake-ups: close the period and read its average power. */
    rbamp_io_write_u8(RB_ADDR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
    sleep_ms(50);

    uint8_t valid = 0;
    rbamp_io_read_u8(RB_ADDR, RB_REG_PERIOD_VALID, &valid);
    if ((valid & 0x01U) == 0) {
        printf("Stale snapshot — skipping cycle\n");
        rbamp_power(false);
        enter_dormant(WAKE_SECONDS);
    }

    float avg_p = 0.0f;
    rbamp_io_read_float_le(RB_ADDR, RB_REG_PERIOD_AVG_P0, &avg_p);
    /* dt approximation: dormant sleep + boot ≈ WAKE_SECONDS. For higher accuracy
     * read the RTC calendar on entry and exit. */
    float dt_s = (float)WAKE_SECONDS;
    s_total_wh += (double)avg_p * dt_s / 3600.0;

    /* Publish via Wi-Fi + MQTT (graceful fallback). */
    net_init();
    if (s_mqtt) {
        char payload[256];
        int n = snprintf(payload, sizeof(payload),
            "{\"wake\":%lu,\"dt_s\":%.1f,\"avg_p\":%.1f,\"energy_wh\":%.3f}",
            (unsigned long)s_wake_count, dt_s, avg_p, s_total_wh);
        mqtt_publish(s_mqtt, "rbamp/remote/state", payload, n,
                     /*qos*/ 1, /*retain*/ 1, NULL, NULL);
        printf("Published: %s\n", payload);
        sleep_ms(200);                   /* let LwIP flush the publish */
    } else {
        printf("Wi-Fi/MQTT failed — value buffered in NOINIT RAM\n");
    }

    /* Power down the CYW43 radio BEFORE dormant — leaving it powered keeps
     * the chip at ~10 mA instead of ~180 µA. Required for the headline
     * "6 months on a 2000 mAh cell" power budget. */
#if PICO_CYW43_SUPPORTED
    cyw43_arch_deinit();
#endif

    rbamp_power(false);
    enter_dormant(WAKE_SECONDS);
}
```

**Notes**:

- The `.uninitialized_data` section is **not zero-initialised** at boot; it survives the brief reset that follows dormant wake. A magic-number sentinel (`s_magic`) lets us distinguish cold boot (uninitialised RAM, random magic) from a wake.
- For permanent persistence across battery removal, periodically commit the total to internal flash using `flash_range_program()`. The Pico SDK has good examples in `pico-examples/flash/`.
- Pico W in dormant sleep with CYW43 powered down sits at ~1 mA. RP2040 alone (Pico) drops to ~180 µA. Use `cyw43_arch_deinit()` before sleep to power down the radio module.

---

## Example 10 — Time-of-use (TOU) tariff with SNTP wall-clock

**Goal**: bill energy under a time-of-use schedule. The Pico W keeps a real wall-clock via SNTP, splits each latched period into the right tariff bucket, resets daily totals at midnight, and publishes payloads with ISO timestamps.
**Hardware**: Pico W + one rbAmp UI1 + Wi-Fi.

`CMakeLists.txt` additions:

```cmake
target_link_libraries(example_10
    pico_stdlib
    hardware_i2c
    pico_cyw43_arch_lwip_threadsafe_background
    pico_lwip_mqtt
    pico_lwip_sntp)
```

`src/example_10.c`:

```c
#include <stdio.h>
#include <string.h>
#include <time.h>
#include "pico/stdlib.h"
#include "pico/cyw43_arch.h"
#include "lwip/apps/mqtt.h"
#include "lwip/apps/sntp.h"
#include "rbamp_io.h"

#define RB_ADDR  0x50U

/* CEST = UTC+2. For full DST handling, recompute on roll-over. */
#define TZ_OFFSET_HOURS  2
#define PEAK_START_HOUR  8
#define PEAK_END_HOUR    22

extern void net_init(void);
extern mqtt_client_t *s_mqtt;

typedef struct {
    double peak_wh, off_peak_wh;
    int    day_of_year;
} day_totals_t;
typedef struct {
    double peak_wh, off_peak_wh;
} lifetime_totals_t;

static day_totals_t      s_today    = { .day_of_year = -1 };
static lifetime_totals_t s_lifetime = { 0 };

/**
 * @brief  Start SNTP and wait until the C library `time(NULL)` returns plausible UTC.
 */
static bool sntp_sync(uint32_t timeout_ms) {
    sntp_setoperatingmode(SNTP_OPMODE_POLL);
    sntp_setservername(0, "pool.ntp.org");
    sntp_setservername(1, "time.google.com");
    sntp_init();

    absolute_time_t deadline = make_timeout_time_ms(timeout_ms);
    while (!time_reached(deadline)) {
        time_t now = time(NULL);
        if (now > 1700000000) {                  /* ≥ year 2023 */
            struct tm lt;
            gmtime_r(&now, &lt);
            printf("SNTP sync: %04d-%02d-%02d %02d:%02d:%02d UTC\n",
                   lt.tm_year + 1900, lt.tm_mon + 1, lt.tm_mday,
                   lt.tm_hour, lt.tm_min, lt.tm_sec);
            return true;
        }
        sleep_ms(500);
    }
    return false;
}

static struct tm local_time(void) {
    time_t now = time(NULL) + (TZ_OFFSET_HOURS * 3600);
    struct tm lt;
    gmtime_r(&now, &lt);
    return lt;
}

static bool is_peak_hour(int hour) {
    return (hour >= PEAK_START_HOUR) && (hour < PEAK_END_HOUR);
}

static void iso_now(char *out, size_t out_len) {
    struct tm lt = local_time();
    char sign = (TZ_OFFSET_HOURS >= 0) ? '+' : '-';
    int off = (TZ_OFFSET_HOURS < 0) ? -TZ_OFFSET_HOURS : TZ_OFFSET_HOURS;
    snprintf(out, out_len, "%04d-%02d-%02dT%02d:%02d:%02d%c%02d00",
             lt.tm_year + 1900, lt.tm_mon + 1, lt.tm_mday,
             lt.tm_hour, lt.tm_min, lt.tm_sec, sign, off);
}

/**
 * @brief  Add one period's energy to the right bucket; roll over the day at midnight.
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
            mqtt_publish(s_mqtt, "rbamp/tou/day_close", payload, n,
                         /*qos*/ 1, /*retain*/ 1, NULL, NULL);
            printf("Day closed: %s\n", payload);
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

int main(void) {
    stdio_init_all();
    sleep_ms(2000);
    net_init();
    sntp_sync(15000);

    rbamp_io_init(i2c0, 4, 5, 100000);
    rbamp_io_wait_ready(RB_ADDR, 2000);
    rbamp_io_write_u8(RB_ADDR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);  /* primer */
    uint32_t t_prev_ms = to_ms_since_boot(get_absolute_time());
    printf("TOU started: peak %02d:00..%02d:00 local\n",
           PEAK_START_HOUR, PEAK_END_HOUR);

    while (true) {
        sleep_ms(60 * 1000);

        rbamp_io_write_u8(RB_ADDR, RB_REG_COMMAND, RB_CMD_LATCH_PERIOD);
        uint32_t t_now_ms = to_ms_since_boot(get_absolute_time());
        sleep_ms(50);

        uint8_t valid = 0;
        rbamp_io_read_u8(RB_ADDR, RB_REG_PERIOD_VALID, &valid);
        if ((valid & 0x01U) == 0) {
            t_prev_ms = to_ms_since_boot(get_absolute_time());
            continue;
        }
        float avg_p = 0.0f;
        rbamp_io_read_float_le(RB_ADDR, RB_REG_PERIOD_AVG_P0, &avg_p);
        float dt_s = (float)(t_now_ms - t_prev_ms) / 1000.0f;
        double e_wh = (double)avg_p * dt_s / 3600.0;
        t_prev_ms = t_now_ms;

        struct tm lt = local_time();
        if (lt.tm_year + 1900 < 2023) {
            printf("WARN: clock not synced — retrying SNTP\n");
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
        mqtt_publish(s_mqtt, "rbamp/tou/state", payload, n,
                     /*qos*/ 0, /*retain*/ 0, NULL, NULL);
        printf("%s\n", payload);
    }
}
```

**Notes**:

- `pico_lwip_sntp` plugs into the LwIP SNTP client; the SDK wires it to the C library `time(NULL)` so `gmtime_r()` works out of the box.
- For full DST handling, store the TZ offset in a small lookup, or compile-in a POSIX TZ string via the newlib `tzset()` mechanism if your build includes it.
- For STANDARD / PRO bidirectional metering, call `accumulate()` twice — once for consumption and once for export — and publish both peak/off-peak buckets.

---

## Example comparison

| # | Complexity | Output | MQTT | DRDY | Multi-module | Bidirectional | Persistence | Use case |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|---|
| 1 | minimal | USB stdio | — | — | — | — | — | smoke test |
| 2 | low | OLED | — | — | — | — | RAM | boxed meter |
| 3 | low | USB stdio | — | — | yes (3) | — | — | home monitoring |
| 4 | medium | — | yes | — | — | — | RAM | per-appliance in HA |
| 5 | medium | USB stdio | — | yes | — | yes (master) | RAM | solar home on BASIC tier |
| 6 | high | — | yes | — | yes (3) | yes | RAM | full home balance |
| 7 | medium | USB + SD (FATFS) | — | yes | — | — | SD card | event detection |
| 8 | medium | — | yes (+disco) | — | — | optional | RAM + MQTT | HA Auto-discovery |
| 9 | medium | — | yes | — | — | — | NOINIT RAM | off-grid / outdoor logger |
| 10 | medium | USB stdio | yes | — | — | optional | RAM + MQTT | TOU peak/off-peak with SNTP |

## Best practices

- **Always check return codes from `rbamp_io_*` calls.** I2C transactions can fail on long buses; production code should retry rather than ignore.
- **Keep GPIO IRQ callbacks tiny.** The Pico SDK uses a single shared callback for all GPIO IRQs — set a `volatile bool` flag and return. All I2C / printf work belongs in the main loop.
- **Use `i2c0` and `i2c1` independently.** rbAmp on `i2c0`, OLED on `i2c0`, SD card on SPI — keep peripherals on the bus they were initialised for.
- **Use `to_ms_since_boot(get_absolute_time())` for the master clock.** Microsecond precision via `to_us_since_boot()`. Monotonic, survives `sleep_ms()`.
- **`PERIOD_VALID` (`0x07`) must be checked after every `CMD_LATCH_PERIOD`.** Stale snapshots happen if the master polls faster than the firmware integrates.
- **Wait ≥ 50 ms after a latch** before reading `0xDC..0xEF`.
- **For multi-module setups**, `rbamp_io_broadcast_latch(tick, group)` via general call gives precise synchronisation. GC reception is **opt-in** per module (`FLEET_CONFIG.bit0`, default OFF). On NACK, fall back to per-module sequential latch.
- **For battery applications on Pico W**, deinit CYW43 (`cyw43_arch_deinit()`) before sleep to drop several mA. RP2040 dormant sleep is ~180 µA, RP2350 even lower.
- **Persist Wh totals to internal flash** with `flash_range_program()` if you need them to survive a full power loss. NOINIT RAM survives dormant sleep but not power cycling.
- **`printf` float support** is enabled by default in `pico_stdlib`. For RP2040, the SDK's `pico_float` implementation is used; for RP2350, hardware FPU support is on by default.

## What next

After working through these ten examples:

- For Arduino-style ESP32 development, see [10_arduino_examples.md](arduino-examples.md) (raw) or [17_arduino_library.md](https://rbamp.com/docs/modules-basic-standard-library-arduino-overview) (library).
- For MicroPython / CircuitPython (including on the Pico itself), see [12_micropython_examples.md](micropython-examples.md) (raw) or [18_micropython_library.md](https://rbamp.com/docs/modules-basic-standard-micropython-examples) (library).
- For native ESP-IDF, see [13_esp_idf_examples.md](esp-idf-examples.md) (raw) or [19_esp_idf_library.md](https://rbamp.com/docs/modules-basic-standard-esp-idf-overview) (library).
- For STM32 HAL, see [14_stm32_hal_examples.md](stm32-hal-examples.md).
- For ESPHome integration (declarative YAML), see [`tools/esphome-rbamp/docs/en/`](https://rbamp.com/docs/modules-basic-standard-esphome-overview).
- The formal I2C register specification used here lives in [11_api_reference.md](api-reference.md).
