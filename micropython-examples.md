# 12 · MicroPython & CircuitPython Examples

This chapter is the Python equivalent of [10_arduino_examples.md](arduino-examples.md): the same ten scenarios, ported to MicroPython on ESP32 / RP2040 / ESP8266 / Pyboard, with CircuitPython deltas where the two dialects differ.

> **These examples talk to rbAmp directly through the raw I2C register API.** They are intended for native, embedded, or otherwise resource-constrained integrations where direct control is required and every byte on the bus matters.
>
> **For a more convenient, structured interface — use the `rbamp` MicroPython library** (see [chapter 18 · MicroPython Library](micropython-examples.md)). The library wraps the register-level details behind a high-level `RbAmp` class with named methods (`dev.voltage`, `dev.latch_period()`, `dev.read_period_snapshot()`, etc.), sync and `uasyncio` APIs, an automatic per-channel Wh accumulator, multi-module helpers, and application-level NACK retry + sanity-check discipline for ESP32 ports via `MachineI2CBackend(retry_attempts=…)` (see the bus-timing and retry guidance in [chapter 3 · API Reference](api-reference.md)). The same package runs unmodified on CPython — see [chapter 20 · Python SBC Library](python-overview.md). Most user-facing projects should start from the library and fall back to the raw API only when needed.

## MicroPython vs CircuitPython — quick reference

The Python language is the same; the device-access modules differ. The table below maps each concept used in the examples to the equivalent module / class on each dialect.

| Concept | MicroPython | CircuitPython |
|---|---|---|
| I2C bus | `from machine import I2C, Pin`<br>`i2c = I2C(0, scl=Pin(22), sda=Pin(21), freq=50000)` | `import board, busio`<br>`i2c = busio.I2C(board.SCL, board.SDA, frequency=50000)` |
| Single byte read | `i2c.readfrom_mem(addr, reg, 1)[0]` | `with i2c_device: i2c_device.write_then_readinto(bytes([reg]), buf, in_end=1)` (using `adafruit_bus_device.I2CDevice`) |
| Single byte write | `i2c.writeto_mem(addr, reg, bytes([val]))` | `i2c_device.write(bytes([reg, val]))` |
| GPIO input + IRQ | `pin = Pin(15, Pin.IN, Pin.PULL_UP)`<br>`pin.irq(trigger=Pin.IRQ_FALLING, handler=fn)` | `import digitalio, countio`<br>`btn = countio.Counter(board.D15, edge=countio.Edge.FALL)`<br>or `keypad.Keys([board.D15], ...)` |
| Sleep (sec / ms) | `time.sleep(s)` / `time.sleep_ms(ms)` | `time.sleep(s)` (no `sleep_ms`) |
| Monotonic ms / µs | `time.ticks_ms()`, `time.ticks_us()`, `time.ticks_diff(a, b)` | `time.monotonic_ns()` / `time.monotonic()` (no `ticks_diff`) |
| WiFi | `import network`<br>`wlan = network.WLAN(network.STA_IF)` | `import wifi`<br>`wifi.radio.connect(ssid, pwd)` (ESP32-S2/S3/C3 with native CP) |
| MQTT | `from umqtt.simple import MQTTClient` | `import adafruit_minimqtt.adafruit_minimqtt as mqtt` |
| NTP | `import ntptime; ntptime.settime()` | `import adafruit_ntp; ntp = adafruit_ntp.NTP(socketpool, server="...")` |
| Deep sleep | `import machine; machine.deepsleep(ms)` | `import alarm; alarm.exit_and_deep_sleep_until_alarms(time_alarm)` |
| RTC retained memory | `machine.RTC().memory(buf)` | `import alarm; alarm.sleep_memory[...]` |
| Async | `import uasyncio as asyncio` | `import asyncio` (CircuitPython 8+) |

Each example below is written in MicroPython. Where the CircuitPython equivalent differs by more than a single import line, an explicit "CircuitPython delta" block is given.

## Examples table of contents

