# Safety — Safe by Design

rbAmp is built so that the dangerous part is never in your hands. The mains side — the high-voltage
connections, the voltage divider, the current-sensor front end — is **sealed and galvanically
isolated inside the module.** Everything *you* wire — the I²C bus to an ESP32, Arduino or Raspberry
Pi — sits on the other side of an isolation barrier and is **never at mains potential.**

This page collects, in one place, the isolation architecture, what you may safely touch, what belongs
to a qualified electrician, and how an isolated design differs from the shunt-based meters common in
DIY energy monitoring.

![HV/LV isolation architecture — the galvanic barrier separating the mains side (voltage divider + CT input) from the low-voltage I²C side (VCC/GND/SDA/SCL/DRDY); the module's GND is the host's ground, not mains neutral.](/web/content/PLACEHOLDER/hw-isolation-architecture.svg)

## The isolation architecture (HV ⟂ LV)

Inside the enclosure there are two electrically separate worlds, joined only by an isolation barrier
that passes the *measurement*, not the *voltage*:

- **High-voltage (HV) side — sealed inside.** The mains L/N tap feeds an isolated voltage divider;
  the current sensor connects through the CT input. This side can be at line potential. You never
  open it, and you never touch it.
- **Low-voltage (LV) side — what you connect.** Five pins: `VCC` (5 V), `GND`, `SDA`, `SCL`,
  `DRDY`. Logic runs at **3.3 V** (an on-board low-noise regulator derives it), the lines are
  **5 V-tolerant**, and — critically — the module's **`GND` is your host's ground, not mains
  neutral.** There is no galvanic path from the bus to the mains.

Because the barrier sits between them, wiring the LV side straight to your ESP32/Arduino/Pi carries
**no shock risk and no short-circuit-through-ground risk.**

> **Never open the enclosure.** The mains side lives in there, and opening it also voids the factory
> calibration.

## What *you* touch — always the safe side

- The **four I²C wires** (`VCC`, `GND`, `SDA`, `SCL`) and the optional **`DRDY`** line — isolated, low-voltage, safe.
- The **CT clamp**, which closes around the *outside* of an **insulated** conductor. A current
  transformer couples magnetically — there is **no electrical contact** with the wire it measures.
  rbAmp's SCT-013-style clamps carry an internal burden, so there's no open-secondary hazard.

That's the whole of the user's job on a current-only build: bus wires to your controller, clamp on an
insulated conductor.

## The voltage connection (UI* voltage-sensing variants)

Modules that measure **voltage** (the UI* variants — real active power, not just current) need a
reference to the line: **connect `L` and `N` to the module's `L`–`N` terminal.** The sensing side is
**galvanically isolated** inside the module — feeding L/N to the terminal keeps your I²C side just as
safe as before. There are two ways to bring L/N in:

1. **The simple, user-safe way — a plug into a socket.** Use a ready-made L–N pigtail (a short cable
   with a proper mains **plug**) from the module's `L`–`N` terminal, and plug it into a wall socket —
   exactly like plugging in any appliance. You never handle bare conductors; the plug and socket do
   the mains connection for you. This is the recommended path for a desk or single-outlet setup.
2. **The permanent way — an electrician.** For a panel install or a hardwired feed, a **qualified
   electrician** connects L/N at the panel to local electrical code.

> ⚠️ The isolation protects the *measurement* side — but the L/N feed itself is still mains. Use a
> proper plug, or a qualified electrician. **Never connect bare mains conductors yourself unless you
> are qualified to do so.**

## Who does what

| Task | You | Electrician |
|---|---|---|
| I²C bus + DRDY to your controller | ✅ | |
| Clamp the CT on an **insulated** conductor | ✅ | |
| Plug-in L–N voltage pigtail into a socket | ✅ | |
| Hardwired L/N at the panel / consumer unit | | ✅ |
| CTs around panel busbars / inside a live panel | | ✅ |
| Any work on bare or exposed mains conductors | | ✅ |

## Isolated by design vs a non-isolated shunt

Much of DIY energy monitoring — and many single-chip meter ICs — measure current through a **shunt
resistor placed in series with the load.** That approach puts the measurement circuit (and, on many
boards, the whole logic side) **at mains potential, with no galvanic isolation.** The consequences:

- The board is **not safe to touch while live**, and a stray contact between it and your grounded
  controller can be hazardous.
- Getting the readings to an MCU or Raspberry Pi safely means **adding isolation yourself** —
  opto-isolators, an isolated supply, careful enclosure — or accepting the risk.

rbAmp does the opposite: **the isolation is built in, on the HV side, before the signal ever reaches
you.** The current path is a CT (isolated by physics); the voltage path is an isolated divider behind
the barrier. The side you wire, handle and debug is never live. That's what "safe by design" means —
not a sticker, an **architecture.**

*(rbAmp is designed for isolation as described here; it is not a substitute for following your local
electrical code, and it is not sold as a certified metering instrument.)*

## In short

- The mains side is **sealed and isolated inside**; you only ever touch the low-voltage I²C side.
- Current is sensed by a **CT** (no contact); voltage (UI* variants) via an **isolated divider** — bring L/N in with a **plug into a socket**, or an electrician.
- **Don't open the enclosure.** For panel work, kill the main, verify dead, and use a qualified electrician.
- Isolation is in the design — unlike a bare shunt meter, the side you handle is never live.

## Related

- [01 · Hardware Connection](/docs/modules-basic-standard-hardware-connection) — pinout, wiring, CT install.
- [FAQ — Is the rbAmp module isolated from mains?](/blog/faq-13/is-the-rbamp-module-isolated-from-mains-57)
- [Three-Phase Energy Monitoring in Home Assistant](/blog/projects-and-tutorials-11/three-phase-energy-monitoring-in-home-assistant-3-rbamp-ui-on-one-esp32-49) — panel-level safety in a real build.

