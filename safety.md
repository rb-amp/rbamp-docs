# Safety — Safe by Design

rbAmp is built so that mains never reaches the board. The module itself is **entirely low-voltage** —
there is no high-voltage measurement circuit on it. Current is read by a **CT clamp that never touches
the wire**, and the only mains reference at all (on voltage-sensing variants) is a **galvanically
isolated** tap. Everything *you* wire — the I²C bus to an ESP32, Arduino or Raspberry Pi — is
**never at mains potential.**

This page collects, in one place, the isolation architecture, what you may safely touch, what belongs
to a qualified electrician, and how an isolated design differs from the shunt-based meters common in
DIY energy monitoring.


## The architecture: an all-low-voltage module

The module itself carries **no high-voltage measurement circuit.** Both ways it senses power keep
mains away from the board you touch:

- **Current — a contactless CT.** The clamp closes around the *outside* of an insulated conductor.
  A current transformer couples **magnetically** — there is **no electrical contact** with the mains
  wire, and nothing at line potential ever enters the module. The CT hands the board only a small,
  isolated signal.
- **Voltage (UI\* variants) — an isolated tap.** The one place mains is referenced at all is the
  `L`–`N` terminal on voltage-sensing variants, and it feeds a **galvanically isolated** sensing
  divider behind an isolation barrier. Ampere-only variants have **no mains connection whatsoever.**
- **The side you wire — pure LV.** Five pins: `VCC` (5 V), `GND`, `SDA`, `SCL`, `DRDY`. Logic runs
  at **3.3 V** (an on-board low-noise regulator derives it), the lines are **5 V-tolerant**, and the
  module's **`GND` is your host's ground, not mains neutral.**

So there is no user-accessible high voltage anywhere: the current sensor never contacts the wire, the
voltage reference is isolated, and the bus rides your own ground. Wiring the module straight to your
ESP32/Arduino/Pi carries **no shock risk and no short-circuit-through-ground risk.**

> **Never open the enclosure** — it voids the factory calibration.

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

rbAmp measures the opposite way. Its current path is a **CT that clamps around the conductor and never
touches it** — no galvanic connection to the mains at all, so nothing on the module sits at line
potential from the current side. On voltage-sensing variants the single `L`–`N` reference goes through
an **isolated** divider. There is **no shunt in the mains path and no live logic board**: the side you
wire, handle and debug is low-voltage by construction. That's what "safe by design" means — not a
sticker, an **architecture.**

*(rbAmp is designed for isolation as described here; it is not a substitute for following your local
electrical code, and it is not sold as a certified metering instrument.)*

## In short

- The module is **all low-voltage** — the CT clamps the wire without touching it, and voltage (UI* variants) comes in through an **isolated** tap; the side you touch is never at mains potential.
- Current is sensed by a **CT** (no contact); voltage (UI* variants) via an **isolated divider** — bring L/N in with a **plug into a socket**, or an electrician.
- **Don't open the enclosure.** For panel work, kill the main, verify dead, and use a qualified electrician.
- Isolation is in the design — unlike a bare shunt meter, the side you handle is never live.

## Related

- [01 · Hardware Connection](/docs/modules-basic-standard-hardware-connection) — pinout, wiring, CT install.
- [FAQ — Is the rbAmp module isolated from mains?](/blog/faq-13/is-the-rbamp-module-isolated-from-mains-57)
- [Three-Phase Energy Monitoring in Home Assistant](/blog/projects-and-tutorials-11/three-phase-energy-monitoring-in-home-assistant-3-rbamp-ui-on-one-esp32-49) — panel-level safety in a real build.