| # | Title | Difficulty | Output | MQTT | DRDY | Multi-module | Bidirectional | Use case |
|:---:|---|:---:|:---:|:---:|:---:|:---:|:---:|---|
| 1 | [Quick read](#example-1--quick-read-minimal) | minimal | REPL | — | — | — | — | Smoke test |
| 2 | [60-second energy meter on OLED](#example-2--60-second-energy-meter-on-oled) | low | SSD1306 | — | — | — | — | Boxed Wh counter |
| 3 | [Multi-module monitor](#example-3--multi-module-monitor) | low | REPL | — | — | yes (3) | — | Whole-home monitoring |
| 4 | [Per-appliance energy tracker (UI3)](#example-4--per-appliance-energy-tracker-ui3) | medium | — | yes | — | — | — | Sub-metering in HA |
| 5 | [Master-side bidirectional on a BASIC module](#example-5--master-side-bidirectional-on-a-basic-module) | medium | REPL | — | yes | — | yes (master) | Solar home on BASIC tier |
| 6 | [Home energy balance](#example-6--home-energy-balance) | high | — | yes | — | yes (3) | yes | Full home balance |
| 7 | [Power-event detection](#example-7--power-event-detection) | medium | REPL + SD | — | yes | — | — | Appliance event log |
| 8 | [MQTT publisher with HA Auto-discovery](#example-8--mqtt-publisher-with-home-assistant-auto-discovery) | medium | — | yes (+disco) | — | — | optional | Drop-in HA integration |
| 9 | [Battery-powered remote logger with deep sleep](#example-9--battery-powered-remote-logger-with-deep-sleep) | medium | — | yes | — | — | — | Off-grid / outdoor meter |
| 10 | [Time-of-use (TOU) tariff with NTP wall-clock](#example-10--time-of-use-tou-tariff-with-ntp-wall-clock) | medium | REPL | yes | — | — | optional | Peak / off-peak tariff metering |

## Common header for all examples

Every example assumes these helpers are present in the same `.py` file (or imported from a `helpers.py`). To save space, the block is not repeated in each sketch.

```python
"""
rbAmp raw-I2C helpers for MicroPython.

For CircuitPython, replace the I2C bring-up:
    import board, busio
    from adafruit_bus_device.i2c_device import I2CDevice
    i2c_bus = busio.I2C(board.SCL, board.SDA, frequency=50000)
    # then use I2CDevice(i2c_bus, addr) for each rbAmp, see notes in each example.
"""

import struct
import time
from machine import I2C, Pin   # MicroPython


def rb_read_u8(i2c, addr, reg):
    """Read one byte from an rbAmp slave register.

    :param i2c:  machine.I2C instance.
    :param addr: 7-bit slave address (e.g. 0x50).
    :param reg:  register address inside the slave.
    :returns:    the byte stored in the register.
    """
    return i2c.readfrom_mem(addr, reg, 1)[0]


def rb_read_float_le(i2c, addr, reg):
    """Read a little-endian float32 from four consecutive registers.

    rbAmp supports READ auto-increment (a single burst-read is valid for
    atomicity). The per-byte form below is shown for clarity. WRITE
    transactions, by contrast, are byte-at-a-time only. See chapter 11.
    """
    b = bytes([rb_read_u8(i2c, addr, reg + i) for i in range(4)])
    return struct.unpack('<f', b)[0]


def rb_read_u32_le(i2c, addr, reg):
    """Read a little-endian uint32 from four consecutive registers."""
    b = bytes([rb_read_u8(i2c, addr, reg + i) for i in range(4)])
    return struct.unpack('<I', b)[0]


def rb_write_u8(i2c, addr, reg, val):
    """Write a single byte to an rbAmp slave register."""
    i2c.writeto_mem(addr, reg, bytes([val & 0xFF]))


def rb_broadcast_latch(i2c, tick16, group=0):
    """Broadcast CMD_LATCH_PERIOD to rbAmp modules with GC reception enabled.

    Canonical 5-byte general-call frame: A5 27 group tick_lo tick_hi.
    Slaves reject any first byte != 0xA5. GC reception is opt-in per
    module (FLEET_CONFIG.bit0, default OFF). See chapter 11 §6.3.2.

    Returns True on ACK; False on NACK (caller falls back to per-module).
    """
    try:
        i2c.writeto(0x00, bytes([0xA5, 0x27, group,
                                  tick16 & 0xFF, (tick16 >> 8) & 0xFF]))
        return True
    except OSError:
        return False


def rb_wait_ready(i2c, addr, timeout_ms=2000):
    """Block until DATA_VALID bit 0 is 1, or raise OSError on timeout."""
    t0 = time.ticks_ms()
    while time.ticks_diff(time.ticks_ms(), t0) < timeout_ms:
        if rb_read_u8(i2c, addr, 0xCE) & 0x01:
            return
        time.sleep_ms(50)
    raise OSError("rbAmp at 0x{:02X} not ready".format(addr))
```

### CircuitPython delta — common header

CircuitPython uses `adafruit_bus_device.i2c_device.I2CDevice` (per-slave wrapper) and lacks `time.ticks_ms`. Equivalent helpers:

```python
import time
import struct
import board, busio
from adafruit_bus_device.i2c_device import I2CDevice

def cp_read_u8(dev, reg):
    """Read one byte from one register (CircuitPython)."""
    buf = bytearray(1)
    with dev:
        dev.write_then_readinto(bytes([reg]), buf)
    return buf[0]

def cp_read_float_le(dev, reg):
    b = bytes(cp_read_u8(dev, reg + i) for i in range(4))
    return struct.unpack('<f', b)[0]

def cp_write_u8(dev, reg, val):
    with dev:
        dev.write(bytes([reg, val & 0xFF]))

def cp_broadcast_latch(i2c_bus, tick16, group=0):
    """General-call latch via raw bus access (CircuitPython).

    Canonical 5-byte GC frame: A5 27 group tick_lo tick_hi. Returns True
    on ACK; False on NACK (no module on the bus has GC enabled).
    """
    while not i2c_bus.try_lock():
        pass
    try:
        try:
            i2c_bus.writeto(0x00, bytes([0xA5, 0x27, group,
                                          tick16 & 0xFF, (tick16 >> 8) & 0xFF]))
            return True
        except OSError:
            return False
    finally:
        i2c_bus.unlock()

def cp_wait_ready(dev, timeout_s=2.0):
    t0 = time.monotonic()
    while time.monotonic() - t0 < timeout_s:
        if cp_read_u8(dev, 0xCE) & 0x01:
            return
        time.sleep(0.05)
    raise OSError("rbAmp not ready")
```

---

## Example 1 — Quick read (minimal)

**Goal**: print U, I, P, PF in a REPL once per second.
**Hardware**: ESP32 / RP2040 / Pico W + one rbAmp UI1.

```python
from machine import I2C, Pin
import time

# I2C bus 0 on ESP32 default pins.
i2c = I2C(0, scl=Pin(22), sda=Pin(21), freq=50000)

RB_ADDR = 0x50

# Wait until the first RT window has been computed.
rb_wait_ready(i2c, RB_ADDR)
ver = rb_read_u8(i2c, RB_ADDR, 0x03)
print("rbAmp version: 0x{:02X}".format(ver))

while True:
    u  = rb_read_float_le(i2c, RB_ADDR, 0x86)   # RMS voltage, V
    i_ = rb_read_float_le(i2c, RB_ADDR, 0x8E)   # RMS current, A
    p  = rb_read_float_le(i2c, RB_ADDR, 0xA6)   # active power, W (signed)
    pf = rb_read_float_le(i2c, RB_ADDR, 0xB2)   # power factor, signed
    print("U={:.1f}V  I={:.3f}A  P={:+.1f}W  PF={:+.3f}".format(u, i_, p, pf))
    time.sleep(1)
```

### CircuitPython delta

```python
import board, busio, time
from adafruit_bus_device.i2c_device import I2CDevice

i2c_bus = busio.I2C(board.SCL, board.SDA, frequency=50000)
dev = I2CDevice(i2c_bus, 0x50)

cp_wait_ready(dev)
print("rbAmp version: 0x{:02X}".format(cp_read_u8(dev, 0x03)))

while True:
    u  = cp_read_float_le(dev, 0x86)
    i_ = cp_read_float_le(dev, 0x8E)
    p  = cp_read_float_le(dev, 0xA6)
    pf = cp_read_float_le(dev, 0xB2)
    print("U={:.1f}V  I={:.3f}A  P={:+.1f}W  PF={:+.3f}".format(u, i_, p, pf))
    time.sleep(1)
```

---

## Example 2 — 60-second energy meter on OLED

**Goal**: Wh counter refreshed once per minute, shown on a 128×64 SSD1306.
**Hardware**: ESP32 + rbAmp + SSD1306 OLED on the same bus.
**Accounting**: unidirectional (BASIC tier).
**Library**: `ssd1306.py` from MicroPython examples (a standard one-file driver).

```python
from machine import I2C, Pin
import time
import ssd1306

i2c = I2C(0, scl=Pin(22), sda=Pin(21), freq=50000)
RB_ADDR = 0x50

oled = ssd1306.SSD1306_I2C(128, 64, i2c)        # OLED typically at 0x3C

rb_wait_ready(i2c, RB_ADDR)

# Primer latch — discard its result.
rb_write_u8(i2c, RB_ADDR, 0x01, 0x27)
t_prev_ms = time.ticks_ms()
total_wh = 0.0

while True:
    time.sleep(60)

    # 1) Final latch — closes the period [t_prev_ms .. now].
    rb_write_u8(i2c, RB_ADDR, 0x01, 0x27)
    t_now_ms = time.ticks_ms()
    time.sleep_ms(50)                            # let firmware finalise the snapshot

    # 2) Verify the snapshot is fresh.
    if (rb_read_u8(i2c, RB_ADDR, 0x07) & 0x01) == 0:
        print("WARN: stale snapshot")
        t_prev_ms = time.ticks_ms()
        continue

    # 3) Read time-averaged power, channel 0.
    avg_p = rb_read_float_le(i2c, RB_ADDR, 0xDC)

    # 4) Energy uses the MASTER clock (authoritative).
    dt_s = time.ticks_diff(t_now_ms, t_prev_ms) / 1000.0
    e_wh = avg_p * dt_s / 3600.0
    total_wh += e_wh
    t_prev_ms = t_now_ms

    # 5) Render on OLED.
    oled.fill(0)
    oled.text("P:  {:6.1f} W".format(avg_p), 0, 0)
    oled.text("dt: {:6.1f} s".format(dt_s),   0, 16)
    oled.text("E:  {:.2f} Wh".format(total_wh), 0, 32)
    oled.show()

    print("avg_P={:.1f}  E_period={:.3f}  total={:.3f}".format(avg_p, e_wh, total_wh))
```

> **CircuitPython delta**: use `adafruit_ssd1306` (`displayio` variant or `framebuf` variant). The OLED constructor differs but the rest of the loop is unchanged — replace `rb_*` calls with the `cp_*` helpers and use `time.monotonic()` for `dt_s`.

---

## Example 3 — Multi-module monitor

**Goal**: poll three modules (main feed, water heater, AC) and print the total.
**Hardware**: ESP32 + 3 × rbAmp UI1 at `0x50`, `0x51`, `0x52`.

```python
from machine import I2C, Pin
import time

i2c = I2C(0, scl=Pin(22), sda=Pin(21), freq=50000)
modules = ((0x50, "Mains "), (0x51, "Boiler"), (0x52, "AC    "))

# I2C scan — confirm every module is present.
present = i2c.scan()
for addr, label in modules:
    if addr not in present:
        raise RuntimeError("{} at 0x{:02X} not on bus".format(label, addr))
print("All modules OK:", [hex(a) for a in present])

while True:
    total_p = 0.0
    print("---")
    for addr, label in modules:
        u = rb_read_float_le(i2c, addr, 0x86)
        p = rb_read_float_le(i2c, addr, 0xA6)
        print("[{}] U={:.1f}  P={:+.1f} W".format(label, u, p))
        total_p += p
    print("TOTAL: {:.1f} W".format(total_p))
    time.sleep(2)
```

---

## Example 4 — Per-appliance energy tracker (UI3)

**Goal**: one UI3 module with three CT clamps on three different loads on the same phase. Independent Wh counters per load, published to MQTT every minute.
**Hardware**: ESP32 + 1 × rbAmp UI3.
**Library**: `umqtt.simple` (MicroPython) or `umqtt.robust` for auto-reconnect.

```python
from machine import I2C, Pin
import network, time
from umqtt.simple import MQTTClient

i2c = I2C(0, scl=Pin(22), sda=Pin(21), freq=50000)
RB_ADDR = 0x50
CH_NAMES = ("main", "heatpump", "lights")

# --- Wi-Fi ---
wlan = network.WLAN(network.STA_IF)
wlan.active(True)
wlan.connect("ssid", "password")
while not wlan.isconnected():
    time.sleep(0.5)

# --- MQTT ---
mqtt = MQTTClient(client_id="rbamp-ui3", server="192.168.1.10")
mqtt.connect()

def mqtt_publish(ch_name, avg_p, e_wh):
    """Publish one channel's state to MQTT.

    :param ch_name: short channel name (used in the topic).
    :param avg_p:   time-averaged power, W.
    :param e_wh:    accumulated channel energy, Wh.
    """
    topic = "rbamp/{}/state".format(ch_name).encode()
    payload = '{{"power":{:.1f},"energy":{:.4f}}}'.format(avg_p, e_wh).encode()
    mqtt.publish(topic, payload)

rb_wait_ready(i2c, RB_ADDR)
rb_write_u8(i2c, RB_ADDR, 0x01, 0x27)            # primer latch
t_prev_ms = time.ticks_ms()
total_wh = [0.0, 0.0, 0.0]

while True:
    time.sleep(60)

    rb_write_u8(i2c, RB_ADDR, 0x01, 0x27)
    t_now_ms = time.ticks_ms()
    time.sleep_ms(50)

    if (rb_read_u8(i2c, RB_ADDR, 0x07) & 0x01) == 0:
        t_prev_ms = time.ticks_ms()
        continue

    # PERIOD_AVG_P_W per channel: I0 at 0xDC, I1 at 0xC2, I2 at 0xC6.
    avg_p = (rb_read_float_le(i2c, RB_ADDR, 0xDC),
             rb_read_float_le(i2c, RB_ADDR, 0xC2),
             rb_read_float_le(i2c, RB_ADDR, 0xC6))

    dt_s = time.ticks_diff(t_now_ms, t_prev_ms) / 1000.0
    t_prev_ms = t_now_ms

    for ch in range(3):
        e = avg_p[ch] * dt_s / 3600.0
        total_wh[ch] += e
        print("[{}] avg_P={:+.1f} W  E_total={:.3f} Wh".format(
            CH_NAMES[ch], avg_p[ch], total_wh[ch]))
        mqtt_publish(CH_NAMES[ch], avg_p[ch], total_wh[ch])
```

> **CircuitPython delta**: replace `umqtt.simple.MQTTClient` with `adafruit_minimqtt.MQTT(...)` (needs a `socket_pool` and `ssl_context` — see Adafruit's WiFi guide). The metering logic is unchanged.

---

## Example 5 — Master-side bidirectional on a BASIC module

**Goal**: implement bidirectional accounting on the master while the module's firmware is BASIC (period accumulator clamps negative samples). Read RT `P_real` on every DRDY edge and split consumption / export.
**Hardware**: ESP32 + rbAmp UI1 on a load with a solar inverter + DRDY wired to a GPIO.

```python
from machine import I2C, Pin
import time

i2c  = I2C(0, scl=Pin(22), sda=Pin(21), freq=50000)
drdy = Pin(15, Pin.IN, Pin.PULL_UP)
RB_ADDR = 0x50

_data_ready = False
_e_consumed_wh = 0.0
_e_exported_wh = 0.0
_t_last_sample_ms = 0

def _drdy_isr(pin):
    """ISR for DRDY falling edge — only sets a flag (no I2C inside ISR)."""
    global _data_ready
    _data_ready = True

drdy.irq(trigger=Pin.IRQ_FALLING, handler=_drdy_isr)

rb_wait_ready(i2c, RB_ADDR)
_t_last_sample_ms = time.ticks_ms()
_last_print = time.ticks_ms()
print("Bidirectional accumulator started")

while True:
    if not _data_ready:
        time.sleep_ms(10)
        continue
    _data_ready = False

    t_now = time.ticks_ms()
    # Signed instantaneous P_real: + consume, − export.
    p = rb_read_float_le(i2c, RB_ADDR, 0xA6)
    dt_s = time.ticks_diff(t_now, _t_last_sample_ms) / 1000.0
    _t_last_sample_ms = t_now

    if p > 0:
        _e_consumed_wh += p * dt_s / 3600.0
    else:
        _e_exported_wh += -p * dt_s / 3600.0

    # Summary print every 5 s — not on every RT window.
    if time.ticks_diff(t_now, _last_print) >= 5000:
        _last_print = t_now
        print("P={:+8.1f}W   consumed={:.4f} Wh  exported={:.4f} Wh  net={:+.4f} Wh"
              .format(p, _e_consumed_wh, _e_exported_wh,
                      _e_consumed_wh - _e_exported_wh))
```

> **CircuitPython delta**: CircuitPython generally discourages user-defined GPIO IRQs. Use `countio.Counter(board.D15, edge=countio.Edge.FALL)` and poll its `count` from the main loop, or `keypad.Keys(...)` for edge events. The accumulator logic is identical.

---

## Example 6 — Home energy balance

**Goal**: a full balance dashboard. Three modules:
- rbAmp #1 on the main feed (STANDARD / PRO — bidirectional)
- rbAmp #2 on the solar inverter output
- rbAmp #3 UI3 on three large loads (HP, AC, EV)

The master computes `total_consumed = solar_self_used + grid_consumed` and `solar_generated = solar_self_used + grid_exported`.
**Hardware**: ESP32 + 3 × rbAmp + WiFi + MQTT.

```python
from machine import I2C, Pin
import network, time, ujson as json
from umqtt.simple import MQTTClient

i2c = I2C(0, scl=Pin(22), sda=Pin(21), freq=50000)
ADDR_MAINS = 0x50        # bidirectional (STANDARD / PRO)
ADDR_SOLAR = 0x51        # generation only
ADDR_LOADS = 0x52        # UI3

# PERIOD_AVG_P_NEG address for the mains module — see the SKU datasheet
# of the STANDARD / PRO product. Set to None on BASIC modules.
REG_PERIOD_AVG_P_NEG_MAINS = None

wlan = network.WLAN(network.STA_IF); wlan.active(True); wlan.connect("ssid", "password")
while not wlan.isconnected():
    time.sleep(0.5)
mqtt = MQTTClient("home-balance", "192.168.1.10"); mqtt.connect()

# Wait until all three modules report DATA_VALID.
for addr in (ADDR_MAINS, ADDR_SOLAR, ADDR_LOADS):
    rb_wait_ready(i2c, addr)

totals = {
    "mains_in_wh":    0.0,
    "mains_out_wh":   0.0,
    "solar_total_wh": 0.0,
    "loads_wh":       [0.0, 0.0, 0.0],
}

gc_tick = 0
if not rb_broadcast_latch(i2c, gc_tick):
    # GC NACK → fall back to per-module sequential latch.
    for addr in (ADDR_MAINS, ADDR_SOLAR, ADDR_LOADS):
        rb_write_u8(i2c, addr, 0x01, 0x27)
gc_tick += 1
t_prev_ms = time.ticks_ms()
print("Home balance started")

while True:
    time.sleep(60)

    if not rb_broadcast_latch(i2c, gc_tick):
        for addr in (ADDR_MAINS, ADDR_SOLAR, ADDR_LOADS):
            rb_write_u8(i2c, addr, 0x01, 0x27)
    gc_tick = (gc_tick + 1) & 0xFFFF
    t_now_ms = time.ticks_ms()
    time.sleep_ms(50)

    dt_s = time.ticks_diff(t_now_ms, t_prev_ms) / 1000.0
    t_prev_ms = t_now_ms

    # --- MAINS (bidirectional, STANDARD / PRO) ---
    if rb_read_u8(i2c, ADDR_MAINS, 0x07) & 0x01:
        avg_p_consume = rb_read_float_le(i2c, ADDR_MAINS, 0xDC)
        avg_p_export  = (rb_read_float_le(i2c, ADDR_MAINS, REG_PERIOD_AVG_P_NEG_MAINS)
                         if REG_PERIOD_AVG_P_NEG_MAINS is not None else 0.0)
        totals["mains_in_wh"]  += avg_p_consume * dt_s / 3600.0
        totals["mains_out_wh"] += avg_p_export  * dt_s / 3600.0

    # --- SOLAR (generation only) ---
    if rb_read_u8(i2c, ADDR_SOLAR, 0x07) & 0x01:
        avg_p = rb_read_float_le(i2c, ADDR_SOLAR, 0xDC)
        totals["solar_total_wh"] += avg_p * dt_s / 3600.0

    # --- LOADS (UI3, three channels) ---
    if rb_read_u8(i2c, ADDR_LOADS, 0x07) & 0x01:
        for i, reg in enumerate((0xDC, 0xC2, 0xC6)):     # I0, I1, I2
            p = rb_read_float_le(i2c, ADDR_LOADS, reg)
            totals["loads_wh"][i] += p * dt_s / 3600.0

    # --- Derived balance figures ---
    total_consumed  = (totals["mains_in_wh"] + totals["solar_total_wh"]
                      - totals["mains_out_wh"])
    solar_self_used = max(0.0, totals["solar_total_wh"] - totals["mains_out_wh"])

    print("MAINS  in={:.2f}  out={:.2f}".format(totals["mains_in_wh"],
                                                 totals["mains_out_wh"]))
    print("SOLAR  gen={:.2f}  self-used={:.2f}  exported={:.2f}".format(
          totals["solar_total_wh"], solar_self_used, totals["mains_out_wh"]))
    print("LOADS  HP={:.2f}  AC={:.2f}  EV={:.2f}".format(*totals["loads_wh"]))
    print("TOTAL  household consumed={:.2f} Wh".format(total_consumed))

    payload = json.dumps({
        "mains_in":       totals["mains_in_wh"],
        "mains_out":      totals["mains_out_wh"],
        "solar":          totals["solar_total_wh"],
        "self_used":      solar_self_used,
        "total_consumed": total_consumed,
        "hp":             totals["loads_wh"][0],
        "ac":             totals["loads_wh"][1],
        "ev":             totals["loads_wh"][2],
    })
    mqtt.publish(b"home/energy/balance", payload.encode(), retain=True)
```

**Notes**:

- `rb_broadcast_latch(i2c, tick)` ensures all three modules snapshot at the same instant — critical for an accurate balance. GC reception is **opt-in** (`FLEET_CONFIG.bit0`, default OFF, see chapter 02). On NACK, fall back to per-module sequential latch (skew ~1 ms/module on 50 kHz).
- `solar_self_used = solar_total − grid_exported`; clamp to zero to absorb measurement noise.

---

## Example 7 — Power-event detection

**Goal**: on every DRDY edge, compare instantaneous P to an EMA and log significant deviations to an SD card.
**Hardware**: ESP32 + rbAmp + SD-card (SPI-driven, e.g. via `sdcard.py` MicroPython driver).
**Use case**: automatic detection of large-load events (microwave, AC, kettle).

```python
from machine import I2C, Pin, SPI
import time, os, sdcard

i2c  = I2C(0, scl=Pin(22), sda=Pin(21), freq=50000)
drdy = Pin(15, Pin.IN, Pin.PULL_UP)
RB_ADDR = 0x50

# Mount the SD card on /sd (SPI bus 1, CS = GPIO 5).
spi = SPI(1, baudrate=1_000_000, sck=Pin(18), mosi=Pin(23), miso=Pin(19))
sd  = sdcard.SDCard(spi, Pin(5))
os.mount(sd, "/sd")

EMA_ALPHA = 0.05                                  # ~4 s time constant at 5 Hz
EVENT_THRESHOLD_W = 200.0

_data_ready = False
def _drdy_isr(pin):
    global _data_ready
    _data_ready = True
drdy.irq(trigger=Pin.IRQ_FALLING, handler=_drdy_isr)

rb_wait_ready(i2c, RB_ADDR)
# Seed the EMA with the current power so we do not log a spurious startup event.
p_ema = rb_read_float_le(i2c, RB_ADDR, 0xA6)
print("Event detector started")

logfile = open("/sd/events.log", "a")

while True:
    if not _data_ready:
        time.sleep_ms(10)
        continue
    _data_ready = False

    p = rb_read_float_le(i2c, RB_ADDR, 0xA6)
    delta = p - p_ema
    p_ema = (1 - EMA_ALPHA) * p_ema + EMA_ALPHA * p

    if abs(delta) > EVENT_THRESHOLD_W:
        event = "TURN_ON" if delta > 0 else "TURN_OFF"
        line = "{}  {}  delta={:+.1f} W  P={:.1f} W  EMA={:.1f} W\n".format(
            time.ticks_ms(), event, delta, p, p_ema)
        print(line, end="")
        logfile.write(line)
        logfile.flush()                          # ensure the line hits the card
```

> **CircuitPython delta**: use `adafruit_sdcard.SDCard` + `storage.mount(VfsFat(sdcard), "/sd")`. Replace the IRQ with `countio.Counter(board.D15, edge=countio.Edge.FALL)` polled from the main loop.

---

## Example 8 — MQTT publisher with Home Assistant Auto-discovery

**Goal**: a drop-in HA integration. The ESP32 connects to WiFi and an MQTT broker, publishes HA Auto-discovery configs (so the entities appear without any YAML), and emits a JSON state every minute.
**Hardware**: ESP32 + one rbAmp UI1 + WiFi network with an MQTT broker.
**Accounting**: unidirectional.

```python
from machine import I2C, Pin
import network, time, ujson as json
from umqtt.simple import MQTTClient

i2c = I2C(0, scl=Pin(22), sda=Pin(21), freq=50000)
RB_ADDR = 0x50

WIFI_SSID  = "ssid"
WIFI_PASS  = "password"
MQTT_HOST  = "192.168.1.10"
MQTT_USER  = "ha"
MQTT_PASS  = "secret"
DEVICE_ID  = "rbamp_main"
DEVICE_NAME = "Mains rbAmp"

# Bring up WiFi.
wlan = network.WLAN(network.STA_IF); wlan.active(True); wlan.connect(WIFI_SSID, WIFI_PASS)
while not wlan.isconnected():
    time.sleep(0.5)
print("WiFi up:", wlan.ifconfig()[0])

mqtt = MQTTClient(DEVICE_ID, MQTT_HOST, user=MQTT_USER, password=MQTT_PASS)
mqtt.connect()

DEVICE = {
    "identifiers": [DEVICE_ID],
    "name":        DEVICE_NAME,
    "manufacturer": "rbAmp",
    "model":       "rbAmp UI*",
}

def publish_discovery_sensor(key, friendly, unit, dev_class, state_class):
    """Publish one HA discovery config (retained).

    :param key:         short id used in the topic and value_template.
    :param friendly:    HA friendly-name suffix.
    :param unit:        unit_of_measurement, or None.
    :param dev_class:   HA device_class string, or None.
    :param state_class: 'measurement' or 'total_increasing'.
    """
    cfg = {
        "name":            "{} {}".format(DEVICE_NAME, friendly),
        "unique_id":       "{}_{}".format(DEVICE_ID, key),
        "state_topic":     "rbamp/{}/state".format(DEVICE_ID),
        "value_template":  "{{{{ value_json.{} }}}}".format(key),
        "state_class":     state_class,
        "device":          DEVICE,
    }
    if unit:      cfg["unit_of_measurement"] = unit
    if dev_class: cfg["device_class"] = dev_class
    topic = "homeassistant/sensor/{}/{}/config".format(DEVICE_ID, key).encode()
    mqtt.publish(topic, json.dumps(cfg).encode(), retain=True)

def publish_discovery_all():
    """Publish the full set of HA discovery configs for one rbAmp module."""
    publish_discovery_sensor("voltage",        "Voltage",        "V",   "voltage",        "measurement")
    publish_discovery_sensor("current",        "Current",        "A",   "current",        "measurement")
    publish_discovery_sensor("power",          "Power",          "W",   "power",          "measurement")
    publish_discovery_sensor("energy",         "Energy",         "Wh",  "energy",         "total_increasing")
    publish_discovery_sensor("frequency",      "Frequency",      "Hz",  "frequency",      "measurement")
    publish_discovery_sensor("power_factor",   "Power Factor",   None,  "power_factor",   "measurement")
    publish_discovery_sensor("apparent_power", "Apparent Power", "VA",  "apparent_power", "measurement")
    publish_discovery_sensor("reactive_power", "Reactive Power", "var", "reactive_power", "measurement")

publish_discovery_all()

rb_wait_ready(i2c, RB_ADDR)
rb_write_u8(i2c, RB_ADDR, 0x01, 0x27)            # primer latch
t_prev_ms = time.ticks_ms()
total_wh = 0.0

while True:
    time.sleep(60)

    # Latch + read period snapshot.
    rb_write_u8(i2c, RB_ADDR, 0x01, 0x27)
    t_now_ms = time.ticks_ms()
    time.sleep_ms(50)
    if (rb_read_u8(i2c, RB_ADDR, 0x07) & 0x01) == 0:
        t_prev_ms = time.ticks_ms()
        continue
    avg_p = rb_read_float_le(i2c, RB_ADDR, 0xDC)
    dt_s  = time.ticks_diff(t_now_ms, t_prev_ms) / 1000.0
    total_wh += avg_p * dt_s / 3600.0
    t_prev_ms = t_now_ms

    # Live RT values.
    u  = rb_read_float_le(i2c, RB_ADDR, 0x86)
    i_ = rb_read_float_le(i2c, RB_ADDR, 0x8E)
    pf = rb_read_float_le(i2c, RB_ADDR, 0xB2)
    q  = rb_read_float_le(i2c, RB_ADDR, 0xD0)
    freq = rb_read_u8(i2c, RB_ADDR, 0x20)

    payload = json.dumps({
        "voltage":        u,
        "current":        i_,
        "power":          avg_p,
        "energy":         total_wh,
        "frequency":      freq,
        "power_factor":   pf,
        "apparent_power": u * i_,
        "reactive_power": q,
    })
    mqtt.publish("rbamp/{}/state".format(DEVICE_ID).encode(), payload.encode())
    print("Published:", payload)
```

> **CircuitPython delta**: use `adafruit_minimqtt.MQTT` with a socket pool from `wifi.radio`. Discovery payloads are constructed the same way. See Adafruit's MQTT + HA discovery guide for the socket boilerplate.

---

## Example 9 — Battery-powered remote logger with deep sleep

**Goal**: an outdoor / off-grid logger that wakes every 10 minutes, latches a period, publishes via MQTT, then goes back to deep sleep. Average current a few mA — a single Li-ion cell lasts months.
**Hardware**: ESP32 (or ESP32-S3) + one rbAmp UI1 + small Li-ion + WiFi.
**Accounting**: unidirectional. Energy survives deep-sleep via RTC memory.

```python
from machine import I2C, Pin, RTC, deepsleep
import network, time, struct, ujson as json
from umqtt.simple import MQTTClient

SLEEP_MS = 10 * 60 * 1000                        # 10-minute wake interval
RB_ADDR  = 0x50

# RTC retained memory layout (with a magic-marker sentinel so garbage RTC RAM
# after a brown-out or watchdog reset is NOT mistaken for valid state):
#   uint32  magic == 0xCAFEFEED     (4 bytes)
#   double  total_wh                (8 bytes)
#   uint32  wake_count              (4 bytes)
#   uint8   primer_done             (1 byte) + 3 bytes pad
#
# NOTE on dt: we do NOT persist a microsecond counter because time.ticks_us()
# is monotonic SINCE BOOT — it resets to 0 on every deep-sleep wake, so
# carrying the previous value across sleep is meaningless. dt is approximated
# by the configured SLEEP_MS interval; for tighter accuracy use machine.RTC()
# .datetime() which DOES survive deep sleep.
_MAGIC = 0xCAFEFEED
_FMT   = "<IdIBxxx"

def _rtc_load():
    """Return (total_wh, wake_count, primer_done) from RTC memory.

    On any non-deep-sleep boot (power-on, brown-out, watchdog) RTC RAM may
    hold garbage — the magic marker tells us whether to trust it.
    """
    try:
        buf = bytes(RTC().memory())
    except Exception:
        return (0.0, 0, 0)
    if len(buf) < struct.calcsize(_FMT):
        return (0.0, 0, 0)
    magic, total, wakes, primer = struct.unpack(_FMT, buf[:struct.calcsize(_FMT)])
    if magic != _MAGIC:
        return (0.0, 0, 0)
    return (total, wakes, primer)

def _rtc_save(total_wh, wake_count, primer_done):
    RTC().memory(struct.pack(_FMT, _MAGIC, total_wh, wake_count, primer_done))

def wifi_connect(timeout_s=8):
    """Bring up WiFi with a hard timeout so deep-sleep is never blocked forever."""
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    wlan.connect("ssid", "password")
    t0 = time.ticks_ms()
    while not wlan.isconnected():
        if time.ticks_diff(time.ticks_ms(), t0) > timeout_s * 1000:
            return None
        time.sleep_ms(100)
    return wlan

def publish_measurement(wake_count, dt_s, avg_p, total_e):
    """Publish one period sample via MQTT, retained."""
    mqtt = MQTTClient("rbamp-remote", "192.168.1.10")
    mqtt.connect()
    payload = json.dumps({
        "wake":      wake_count,
        "dt_s":      round(dt_s, 1),
        "avg_p":     round(avg_p, 1),
        "energy_wh": round(total_e, 3),
    })
    mqtt.publish(b"rbamp/remote/state", payload.encode(), retain=True)
    mqtt.disconnect()
    print("Published:", payload)

# ===== Main wake-handler — runs on every wake-up =====

i2c = I2C(0, scl=Pin(22), sda=Pin(21), freq=50000)
total_wh, wake_count, primer_done = _rtc_load()
wake_count += 1

rb_wait_ready(i2c, RB_ADDR)

# First wake-up after a cold boot: primer latch only.
if not primer_done:
    rb_write_u8(i2c, RB_ADDR, 0x01, 0x27)
    _rtc_save(total_wh, wake_count, 1)
    print("Primer done — sleeping until first real period")
    deepsleep(SLEEP_MS)

# Subsequent wake-ups: close the period and read its average power.
rb_write_u8(i2c, RB_ADDR, 0x01, 0x27)
time.sleep_ms(50)

if (rb_read_u8(i2c, RB_ADDR, 0x07) & 0x01) == 0:
    print("WARN: stale snapshot — skipping this cycle")
    _rtc_save(total_wh, wake_count, 1)
    deepsleep(SLEEP_MS)

avg_p = rb_read_float_le(i2c, RB_ADDR, 0xDC)
# dt approximation: deep sleep + boot ≈ SLEEP_MS. time.ticks_us() resets
# every wake so a stored "previous" value would be meaningless; the
# configured interval is accurate to single-digit-% on the ESP32 RTC.
dt_s = SLEEP_MS / 1000.0
total_wh += avg_p * dt_s / 3600.0
_rtc_save(total_wh, wake_count, 1)

# Publish; fall back gracefully if WiFi is unreachable.
if wifi_connect(timeout_s=8):
    try:
        publish_measurement(wake_count, dt_s, avg_p, total_wh)
    except Exception as e:
        print("MQTT failed, value buffered in RTC:", e)
else:
    print("WiFi failed — value buffered in RTC, retry next wake")

print("Deep sleep")
deepsleep(SLEEP_MS)
```

**Power budget (typical, ESP32-WROOM)**:

- Awake duration per wake: ~3 s.
- Wake current: ~80 mA average → ~0.07 mAh per wake.
- Sleep current: ~10 µA (RTC retained domains).
- One wake every 10 min → ~10 mAh/day → a single 2000 mAh Li-ion lasts ~6 months.

> **CircuitPython delta**: use `alarm.time.TimeAlarm(monotonic_time=...)` + `alarm.exit_and_deep_sleep_until_alarms(alarm)` instead of `machine.deepsleep`. Persist `total_wh` in `alarm.sleep_memory` (a `bytearray` survives deep sleep). NVM (`microcontroller.nvm`) survives even power loss but is wear-limited.

---

## Example 10 — Time-of-use (TOU) tariff with NTP wall-clock

**Goal**: bill energy under a time-of-use schedule (different rates for peak and off-peak hours). The ESP32 keeps a real wall-clock via NTP, splits each latched period into the right tariff bucket, resets daily totals at midnight, and publishes payloads with ISO timestamps.
**Hardware**: ESP32 + one rbAmp UI1 + WiFi.
**Accounting**: unidirectional. The bucket logic works on STANDARD / PRO — just feed both accumulators through it.

```python
from machine import I2C, Pin
import network, ntptime, time, ujson as json
from umqtt.simple import MQTTClient

i2c = I2C(0, scl=Pin(22), sda=Pin(21), freq=50000)
RB_ADDR = 0x50

WIFI_SSID  = "ssid"
WIFI_PASS  = "password"
MQTT_HOST  = "192.168.1.10"

# Local UTC offset in hours. For DST, recompute on roll-over or use a TZ library.
# Example: Central European Summer Time = UTC+2.
TZ_OFFSET_HOURS = 2

# Time-of-use schedule: peak hours [PEAK_START_HOUR .. PEAK_END_HOUR), local time.
PEAK_START_HOUR = 8
PEAK_END_HOUR   = 22

def wifi_connect():
    wlan = network.WLAN(network.STA_IF); wlan.active(True)
    wlan.connect(WIFI_SSID, WIFI_PASS)
    while not wlan.isconnected():
        time.sleep(0.5)
    print("WiFi up:", wlan.ifconfig()[0])

def ntp_sync():
    """Synchronise the on-board RTC with NTP (UTC).

    Returns True on success, False on persistent failure. DOES NOT raise —
    a TOU meter that crashes when NTP is unreachable is worse than one that
    skips bucketing for the current period and retries later.
    """
    ntptime.host = "pool.ntp.org"
    for _ in range(5):
        try:
            ntptime.settime()
            t = time.gmtime()
            print("NTP sync: {:04d}-{:02d}-{:02d} {:02d}:{:02d}:{:02d} UTC".format(*t[:6]))
            return True
        except Exception as e:
            print("NTP retry:", e)
            time.sleep(2)
    print("WARN: NTP sync failed — tariff bucketing paused until next sync")
    return False

def local_time():
    """Return current local time as a 9-tuple struct_time (year, mon, ..., wday, yday)."""
    return time.gmtime(time.time() + TZ_OFFSET_HOURS * 3600)

def is_peak_hour(hour):
    """True if `hour` (0..23 local) is inside the peak tariff window.

    For schedules that cross midnight (peak 22..6), change to
        return hour >= PEAK_START_HOUR or hour < PEAK_END_HOUR
    """
    return PEAK_START_HOUR <= hour < PEAK_END_HOUR

def iso_now():
    """Return the current local time as an ISO 8601 string with offset."""
    t = local_time()
    sign = "+" if TZ_OFFSET_HOURS >= 0 else "-"
    return "{:04d}-{:02d}-{:02d}T{:02d}:{:02d}:{:02d}{}{:02d}00".format(
        t[0], t[1], t[2], t[3], t[4], t[5], sign, abs(TZ_OFFSET_HOURS))

wifi_connect()
ntp_sync()

mqtt = MQTTClient("rbamp-tou", MQTT_HOST); mqtt.connect()

rb_wait_ready(i2c, RB_ADDR)
rb_write_u8(i2c, RB_ADDR, 0x01, 0x27)            # primer latch
t_prev_ms = time.ticks_ms()
print("TOU started: peak {:02d}:00..{:02d}:00 local".format(PEAK_START_HOUR, PEAK_END_HOUR))

# Tariff buckets.
today = {"peak_wh": 0.0, "off_peak_wh": 0.0, "yday": -1}
lifetime = {"peak_wh": 0.0, "off_peak_wh": 0.0}

def accumulate(e_wh, tm):
    """Add one period's energy to the right bucket; roll over the day at midnight.

    :param e_wh: Wh in this period.
    :param tm:   local struct_time at the latch instant.
    """
    yday = tm[7]                                  # struct_time.tm_yday is index 7
    if today["yday"] != yday:
        if today["yday"] >= 0:
            payload = json.dumps({
                "yday":        today["yday"],
                "peak_wh":     today["peak_wh"],
                "off_peak_wh": today["off_peak_wh"],
                "total_wh":    today["peak_wh"] + today["off_peak_wh"],
            })
            mqtt.publish(b"rbamp/tou/day_close", payload.encode(), retain=True)
            print("Day closed:", payload)
        today["peak_wh"] = 0.0
        today["off_peak_wh"] = 0.0
        today["yday"] = yday

    hour = tm[3]                                  # struct_time.tm_hour is index 3
    if is_peak_hour(hour):
        today["peak_wh"]    += e_wh
        lifetime["peak_wh"] += e_wh
    else:
        today["off_peak_wh"]    += e_wh
        lifetime["off_peak_wh"] += e_wh

while True:
    time.sleep(60)

    rb_write_u8(i2c, RB_ADDR, 0x01, 0x27)
    t_now_ms = time.ticks_ms()
    time.sleep_ms(50)
    if (rb_read_u8(i2c, RB_ADDR, 0x07) & 0x01) == 0:
        t_prev_ms = time.ticks_ms()
        continue
    avg_p = rb_read_float_le(i2c, RB_ADDR, 0xDC)
    dt_s  = time.ticks_diff(t_now_ms, t_prev_ms) / 1000.0
    e_wh  = avg_p * dt_s / 3600.0
    t_prev_ms = t_now_ms

    tm = local_time()
    accumulate(e_wh, tm)
    peak = is_peak_hour(tm[3])

    payload = json.dumps({
        "ts":                iso_now(),
        "tariff":            "peak" if peak else "off_peak",
        "avg_p":             round(avg_p, 1),
        "period_wh":         round(e_wh, 4),
        "today_peak_wh":     round(today["peak_wh"], 3),
        "today_off_peak_wh": round(today["off_peak_wh"], 3),
        "life_peak_wh":      round(lifetime["peak_wh"], 3),
        "life_off_peak_wh":  round(lifetime["off_peak_wh"], 3),
    })
    mqtt.publish(b"rbamp/tou/state", payload.encode())
    print(payload)
```

**Notes**:

- MicroPython's `time` module is UTC-based on most ports — convert with `time.gmtime(time.time() + offset_seconds)`. For full DST handling, a small TZ table or the `utimezone` library can replace `TZ_OFFSET_HOURS`.
- Boundary error: a 60 s period that crosses 22:00 is attributed entirely to off-peak (timestamp at the latch instant). At most ~30 s of misattribution per day — negligible for residential billing.
- For STANDARD / PRO bidirectional metering, call `accumulate()` twice — once on the consumption side and once on the export side — and publish both sets of peak/off-peak totals.

> **CircuitPython delta**: use `adafruit_ntp.NTP(socketpool, server="pool.ntp.org", tz_offset=2)` + `rtc.RTC().datetime = ntp.datetime` to set the system clock, then read it via `time.localtime()`. The bucket logic is identical.

---

## Example comparison

| # | Complexity | Output | MQTT | DRDY | Multi-module | Bidirectional | Persistence | Use case |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|---|
| 1 | minimal | REPL | — | — | — | — | — | smoke test |
| 2 | low | OLED | — | — | — | — | RAM | boxed meter |
| 3 | low | REPL | — | — | yes (3) | — | — | home monitoring |
| 4 | medium | — | yes | — | — | — | RAM | per-appliance in HA |
| 5 | medium | REPL | — | yes | — | yes (master) | RAM | solar home on BASIC tier |
| 6 | high | — | yes | — | yes (3) | yes | RAM | full home balance |
| 7 | medium | REPL + SD | — | yes | — | — | SD card | event detection |
| 8 | medium | — | yes (+disco) | — | — | optional | RAM + MQTT | HA Auto-discovery |
| 9 | medium | — | yes | — | — | — | RTC memory | off-grid / outdoor logger |
| 10 | medium | REPL | yes | — | — | optional | RAM + MQTT | TOU peak/off-peak with NTP |

## Best practices

- Always check `DATA_VALID` (bit 0 of register `0xCE`) after boot and `PERIOD_VALID` (bit 0 of `0x07`) after each `CMD_LATCH_PERIOD`.
- Allow at least 50 ms after a latch before reading the snapshot.
- The **master clock is authoritative** for Wh — do not use `PERIOD_LATCH_MS` (`0xEC`) for energy computation.
- For multi-module setups, `rb_broadcast_latch(i2c, tick)` via general call gives precise synchronisation. GC reception is **opt-in** per module (`FLEET_CONFIG.bit0`, default OFF). Fall back to per-module sequential `CMD_LATCH_PERIOD` on NACK.
- **Master-side persistence is mandatory** for long-term accounting — rbAmp does not store Wh internally. Use RTC memory (Example 9), SD card (Example 7), or MQTT-retained payloads (Example 6).
- MicroPython IRQ handlers must be tiny (set a flag, no I2C). Do all I2C work in the main loop.
- For CircuitPython, prefer `countio.Counter` or `keypad.Keys` for edge events instead of user-defined IRQs.

## What next

After working through these ten examples:

- For the high-level `rbamp` package (MicroPython + CircuitPython), see [18_micropython_library.md](micropython-examples.md).
- For ESPHome integration (declarative YAML), see [`tools/esphome-rbamp/docs/en/`](esphome-overview.md).
- For a richer SBC environment (Raspberry Pi, full CPython), see [20_python_sbc_library.md](python-overview.md).
- Tasmota Berry driver — deferred (no library shipped).
- The formal I2C register specification used here lives in [11_api_reference.md](api-reference.md).
