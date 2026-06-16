# 16 · Python on Linux SBC Examples

This chapter is the SBC equivalent of [10_arduino_examples.md](arduino-examples.md), [12_micropython_examples.md](micropython-examples.md), [13_esp_idf_examples.md](esp-idf-examples.md), [14_stm32_hal_examples.md](stm32-hal-examples.md) and [15_pico_sdk_examples.md](pico-sdk-examples.md): the same ten scenarios, ported to **CPython on Linux** (Raspberry Pi OS, Armbian, generic ARM/x86 Linux).

> **These examples talk to rbAmp directly through the raw I2C register API.** They are intended for service-style integrations on Linux — Home Assistant gateways, OpenWrt routers, Docker / k3s containers, custom IoT bridges — where Python is the right tool for the job and direct register access is fine.
>
> **For a more convenient, structured interface — use the `rbamp` Python package** (see [chapter 20 · Python SBC Library](https://rbamp.com/docs/modules-basic-standard-python-overview)). The package wraps the register-level details behind a `RbAmp` class with named methods (`dev.voltage`, `dev.latch_period()`, `dev.read_period_snapshot()`, etc.), a per-channel `RbAmpEnergy` accumulator, an `asyncio` variant via `async for`, multi-module helpers, HA MQTT auto-discovery and systemd service templates in the bundled examples, and a `rbamp` CLI (`rbamp scan` / `read --watch` / `period`). The same package runs unmodified on MicroPython — see [chapter 18](https://rbamp.com/docs/modules-basic-standard-micropython-examples). Most projects should start from the package and fall back to the raw API only when needed.

## How Linux differs from a microcontroller

The single-biggest difference: **Linux already does a lot of the work** that the bare-metal examples wire up by hand. You don't initialise the I2C peripheral, you don't synchronise the wall clock, you don't fight for retained memory across reboots. All of that exists at the system level. Your Python script does measurement-and-business-logic and nothing else.

Concretely:

| Concern | Bare-metal MCU | Linux SBC |
|---|---|---|
| **I2C peripheral init** | `i2c_init()` / `HAL_I2C_Init()` | Already done by the kernel driver (`i2c-dev`). Open `/dev/i2c-N` as a file descriptor. |
| **Pull-ups** | Decide in board-design or via `gpio_pull_up()` | Decided in hardware. Linux does not touch them. |
| **Wall clock (UTC)** | Implement SNTP, call `gmtime_r()` | `systemd-timesyncd` / `chrony` keeps the clock synced automatically. `time.time()` returns real seconds-since-epoch. |
| **Persistence across reboots** | RTC backup registers, NOINIT RAM, NVM | The filesystem. Use a JSON file, SQLite, or `/var/lib/rbamp/`. |
| **Deep sleep** | `esp_deep_sleep_start()` / `HAL_PWR_EnterSTANDBYMode()` | Irrelevant. The SBC runs continuously; idle CPU costs are dominated by Wi-Fi / Ethernet anyway. Sleep is a foreign concept. |
| **DRDY ISR** | Pin IRQ in firmware | Userspace edge events via `gpiod` / `gpiozero`. Latency is a few hundred microseconds, dominated by kernel scheduling. |
| **MQTT** | Bring up Wi-Fi, hand-roll a TCP/TLS stack | `pip install paho-mqtt`. |
| **Daemon / supervisor** | While-loop forever | A `systemd` service unit, with auto-restart and journal logging. |
| **Permissions** | Not applicable | The user running your script must be in the `i2c` and `gpio` groups, or run as root. |
| **Build system** | CMake / `idf.py` / CubeIDE | `pip install -r requirements.txt` (or a `pyproject.toml`). |

If you've been writing firmware up to now, the mental shift is: **stop reaching for low-level APIs**. The OS already exposes everything as files or stdlib calls.

## Linux I2C setup (one-time)

Steps below are for **Raspberry Pi OS** with a Raspberry Pi 3/4/5; other SBCs (Orange Pi, BeagleBone, x86 + USB-I2C adapter) follow the same pattern with a different bus number.

### 1. Enable the I2C controller in the kernel

Raspberry Pi OS — easy path:

```bash
sudo raspi-config nonint do_i2c 0           # 0 = enable
```

Or edit the device tree manually:

```bash
sudo nano /boot/firmware/config.txt          # /boot/config.txt on older releases
# Add or uncomment the next line:
dtparam=i2c_arm=on
# For 400 kHz (Fast mode) — see also chapter 01:
dtparam=i2c_arm=on,i2c_arm_baudrate=100000

sudo reboot
```

After reboot, the kernel exposes the bus as a character device:

```bash
ls /dev/i2c-*
# -> /dev/i2c-1
```

For Pi 4/5 you can also enable additional buses through `dtoverlay=i2c-bus,bus=N` entries — useful when you need a separate I2C for rbAmp away from HATs.

Other SBCs:

- **Orange Pi / NanoPi / Banana Pi**: similar `dtoverlay` or `extlinux.conf` knobs depending on the BSP.
- **BeagleBone Black**: `config-pin P9.19 i2c` / `P9.20 i2c` for I2C-2; node is `/dev/i2c-2`.
- **x86 / generic Linux**: a USB-I2C adapter (FT232H, MCP2221A, CH341) shows up as `/dev/i2c-N` after the right kernel module is loaded.

### 2. Install kernel-level access tools (optional but useful)

```bash
sudo apt update
sudo apt install -y i2c-tools
i2cdetect -y 1                              # scan bus 1
```

`i2cdetect` should show `50` at the rbAmp's default address:

```
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
...
50: 50 -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- --
```

### 3. Userspace permissions

Add your user to the `i2c` group so you do not need `sudo` for every script:

```bash
sudo usermod -aG i2c $USER
# Log out and back in for the group change to take effect.
```

For DRDY / GPIO access:

```bash
sudo usermod -aG gpio $USER                  # Raspberry Pi
sudo usermod -aG dialout $USER               # often required for serial / USB-CDC
```

Verify:

```bash
groups
# -> ... i2c gpio ...
```

### 4. Python packages

Use a virtual environment to keep dependencies isolated:

```bash
python3 -m venv ~/venv-rbamp
source ~/venv-rbamp/bin/activate

pip install --upgrade pip
pip install smbus2 gpiozero paho-mqtt
# Optional extras used in some examples:
pip install luma.oled            # SSD1306 / SH1106
pip install RPi.GPIO             # alternative to gpiozero on Pi
pip install python-periphery     # cross-SBC I2C / GPIO without smbus2
```

For a published project use `pyproject.toml` (PEP 621):

```toml
[project]
name = "my-rbamp-app"
version = "0.1.0"
dependencies = [
    "smbus2>=0.4.0",
    "paho-mqtt>=2.0,<3.0",          # 2.x: callback API version must be passed explicitly
    "gpiozero>=2.0",
]
```

> **paho-mqtt 2.x note** — the 2.0 release made the callback-API version a required constructor argument. All examples in this chapter pass `mqtt.CallbackAPIVersion.VERSION1` to keep the simple legacy signatures (`on_connect(client, userdata, flags, rc, properties=None)`); if you prefer v2 (with `ReasonCode` + `ConnectFlags`), pass `CallbackAPIVersion.VERSION2` and update the callbacks accordingly.

### 5. Time

By default `systemd-timesyncd` keeps the system clock in UTC sync with NTP. Verify:

```bash
timedatectl
#                Local time: Thu 2026-05-22 18:30:01 CEST
#            Universal time: Thu 2026-05-22 16:30:01 UTC
#  System clock synchronized: yes
#                NTP service: active
```

In Python you then call `time.time()` (UTC seconds since epoch), `datetime.datetime.now(timezone.utc)`, `time.localtime()` — and that's the wall-clock you need. **No SNTP code in your script**.

### 6. SBC quick-reference

| SBC | I2C node | Group | DRDY notes |
|---|---|---|---|
| Raspberry Pi 3/4/5 | `/dev/i2c-1` | `i2c`, `gpio` | `gpiozero` recommended |
| BeagleBone Black | `/dev/i2c-2` | `i2c`, `gpio` | `Adafruit_BBIO` / `libgpiod` |
| Orange Pi 4/5 | `/dev/i2c-3` (varies) | `i2c`, `gpio` | `OPi.GPIO` or `libgpiod` |
| Generic x86 + USB-I2C | `/dev/i2c-3..N` | `i2c` | DRDY via the adapter's GPIO (FT232H supports it) |

The rest of this chapter assumes Raspberry Pi 4 / 5 + Raspberry Pi OS — but every example works on any SBC after the corresponding bus number and GPIO library substitution.

---

## Examples table of contents

| # | Title | Difficulty | Output | MQTT | DRDY | Multi-module | Bidirectional | Use case |
|:---:|---|:---:|:---:|:---:|:---:|:---:|:---:|---|
| 1 | [Quick read](#example-1--quick-read-minimal) | minimal | Console | — | — | — | — | Smoke test |
| 2 | [60-second energy meter on OLED](#example-2--60-second-energy-meter-on-oled) | low | SSD1306 | — | — | — | — | Boxed Wh counter |
| 3 | [Multi-module monitor](#example-3--multi-module-monitor) | low | Console | — | — | yes (3) | — | Whole-home monitoring |
| 4 | [Per-appliance energy tracker (UI3)](#example-4--per-appliance-energy-tracker-ui3) | medium | — | yes | — | — | — | Sub-metering in HA |
| 5 | [Master-side bidirectional on a BASIC module](#example-5--master-side-bidirectional-on-a-basic-module) | medium | Console | — | yes | — | yes (master) | Solar home on BASIC tier |
| 6 | [Home energy balance](#example-6--home-energy-balance) | high | — | yes | — | yes (3) | yes | Full home balance |
| 7 | [Power-event detection](#example-7--power-event-detection) | medium | File log | — | yes | — | — | Appliance event log |
| 8 | [MQTT publisher with HA Auto-discovery](#example-8--mqtt-publisher-with-home-assistant-auto-discovery) | medium | — | yes (+disco) | — | — | optional | Drop-in HA integration |
| 9 | [systemd daemon with SQLite persistence](#example-9--systemd-daemon-with-sqlite-persistence) | medium | journald + DB | yes | — | — | — | Production logger |
| 10 | [Time-of-use (TOU) tariff with system clock](#example-10--time-of-use-tou-tariff-with-system-clock) | medium | Console | yes | — | — | optional | Peak / off-peak tariff metering |

---

## Common helpers (`rbamp_io.py`)

Place this file alongside your scripts (or import from a package). To save space, it is not repeated in the examples below.

```python
"""rbAmp raw-I2C helpers for Linux CPython.

Uses smbus2 — the most widely available userspace I2C library on Linux.
For older systems, ``python3-smbus`` (the official Linux i2c-tools binding)
exposes a nearly identical API.

For cross-SBC portability without smbus2, swap in python-periphery; the
helpers are thin enough to port in a few minutes.
"""
from __future__ import annotations

import struct
import time
from typing import Optional

from smbus2 import SMBus, i2c_msg


# ---------- rbAmp register addresses (subset used in the examples) ----------
REG_STATUS         = 0x00
REG_COMMAND        = 0x01
REG_VERSION        = 0x03
REG_CT_MODEL       = 0x05
REG_PERIOD_VALID   = 0x07
REG_AC_FREQ        = 0x20
REG_I2C_ADDRESS    = 0x30
REG_U_RMS          = 0x86
REG_U_PEAK         = 0x8A
REG_I0_RMS         = 0x8E
REG_I1_RMS         = 0x92
REG_I2_RMS         = 0x96
REG_P0_REAL        = 0xA6
REG_PF0            = 0xB2
REG_Q0             = 0xD0
REG_PERIOD_AVG_P1  = 0xC2
REG_PERIOD_AVG_P2  = 0xC6
REG_DATA_VALID     = 0xCE
REG_PERIOD_AVG_P0  = 0xDC
REG_PERIOD_MAX_P   = 0xE0
REG_PERIOD_LATCH_MS= 0xEC

CMD_RESET          = 0x01
CMD_SAVE_GAINS     = 0x26
CMD_LATCH_PERIOD   = 0x27


class RbAmpIO:
    """Thin wrapper around an SMBus / address pair.

    Holds the bus handle and the target slave address, exposing one helper
    per primitive operation. Designed to be composed into larger classes
    (or used directly in scripts).
    """

    def __init__(self, bus: SMBus, addr: int = 0x50) -> None:
        """Bind to an already-opened SMBus and a 7-bit address.

        :param bus: An ``smbus2.SMBus(N)`` instance.
        :param addr: 7-bit slave address (default 0x50).
        """
        self.bus = bus
        self.addr = addr

    # ----- I2C primitives -----

    def read_u8(self, reg: int) -> int:
        """Read one byte from a register."""
        return self.bus.read_byte_data(self.addr, reg)

    def write_u8(self, reg: int, val: int) -> None:
        """Write one byte to a register."""
        self.bus.write_byte_data(self.addr, reg, val & 0xFF)

    def read_float_le(self, reg: int) -> float:
        """Read a little-endian float32 from four consecutive registers.

        rbAmp supports READ auto-increment (a single burst-read is valid for
        atomicity). The per-byte form below is shown for clarity. WRITE
        transactions, by contrast, are byte-at-a-time only. See chapter 11.
        """
        b = bytes(self.read_u8(reg + i) for i in range(4))
        return struct.unpack("<f", b)[0]

    def read_u32_le(self, reg: int) -> int:
        """Read a little-endian uint32 from four consecutive registers."""
        b = bytes(self.read_u8(reg + i) for i in range(4))
        return struct.unpack("<I", b)[0]

    # ----- High-level helpers -----

    def probe(self) -> bool:
        """Return True if the slave acks a single-byte read of REG_STATUS."""
        try:
            self.bus.read_byte_data(self.addr, REG_STATUS)
            return True
        except OSError:
            return False

    def wait_ready(self, timeout_s: float = 2.0) -> None:
        """Block until DATA_VALID bit 0 reads 1, or raise TimeoutError."""
        deadline = time.monotonic() + timeout_s
        while time.monotonic() < deadline:
            try:
                if self.read_u8(REG_DATA_VALID) & 0x01:
                    return
            except OSError:
                pass
            time.sleep(0.05)
        raise TimeoutError(f"rbAmp at 0x{self.addr:02X} not ready in {timeout_s}s")


def broadcast_latch(bus: SMBus, tick16: int, group: int = 0) -> bool:
    """Broadcast CMD_LATCH_PERIOD to rbAmp modules with GC reception enabled.

    Canonical 5-byte general-call frame: A5 27 group tick_lo tick_hi.
    Slaves reject any first byte != 0xA5. GC reception is opt-in per
    module (FLEET_CONFIG.bit0, default OFF). See chapter 11 §6.3.2.

    Returns True on ACK; False on NACK (caller must fall back to
    per-module sequential latch).
    """
    msg = i2c_msg.write(0x00, [
        0xA5,                            # frame magic
        CMD_LATCH_PERIOD,                # opcode 0x27
        group & 0xFF,                    # group_id (0 = all-call)
        tick16 & 0xFF,                   # tick_lo
        (tick16 >> 8) & 0xFF,            # tick_hi
    ])
    try:
        bus.i2c_rdwr(msg)
        return True
    except OSError:
        return False
```

---

## Example 1 — Quick read (minimal)

**Goal**: print U, I, P, PF to the console once per second.
**Hardware**: any SBC + one rbAmp UI1 on `/dev/i2c-1`.

`example_01.py`:

```python
#!/usr/bin/env python3
"""rbAmp quick-read smoke test."""
import time
from smbus2 import SMBus
from rbamp_io import RbAmpIO, REG_VERSION, REG_U_RMS, REG_I0_RMS, REG_P0_REAL, REG_PF0

I2C_BUS = 1
RB_ADDR = 0x50

with SMBus(I2C_BUS) as bus:
    meter = RbAmpIO(bus, RB_ADDR)
    meter.wait_ready()                              # wait for first RT window
    print(f"rbAmp version: 0x{meter.read_u8(REG_VERSION):02X}")

    while True:
        u  = meter.read_float_le(REG_U_RMS)         # RMS voltage,  V
        i_ = meter.read_float_le(REG_I0_RMS)        # RMS current,  A
        p  = meter.read_float_le(REG_P0_REAL)       # active power, W (signed)
        pf = meter.read_float_le(REG_PF0)           # power factor, signed
        print(f"U={u:.1f}V  I={i_:.3f}A  P={p:+.1f}W  PF={pf:+.3f}")
        time.sleep(1)
```

Run:

```bash
python3 example_01.py
# rbAmp version: 0x04
# U=230.1V  I=0.262A  P=+60.2W  PF=+0.998
# U=229.9V  I=0.262A  P=+60.1W  PF=+0.998
# ...
```

If you see `PermissionError: [Errno 13]` on `/dev/i2c-1`, your user is not in the `i2c` group — fix per the Linux setup section above.

---

## Example 2 — 60-second energy meter on OLED

**Goal**: Wh counter refreshed once per minute, shown on a 128×64 SSD1306 sharing the rbAmp I2C bus.
**Hardware**: SBC + rbAmp + SSD1306 OLED (`0x3C`) on the same bus.
**Accounting**: unidirectional (BASIC tier).
**Library**: `luma.oled` for the display (`pip install luma.oled`).

```python
#!/usr/bin/env python3
"""60-second energy meter on a 128x64 SSD1306, shared I2C bus."""
import time
from smbus2 import SMBus
from luma.core.interface.serial import i2c as luma_i2c
from luma.oled.device import ssd1306
from luma.core.render import canvas

from rbamp_io import (
    RbAmpIO, REG_COMMAND, REG_PERIOD_VALID,
    REG_PERIOD_AVG_P0, CMD_LATCH_PERIOD,
)

I2C_BUS = 1
RB_ADDR = 0x50

# luma opens its own connection but still drives /dev/i2c-1 — coexists with smbus2.
serial = luma_i2c(port=I2C_BUS, address=0x3C)
oled = ssd1306(serial, width=128, height=64)


def draw(avg_p: float, dt_s: float, total_wh: float) -> None:
    """Render the three lines."""
    with canvas(oled) as c:
        c.text((0, 0),  f"P:  {avg_p:6.1f} W", fill="white")
        c.text((0, 16), f"dt: {dt_s:6.1f} s",  fill="white")
        c.text((0, 32), f"{total_wh:.2f} Wh",  fill="white")


with SMBus(I2C_BUS) as bus:
    meter = RbAmpIO(bus, RB_ADDR)
    meter.wait_ready()

    # Primer latch — discard its snapshot.
    meter.write_u8(REG_COMMAND, CMD_LATCH_PERIOD)
    t_prev = time.monotonic()
    total_wh = 0.0

    while True:
        time.sleep(60)

        # 1) Final latch — closes the period [t_prev .. now].
        meter.write_u8(REG_COMMAND, CMD_LATCH_PERIOD)
        t_now = time.monotonic()
        time.sleep(0.05)                              # let firmware finalise snapshot

        # 2) Verify the snapshot is fresh.
        if (meter.read_u8(REG_PERIOD_VALID) & 0x01) == 0:
            print("WARN: stale snapshot")
            t_prev = time.monotonic()
            continue

        # 3) Read time-averaged power, channel 0.
        avg_p = meter.read_float_le(REG_PERIOD_AVG_P0)

        # 4) Energy uses the MASTER clock — same idea as on a microcontroller,
        # except `time.monotonic()` is the Linux equivalent of `HAL_GetTick()`.
        dt_s = t_now - t_prev
        e_wh = avg_p * dt_s / 3600.0
        total_wh += e_wh
        t_prev = t_now

        draw(avg_p, dt_s, total_wh)
        print(f"avg_P={avg_p:.1f}  E_period={e_wh:.3f}  total={total_wh:.3f}")
```

---

## Example 3 — Multi-module monitor

**Goal**: poll three modules (main feed, water heater, AC) and print the total.
**Hardware**: SBC + 3 × rbAmp UI1 at `0x50`, `0x51`, `0x52`.

```python
#!/usr/bin/env python3
"""Round-robin polling of three rbAmp modules."""
import time
from smbus2 import SMBus
from rbamp_io import RbAmpIO, REG_U_RMS, REG_P0_REAL

I2C_BUS = 1
MODULES = [
    (0x50, "Mains "),
    (0x51, "Boiler"),
    (0x52, "AC    "),
]

with SMBus(I2C_BUS) as bus:
    meters = [(RbAmpIO(bus, addr), label) for addr, label in MODULES]

    # Probe every module before entering the read loop.
    for meter, label in meters:
        if not meter.probe():
            raise SystemExit(f"{label} at 0x{meter.addr:02X} not found")
    print("All modules OK")

    while True:
        total_p = 0.0
        print("---")
        for meter, label in meters:
            u = meter.read_float_le(REG_U_RMS)
            p = meter.read_float_le(REG_P0_REAL)
            print(f"[{label}] U={u:.1f}  P={p:+.1f} W")
            total_p += p
        print(f"TOTAL: {total_p:.1f} W")
        time.sleep(2)
```

`probe()` calls a 1-byte `read_byte_data` and catches `OSError(EREMOTEIO)` — the Linux equivalent of an I2C NACK.

---

## Example 4 — Per-appliance energy tracker (UI3)

**Goal**: a UI3 module with three CT clamps on three different loads on the same phase; independent Wh counters published to MQTT every minute.
**Hardware**: SBC + 1 × rbAmp UI3 + MQTT broker (Mosquitto, EMQX, HA Mosquitto add-on).

```python
#!/usr/bin/env python3
"""Per-appliance UI3 energy tracker with MQTT publishing."""
import json
import time

import paho.mqtt.client as mqtt
from smbus2 import SMBus

from rbamp_io import (
    RbAmpIO, REG_COMMAND, REG_PERIOD_VALID,
    REG_PERIOD_AVG_P0, REG_PERIOD_AVG_P1, REG_PERIOD_AVG_P2,
    CMD_LATCH_PERIOD,
)

I2C_BUS    = 1
RB_ADDR    = 0x50
CH_NAMES   = ("main", "heatpump", "lights")
MQTT_HOST  = "192.168.1.10"

client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION1, client_id="rbamp-ui3")
try:
    client.connect(MQTT_HOST, 1883, keepalive=60)
except OSError as e:
    print(f"MQTT broker unreachable at boot ({e}); will retry in loop_start()")
client.loop_start()                            # paho's thread reconnects automatically


def mqtt_publish(name: str, avg_p: float, e_wh: float) -> None:
    """Publish one channel's state to MQTT."""
    payload = json.dumps({"power": round(avg_p, 1), "energy": round(e_wh, 4)})
    client.publish(f"rbamp/{name}/state", payload, qos=0, retain=False)


with SMBus(I2C_BUS) as bus:
    meter = RbAmpIO(bus, RB_ADDR)
    meter.wait_ready()

    meter.write_u8(REG_COMMAND, CMD_LATCH_PERIOD)        # primer
    t_prev = time.monotonic()
    total_wh = [0.0, 0.0, 0.0]

    while True:
        time.sleep(60)

        meter.write_u8(REG_COMMAND, CMD_LATCH_PERIOD)
        t_now = time.monotonic()
        time.sleep(0.05)

        if (meter.read_u8(REG_PERIOD_VALID) & 0x01) == 0:
            t_prev = time.monotonic()
            continue

        # PERIOD_AVG_P_W per channel: I0 at 0xDC, I1 at 0xC2, I2 at 0xC6.
        avg_p = (
            meter.read_float_le(REG_PERIOD_AVG_P0),
            meter.read_float_le(REG_PERIOD_AVG_P1),
            meter.read_float_le(REG_PERIOD_AVG_P2),
        )
        dt_s = t_now - t_prev
        t_prev = t_now

        for ch in range(3):
            e = avg_p[ch] * dt_s / 3600.0
            total_wh[ch] += e
            print(f"[{CH_NAMES[ch]}] avg_P={avg_p[ch]:+.1f} W  E_total={total_wh[ch]:.3f} Wh")
            mqtt_publish(CH_NAMES[ch], avg_p[ch], total_wh[ch])
```

---

## Example 5 — Master-side bidirectional on a BASIC module

**Goal**: implement bidirectional accounting on the master while the module is BASIC firmware. The SBC reacts to the DRDY falling edge through `gpiozero` and splits consumption / export.
**Hardware**: Raspberry Pi + rbAmp UI1; DRDY wired to GPIO 15 (BCM numbering).

```python
#!/usr/bin/env python3
"""Master-side bidirectional accounting via DRDY GPIO IRQ."""
import time
from threading import Event

from gpiozero import Button
from smbus2 import SMBus

from rbamp_io import RbAmpIO, REG_P0_REAL

I2C_BUS  = 1
RB_ADDR  = 0x50
PIN_DRDY = 15                  # BCM

# Wrap the bus in a context manager so the /dev/i2c-N fd is released cleanly
# on any unhandled exception (Ctrl-C, NACK retry exhaustion, etc.).
with SMBus(I2C_BUS) as bus:
    meter  = RbAmpIO(bus, RB_ADDR)
    meter.wait_ready()

    # gpiozero treats DRDY as a "Button":
    #   pull_up=True -> the line idles HIGH (matches rbAmp's open-drain output)
    #   bounce_time=None — we want the raw falling edges, not debounced
    drdy = Button(PIN_DRDY, pull_up=True, bounce_time=None)
    data_ready = Event()

    def on_drdy() -> None:
        """gpiozero callback — only sets the flag. All I/O happens in the main loop."""
        data_ready.set()

    drdy.when_pressed = on_drdy

    e_consumed_wh = 0.0
    e_exported_wh = 0.0
    t_last = time.monotonic()
    t_last_print = t_last
    print("Bidirectional accumulator started")

    while True:
        if not data_ready.wait(timeout=1.0):
            continue                                # no edge in the last second
        data_ready.clear()

        t_now = time.monotonic()
        p = meter.read_float_le(REG_P0_REAL)        # signed P_real
        dt_s = t_now - t_last
        t_last = t_now

        if p > 0:
            e_consumed_wh += p * dt_s / 3600.0
        else:
            e_exported_wh += -p * dt_s / 3600.0

        # Summary print every 5 s.
        if t_now - t_last_print >= 5.0:
            t_last_print = t_now
            print(f"P={p:+8.1f}W   consumed={e_consumed_wh:.4f} Wh  "
                  f"exported={e_exported_wh:.4f} Wh  "
                  f"net={(e_consumed_wh - e_exported_wh):+.4f} Wh")
```

**Notes**:

- `threading.Event` is the natural synchronisation primitive — `set()` from the callback, `wait()` from the main loop. Linux scheduling latency is well below the 200 ms RT-window cadence, so no edge is lost.
- For older Pi OS releases without `gpiozero`, swap in `RPi.GPIO.add_event_detect(15, GPIO.FALLING, callback=on_drdy)`. For non-Pi SBCs use `gpiod.LineRequest` (libgpiod 2.x) — the rest of the script is unchanged.

---

## Example 6 — Home energy balance

**Goal**: full balance dashboard. Three modules:
- rbAmp #1 on the main feed (STANDARD / PRO — bidirectional)
- rbAmp #2 on the solar inverter output (BASIC OK)
- rbAmp #3 UI3 on three large loads (HP, AC, EV)

Master uses `broadcast_latch()` for synchronous snapshots and publishes a retained balance to MQTT every minute.

```python
#!/usr/bin/env python3
"""Home energy balance via synchronous broadcast latch."""
import json
import time

import paho.mqtt.client as mqtt
from smbus2 import SMBus

from rbamp_io import (
    RbAmpIO, broadcast_latch,
    REG_PERIOD_VALID, REG_PERIOD_AVG_P0, REG_PERIOD_AVG_P1, REG_PERIOD_AVG_P2,
)

I2C_BUS    = 1
ADDR_MAINS = 0x50                # bidirectional (STANDARD/PRO)
ADDR_SOLAR = 0x51                # generation only (BASIC ok)
ADDR_LOADS = 0x52                # UI3
MQTT_HOST  = "192.168.1.10"

# PERIOD_AVG_P_NEG register for the mains module — see the SKU datasheet
# of the STANDARD/PRO product. Set to None on BASIC modules.
from typing import Optional
REG_PERIOD_AVG_P_NEG_MAINS: Optional[int] = None     # Optional[int] for Py 3.9+ compat

client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION1, client_id="home-balance")
try:
    client.connect(MQTT_HOST, 1883, keepalive=60)
except OSError as e:
    print(f"MQTT broker unreachable at boot ({e}); will retry in loop_start()")
client.loop_start()


def safe_read_float(meter: RbAmpIO, reg: int) -> float:
    """Return 0.0 if the read fails (e.g. NACK from a BASIC module)."""
    try:
        return meter.read_float_le(reg)
    except OSError:
        return 0.0


totals = {
    "mains_in_wh":    0.0,
    "mains_out_wh":   0.0,
    "solar_total_wh": 0.0,
    "loads_wh":       [0.0, 0.0, 0.0],
}

with SMBus(I2C_BUS) as bus:
    mains = RbAmpIO(bus, ADDR_MAINS)
    solar = RbAmpIO(bus, ADDR_SOLAR)
    loads = RbAmpIO(bus, ADDR_LOADS)
    for m in (mains, solar, loads):
        m.wait_ready()

    gc_tick = 0
    if not broadcast_latch(bus, gc_tick):
        # GC NACK → fall back to per-module sequential latch.
        for m in (mains, solar, loads):
            m.write_u8(REG_COMMAND, CMD_LATCH_PERIOD)
    gc_tick += 1
    t_prev = time.monotonic()
    print("Home balance started")

    while True:
        time.sleep(60)

        # Synchronous latch — all modules snapshot at the same instant.
        if not broadcast_latch(bus, gc_tick):
            for m in (mains, solar, loads):
                m.write_u8(REG_COMMAND, CMD_LATCH_PERIOD)
        gc_tick = (gc_tick + 1) & 0xFFFF
        t_now = time.monotonic()
        time.sleep(0.05)
        dt_s = t_now - t_prev
        t_prev = t_now

        # ---- MAINS (bidirectional) ----
        if mains.read_u8(REG_PERIOD_VALID) & 0x01:
            p_consume = safe_read_float(mains, REG_PERIOD_AVG_P0)
            p_export  = (safe_read_float(mains, REG_PERIOD_AVG_P_NEG_MAINS)
                         if REG_PERIOD_AVG_P_NEG_MAINS is not None else 0.0)
            totals["mains_in_wh"]  += p_consume * dt_s / 3600.0
            totals["mains_out_wh"] += p_export  * dt_s / 3600.0

        # ---- SOLAR ----
        if solar.read_u8(REG_PERIOD_VALID) & 0x01:
            p = safe_read_float(solar, REG_PERIOD_AVG_P0)
            totals["solar_total_wh"] += p * dt_s / 3600.0

        # ---- LOADS (UI3) ----
        if loads.read_u8(REG_PERIOD_VALID) & 0x01:
            for i, reg in enumerate((REG_PERIOD_AVG_P0, REG_PERIOD_AVG_P1, REG_PERIOD_AVG_P2)):
                p = safe_read_float(loads, reg)
                totals["loads_wh"][i] += p * dt_s / 3600.0

        # Derived balance figures.
        total_consumed  = (totals["mains_in_wh"]
                           + totals["solar_total_wh"]
                           - totals["mains_out_wh"])
        solar_self_used = max(0.0, totals["solar_total_wh"] - totals["mains_out_wh"])

        print(f"MAINS  in={totals['mains_in_wh']:.2f}  out={totals['mains_out_wh']:.2f}")
        print(f"SOLAR  gen={totals['solar_total_wh']:.2f}  "
              f"self-used={solar_self_used:.2f}  exported={totals['mains_out_wh']:.2f}")
        print(f"LOADS  HP={totals['loads_wh'][0]:.2f}  "
              f"AC={totals['loads_wh'][1]:.2f}  EV={totals['loads_wh'][2]:.2f}")
        print(f"TOTAL  household consumed={total_consumed:.2f} Wh")

        payload = json.dumps({
            "mains_in":       round(totals["mains_in_wh"], 3),
            "mains_out":      round(totals["mains_out_wh"], 3),
            "solar":          round(totals["solar_total_wh"], 3),
            "self_used":      round(solar_self_used, 3),
            "total_consumed": round(total_consumed, 3),
            "hp":             round(totals["loads_wh"][0], 3),
            "ac":             round(totals["loads_wh"][1], 3),
            "ev":             round(totals["loads_wh"][2], 3),
        })
        client.publish("home/energy/balance", payload, qos=1, retain=True)
```

---

## Example 7 — Power-event detection

**Goal**: on every DRDY edge, compare instantaneous P to an EMA and append significant deviations to a rotating log file.
**Hardware**: any SBC + rbAmp + DRDY wired to GPIO 15.
**Logging**: Python's stdlib `logging` with `RotatingFileHandler` — survives across reboots, no SD-card driver involved.

```python
#!/usr/bin/env python3
"""Power-event detection with rotating-file logging."""
import logging
import logging.handlers
import time
from threading import Event

from gpiozero import Button
from smbus2 import SMBus

from rbamp_io import RbAmpIO, REG_P0_REAL

I2C_BUS  = 1
RB_ADDR  = 0x50
PIN_DRDY = 15

EMA_ALPHA = 0.05
EVENT_THRESHOLD_W = 200.0
LOG_PATH = "/var/log/rbamp/events.log"   # ensure the directory exists and is writeable

# Configure a rotating logger — 5 files × 1 MB.
handler = logging.handlers.RotatingFileHandler(
    LOG_PATH, maxBytes=1_000_000, backupCount=5)
handler.setFormatter(logging.Formatter("%(asctime)s  %(message)s",
                                       "%Y-%m-%dT%H:%M:%S"))
log = logging.getLogger("rbamp.events")
log.addHandler(handler)
log.setLevel(logging.INFO)

# Wrap the bus in a context manager so the /dev/i2c-N fd is released cleanly
# on any unhandled exception.
with SMBus(I2C_BUS) as bus:
    meter = RbAmpIO(bus, RB_ADDR)
    meter.wait_ready()

    # Seed the EMA with current power so the very first sample does not look like an event.
    p_ema = meter.read_float_le(REG_P0_REAL)

    drdy = Button(PIN_DRDY, pull_up=True, bounce_time=None)
    data_ready = Event()
    drdy.when_pressed = data_ready.set

    print("Event detector started; log:", LOG_PATH)

    while True:
        if not data_ready.wait(timeout=1.0):
            continue
        data_ready.clear()

        p = meter.read_float_le(REG_P0_REAL)
        delta = p - p_ema
        p_ema = (1 - EMA_ALPHA) * p_ema + EMA_ALPHA * p

        if abs(delta) > EVENT_THRESHOLD_W:
            event_type = "TURN_ON" if delta > 0 else "TURN_OFF"
            line = f"{event_type}  delta={delta:+.1f}W  P={p:.1f}W  EMA={p_ema:.1f}W"
            print(line)
            log.info(line)                          # MUST be inside the `if` —
                                                    # otherwise NameError when
                                                    # delta is below threshold.
```

> The directory must exist with the right permissions:
> ```bash
> sudo mkdir -p /var/log/rbamp && sudo chown $USER /var/log/rbamp
> ```
> Or write to `~/.local/state/rbamp/events.log` if you do not want root-managed paths.

---

## Example 8 — MQTT publisher with Home Assistant Auto-discovery

**Goal**: drop-in HA integration. The SBC publishes HA Auto-discovery configs (retained) and emits a JSON state every minute.
**Hardware**: any SBC + one rbAmp UI1 + MQTT broker.

```python
#!/usr/bin/env python3
"""rbAmp MQTT publisher with Home Assistant Auto-discovery."""
import json
import time

import paho.mqtt.client as mqtt
from smbus2 import SMBus

from rbamp_io import (
    RbAmpIO,
    REG_COMMAND, REG_PERIOD_VALID, REG_PERIOD_AVG_P0,
    REG_U_RMS, REG_I0_RMS, REG_PF0, REG_Q0, REG_AC_FREQ,
    CMD_LATCH_PERIOD,
)

I2C_BUS     = 1
RB_ADDR     = 0x50
DEVICE_ID   = "rbamp_main"
DEVICE_NAME = "Mains rbAmp"
MQTT_HOST   = "192.168.1.10"
MQTT_USER   = None     # set to a string to enable basic auth
MQTT_PASS   = None

DEVICE_BLOCK = {
    "identifiers":  [DEVICE_ID],
    "name":         DEVICE_NAME,
    "manufacturer": "rbAmp",
    "model":        "rbAmp UI*",
}


def publish_discovery_sensor(client: mqtt.Client, key: str, friendly: str,
                             unit: str | None, dev_class: str | None,
                             state_class: str) -> None:
    """Publish one HA discovery config (retained)."""
    cfg = {
        "name":            f"{DEVICE_NAME} {friendly}",
        "unique_id":       f"{DEVICE_ID}_{key}",
        "state_topic":     f"rbamp/{DEVICE_ID}/state",
        "value_template":  "{{ value_json." + key + " }}",
        "state_class":     state_class,
        "device":          DEVICE_BLOCK,
    }
    if unit:      cfg["unit_of_measurement"] = unit
    if dev_class: cfg["device_class"] = dev_class
    client.publish(f"homeassistant/sensor/{DEVICE_ID}/{key}/config",
                   json.dumps(cfg), qos=1, retain=True)


def publish_discovery_all(client: mqtt.Client) -> None:
    """Publish the full HA discovery set for one rbAmp module."""
    publish_discovery_sensor(client, "voltage",        "Voltage",        "V",   "voltage",        "measurement")
    publish_discovery_sensor(client, "current",        "Current",        "A",   "current",        "measurement")
    publish_discovery_sensor(client, "power",          "Power",          "W",   "power",          "measurement")
    publish_discovery_sensor(client, "energy",         "Energy",         "Wh",  "energy",         "total_increasing")
    publish_discovery_sensor(client, "frequency",      "Frequency",      "Hz",  "frequency",      "measurement")
    publish_discovery_sensor(client, "power_factor",   "Power Factor",   None,  "power_factor",   "measurement")
    publish_discovery_sensor(client, "apparent_power", "Apparent Power", "VA",  "apparent_power", "measurement")
    publish_discovery_sensor(client, "reactive_power", "Reactive Power", "var", "reactive_power", "measurement")


def on_connect(client, userdata, flags, rc, properties=None):
    """Republish discovery on every (re)connect — retained, so HA picks it up."""
    print(f"MQTT connected rc={rc}")
    publish_discovery_all(client)


client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION1,
                     client_id=DEVICE_ID, protocol=mqtt.MQTTv5)
if MQTT_USER:
    client.username_pw_set(MQTT_USER, MQTT_PASS)
client.on_connect = on_connect
try:
    client.connect(MQTT_HOST, 1883, keepalive=60)
except OSError as e:
    print(f"MQTT broker unreachable at boot ({e}); will retry in loop_start()")
client.loop_start()

with SMBus(I2C_BUS) as bus:
    meter = RbAmpIO(bus, RB_ADDR)
    meter.wait_ready()

    meter.write_u8(REG_COMMAND, CMD_LATCH_PERIOD)        # primer
    t_prev = time.monotonic()
    total_wh = 0.0

    while True:
        time.sleep(60)

        meter.write_u8(REG_COMMAND, CMD_LATCH_PERIOD)
        t_now = time.monotonic()
        time.sleep(0.05)

        if (meter.read_u8(REG_PERIOD_VALID) & 0x01) == 0:
            t_prev = time.monotonic()
            continue

        avg_p = meter.read_float_le(REG_PERIOD_AVG_P0)
        dt_s  = t_now - t_prev
        total_wh += avg_p * dt_s / 3600.0
        t_prev = t_now

        # Live RT values for the state payload.
        u  = meter.read_float_le(REG_U_RMS)
        i_ = meter.read_float_le(REG_I0_RMS)
        pf = meter.read_float_le(REG_PF0)
        q  = meter.read_float_le(REG_Q0)
        freq = meter.read_u8(REG_AC_FREQ)

        payload = json.dumps({
            "voltage":        round(u, 1),
            "current":        round(i_, 3),
            "power":          round(avg_p, 1),
            "energy":         round(total_wh, 3),
            "frequency":      int(freq),
            "power_factor":   round(pf, 3),
            "apparent_power": round(u * i_, 1),
            "reactive_power": round(q, 1),
        })
        client.publish(f"rbamp/{DEVICE_ID}/state", payload, qos=0, retain=False)
        print(f"Published: {payload}")
```

After the first publish Home Assistant auto-creates a `Mains rbAmp` device with eight sensors, including an `Energy` entity that drops straight into the Energy dashboard.

---

## Example 9 — systemd daemon with SQLite persistence

> This example replaces the "battery-powered deep-sleep logger" from the microcontroller chapters. On Linux the SBC runs continuously, so the right pattern is a **systemd-managed daemon** with auto-restart on failure and a small **SQLite database** for persistence across power cycles.

**Goal**: a long-running daemon that latches a period every minute, writes the result to SQLite, and publishes a state to MQTT. The process restarts cleanly after a reboot, picking up the accumulated total from the database.
**Hardware**: any SBC + one rbAmp UI1.

### Daemon (`rbamp_daemon.py`)

```python
#!/usr/bin/env python3
"""rbAmp daemon — periodic latch + SQLite persistence + MQTT."""
import json
import logging
import signal
import sqlite3
import sys
import time
from contextlib import closing

import paho.mqtt.client as mqtt
from smbus2 import SMBus

from rbamp_io import (
    RbAmpIO, REG_COMMAND, REG_PERIOD_VALID,
    REG_PERIOD_AVG_P0, CMD_LATCH_PERIOD,
)

# ----- Configuration -----
I2C_BUS    = 1
RB_ADDR    = 0x50
DB_PATH    = "/var/lib/rbamp/energy.db"
MQTT_HOST  = "192.168.1.10"
PERIOD_S   = 60

# ----- Logging -----
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s  %(levelname)s  %(message)s",
    stream=sys.stdout,                          # systemd captures stdout into the journal
)
log = logging.getLogger("rbamp.daemon")


# ----- SQLite persistence -----

def db_init(conn: sqlite3.Connection) -> None:
    """Create the schema if it does not exist."""
    conn.executescript("""
        CREATE TABLE IF NOT EXISTS energy_state (
            id          INTEGER PRIMARY KEY CHECK (id = 1),
            total_wh    REAL NOT NULL DEFAULT 0
        );
        INSERT OR IGNORE INTO energy_state(id, total_wh) VALUES (1, 0);

        CREATE TABLE IF NOT EXISTS energy_log (
            ts          INTEGER NOT NULL,
            dt_s        REAL    NOT NULL,
            avg_p_w     REAL    NOT NULL,
            e_wh        REAL    NOT NULL,
            total_wh    REAL    NOT NULL
        );
        CREATE INDEX IF NOT EXISTS idx_energy_log_ts ON energy_log(ts);
    """)
    conn.commit()


def db_load_total(conn: sqlite3.Connection) -> float:
    """Return the accumulated total_wh persisted across restarts."""
    row = conn.execute("SELECT total_wh FROM energy_state WHERE id = 1").fetchone()
    return float(row[0]) if row else 0.0


def db_record(conn: sqlite3.Connection, dt_s: float, avg_p: float,
              e_wh: float, total: float) -> None:
    """Append one period sample and update the running total atomically."""
    with conn:                                  # implicit BEGIN/COMMIT
        conn.execute("UPDATE energy_state SET total_wh = ? WHERE id = 1", (total,))
        conn.execute(
            "INSERT INTO energy_log(ts, dt_s, avg_p_w, e_wh, total_wh) "
            "VALUES (?, ?, ?, ?, ?)",
            (int(time.time()), dt_s, avg_p, e_wh, total),
        )


# ----- Signal handling for clean shutdown -----

_running = True

def _shutdown(signum, frame):
    """Handle SIGTERM (systemd stop) / SIGINT (Ctrl-C) gracefully."""
    global _running
    log.info("Shutdown signal %d received", signum)
    _running = False

signal.signal(signal.SIGTERM, _shutdown)
signal.signal(signal.SIGINT,  _shutdown)


# ----- Main -----

def main() -> int:
    client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION1, client_id="rbamp-daemon")
    try:
        client.connect(MQTT_HOST, 1883, keepalive=60)
    except OSError as e:
        log.warning("MQTT broker unreachable at boot (%s); paho will retry", e)
    client.loop_start()                        # paho's thread reconnects automatically

    with SMBus(I2C_BUS) as bus, closing(sqlite3.connect(DB_PATH)) as conn:
        db_init(conn)
        total_wh = db_load_total(conn)
        log.info("Resumed total_wh = %.3f Wh", total_wh)

        meter = RbAmpIO(bus, RB_ADDR)
        meter.wait_ready()

        meter.write_u8(REG_COMMAND, CMD_LATCH_PERIOD)    # primer
        t_prev = time.monotonic()

        while _running:
            # Sleep in 1-second slices so we can respond to SIGTERM quickly.
            for _ in range(PERIOD_S):
                if not _running:
                    break
                time.sleep(1)
            if not _running:
                break

            meter.write_u8(REG_COMMAND, CMD_LATCH_PERIOD)
            t_now = time.monotonic()
            time.sleep(0.05)

            if (meter.read_u8(REG_PERIOD_VALID) & 0x01) == 0:
                log.warning("Stale snapshot — skipping")
                t_prev = time.monotonic()
                continue

            avg_p = meter.read_float_le(REG_PERIOD_AVG_P0)
            dt_s  = t_now - t_prev
            e_wh  = avg_p * dt_s / 3600.0
            total_wh += e_wh
            t_prev = t_now

            db_record(conn, dt_s, avg_p, e_wh, total_wh)
            client.publish("rbamp/daemon/state",
                           json.dumps({
                               "dt_s": round(dt_s, 1),
                               "avg_p": round(avg_p, 1),
                               "energy_wh": round(total_wh, 3),
                           }),
                           qos=1, retain=True)
            log.info("avg_P=%+.1f W  E_period=%.3f Wh  total=%.3f Wh",
                     avg_p, e_wh, total_wh)

    log.info("Daemon stopped cleanly; total_wh = %.3f Wh", total_wh)
    client.loop_stop()
    client.disconnect()
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

### systemd unit (`/etc/systemd/system/rbamp.service`)

```ini
[Unit]
Description=rbAmp energy daemon
After=network-online.target time-sync.target
Wants=network-online.target time-sync.target

[Service]
Type=simple
ExecStart=/home/pi/venv-rbamp/bin/python /opt/rbamp/rbamp_daemon.py
User=pi
Group=i2c
Restart=on-failure
RestartSec=5
# Pre-create the persistence directory so the daemon never sees ENOENT.
RuntimeDirectoryMode=0755
StateDirectory=rbamp
StateDirectoryMode=0755

[Install]
WantedBy=multi-user.target
```

Enable + start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now rbamp.service
journalctl -fu rbamp.service                    # follow the log
```

The daemon:

- Restarts on crash (`Restart=on-failure`).
- Starts after both networking and clock sync (`After=network-online.target time-sync.target`).
- Survives reboots; the SQLite database under `/var/lib/rbamp/energy.db` preserves the running total.
- Responds to `systemctl stop rbamp` by trapping `SIGTERM` and writing a final state record.

This is the SBC equivalent of "deep sleep + RTC retained memory" — instead of saving precious µA, we lean on the OS to keep us up and on a battery-backed RTC to keep time.

---

## Example 10 — Time-of-use (TOU) tariff with system clock

**Goal**: bill energy under a time-of-use schedule. The SBC uses the **system clock** (already synced by `systemd-timesyncd`) — no SNTP code is needed.
**Hardware**: any SBC + one rbAmp UI1.

```python
#!/usr/bin/env python3
"""TOU tariff metering using the Linux system clock."""
import json
import time
from datetime import datetime, timezone
from zoneinfo import ZoneInfo               # stdlib since Python 3.9

import paho.mqtt.client as mqtt
from smbus2 import SMBus

from rbamp_io import (
    RbAmpIO, REG_COMMAND, REG_PERIOD_VALID, REG_PERIOD_AVG_P0,
    CMD_LATCH_PERIOD,
)

I2C_BUS   = 1
RB_ADDR   = 0x50
MQTT_HOST = "192.168.1.10"

# IANA timezone name — Python's stdlib handles DST automatically.
TZ = ZoneInfo("Europe/Berlin")
PEAK_START_HOUR = 8        # inclusive
PEAK_END_HOUR   = 22       # exclusive


def is_peak_hour(hour: int) -> bool:
    """Return True if `hour` (local) falls inside the peak window.

    For schedules that span midnight (peak = 22..6), change to
    ``return hour >= PEAK_START_HOUR or hour < PEAK_END_HOUR``.
    """
    return PEAK_START_HOUR <= hour < PEAK_END_HOUR


client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION1, client_id="rbamp-tou")
try:
    client.connect(MQTT_HOST, 1883, keepalive=60)
except OSError as e:
    print(f"MQTT broker unreachable at boot ({e}); will retry in loop_start()")
client.loop_start()

# Tariff buckets.
today = {"peak_wh": 0.0, "off_peak_wh": 0.0, "yday": -1}
lifetime = {"peak_wh": 0.0, "off_peak_wh": 0.0}


def accumulate(e_wh: float, now_local: datetime) -> None:
    """Add this period's energy to the right bucket and roll the day at midnight."""
    yday = now_local.timetuple().tm_yday
    if today["yday"] != yday:
        if today["yday"] >= 0:
            payload = json.dumps({
                "yday":        today["yday"],
                "peak_wh":     round(today["peak_wh"], 3),
                "off_peak_wh": round(today["off_peak_wh"], 3),
                "total_wh":    round(today["peak_wh"] + today["off_peak_wh"], 3),
            })
            client.publish("rbamp/tou/day_close", payload, qos=1, retain=True)
            print("Day closed:", payload)
        today["peak_wh"] = 0.0
        today["off_peak_wh"] = 0.0
        today["yday"] = yday

    if is_peak_hour(now_local.hour):
        today["peak_wh"]    += e_wh
        lifetime["peak_wh"] += e_wh
    else:
        today["off_peak_wh"]    += e_wh
        lifetime["off_peak_wh"] += e_wh


with SMBus(I2C_BUS) as bus:
    meter = RbAmpIO(bus, RB_ADDR)
    meter.wait_ready()
    meter.write_u8(REG_COMMAND, CMD_LATCH_PERIOD)        # primer
    t_prev = time.monotonic()
    print(f"TOU started: peak {PEAK_START_HOUR:02d}:00..{PEAK_END_HOUR:02d}:00 local")

    while True:
        time.sleep(60)

        meter.write_u8(REG_COMMAND, CMD_LATCH_PERIOD)
        t_now = time.monotonic()
        time.sleep(0.05)

        if (meter.read_u8(REG_PERIOD_VALID) & 0x01) == 0:
            t_prev = time.monotonic()
            continue

        avg_p = meter.read_float_le(REG_PERIOD_AVG_P0)
        dt_s  = t_now - t_prev
        e_wh  = avg_p * dt_s / 3600.0
        t_prev = t_now

        # The Linux system clock is already authoritative — no SNTP code required.
        now_local = datetime.now(TZ)
        accumulate(e_wh, now_local)
        peak = is_peak_hour(now_local.hour)

        payload = json.dumps({
            "ts":                now_local.isoformat(timespec="seconds"),
            "tariff":            "peak" if peak else "off_peak",
            "avg_p":             round(avg_p, 1),
            "period_wh":         round(e_wh, 4),
            "today_peak_wh":     round(today["peak_wh"], 3),
            "today_off_peak_wh": round(today["off_peak_wh"], 3),
            "life_peak_wh":      round(lifetime["peak_wh"], 3),
            "life_off_peak_wh":  round(lifetime["off_peak_wh"], 3),
        })
        client.publish("rbamp/tou/state", payload, qos=0, retain=False)
        print(payload)
```

**Notes**:

- `zoneinfo` is the modern Python 3.9+ stdlib timezone API. On Debian / Raspberry Pi OS it pulls IANA data from the system `tzdata` package — DST transitions are handled correctly.
- For older Pythons (3.8 and earlier) substitute `pytz`: `import pytz; TZ = pytz.timezone("Europe/Berlin")`.
- ISO 8601 timestamps with offset are produced by `datetime.isoformat(timespec="seconds")` — for example, `2026-05-22T18:30:00+02:00`.
- For STANDARD / PRO bidirectional metering, call `accumulate()` twice — once for consumption and once for export — and publish both peak/off-peak buckets.

---

## Example comparison

| # | Complexity | Output | MQTT | DRDY | Multi-module | Bidirectional | Persistence | Use case |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|---|
| 1 | minimal | Console | — | — | — | — | — | smoke test |
| 2 | low | OLED | — | — | — | — | RAM | boxed meter |
| 3 | low | Console | — | — | yes (3) | — | — | home monitoring |
| 4 | medium | — | yes | — | — | — | RAM | per-appliance in HA |
| 5 | medium | Console | — | yes | — | yes (master) | RAM | solar home on BASIC tier |
| 6 | high | — | yes | — | yes (3) | yes | RAM + MQTT (retained) | full home balance |
| 7 | medium | Rotating log file | — | yes | — | — | filesystem | event detection |
| 8 | medium | — | yes (+disco) | — | — | optional | RAM + MQTT | HA Auto-discovery |
| 9 | medium | journald + SQLite | yes | — | — | — | SQLite | production daemon |
| 10 | medium | Console | yes | — | — | optional | RAM + MQTT (day close) | TOU peak/off-peak |

## Best practices

- **Run scripts as a non-root user**, in the `i2c` and `gpio` groups. Avoid `sudo python ...` — it pollutes paths and breaks `pip --user` installs.
- **Use a virtual environment** (`python3 -m venv ...`) instead of installing packages with `sudo pip` system-wide.
- **Always catch `OSError`** around I2C reads — a transient NACK or a temporarily disconnected slave shows up as `OSError(EREMOTEIO)`. Production code retries; smoke-test code lets it crash so the failure is loud.
- **Use `time.monotonic()` for the master clock**. It is unaffected by `systemd-timesyncd` stepping the wall clock (e.g. on the first sync after boot). Use `time.time()` / `datetime` only for timestamps.
- **`PERIOD_VALID` (`0x07`) must be checked after every `CMD_LATCH_PERIOD`.** Same rule as on a microcontroller — Linux doesn't change rbAmp's protocol.
- **Wait ≥ 50 ms after a latch** (`time.sleep(0.05)`) before reading `0xDC..0xEF`.
- **For multi-module setups**, `broadcast_latch(bus, tick)` via general call gives precise synchronisation. GC reception is **opt-in** per module (`FLEET_CONFIG.bit0`, default OFF). On NACK, fall back to per-module sequential `CMD_LATCH_PERIOD`.
- **Persist totals to the filesystem** — JSON, SQLite, InfluxDB. Filesystem is the SBC equivalent of "RTC backup registers + NVM". Cheap and reliable.
- **Wrap the main loop with signal handlers** (`SIGTERM`, `SIGINT`) so `systemctl stop` shuts the daemon down cleanly with a final state write.
- **Log to stdout/stderr** in daemons — `journald` captures it automatically; you get rotation, filtering and `journalctl -u rbamp` for free.

## What next

After working through these ten examples:

- For Arduino-style ESP32 development, see [10_arduino_examples.md](arduino-examples.md).
- For MicroPython / CircuitPython on the embedded side, see [12_micropython_examples.md](micropython-examples.md).
- For native ESP-IDF firmware, see [13_esp_idf_examples.md](esp-idf-examples.md).
- For STM32 HAL, see [14_stm32_hal_examples.md](stm32-hal-examples.md).
- For Raspberry Pi Pico SDK (C, not the Linux-on-Pi case), see [15_pico_sdk_examples.md](pico-sdk-examples.md).
- For the high-level `rbamp` Python package (CPython on Linux SBC), see [20_python_sbc_library.md](https://rbamp.com/docs/modules-basic-standard-python-overview).
- For ESPHome integration (declarative YAML), see [`tools/esphome-rbamp/docs/en/`](https://rbamp.com/docs/modules-basic-standard-esphome-overview).
- The formal I2C register specification used here lives in [11_api_reference.md](api-reference.md).
