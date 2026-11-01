# Data-ready output and events

Hosts that poll on a timer waste bus transactions and still lag the data. The STD module offers a
hardware **DRDY (data-ready) line** plus a register-side event model, shared across the rbAmp
family.

## DRDY line contract

- **Electrical:** open-drain output without a pull-up on the module — when DRDY is used, fit a pull-up on the master side (10 kΩ to the master's 3.3 V logic supply is recommended); idle **HIGH**.
- **Pulse:** **~10 µs LOW pulse** for every **new measurement window** — the module publishes a complete snapshot
  **once per finalized measurement window**: whole mains cycles nearest 100 ms (5 at 50 Hz, 6 at
  60 Hz), i.e. about **10 pulses a second**. Without a new window the module still publishes on a slower fixed fallback
  tick (500 ms) that refreshes status and clock; that fallback publish steps `WIN_SEQ` but
  gives **no DRDY edge**. The pulse fires *after* the registers are updated —
  a read triggered by the falling edge always sees the fresh snapshot.
- **`WIN_SEQ` (0x0016, u16)** increments by exactly 1 in the same instant as the pulse (and also on a fallback publish, which has no pulse). Pairing rule:
  - with the DRDY line: on each serviced edge check WIN_SEQ stepped by 1 — a larger step means
    missed edges;
  - without the line: poll WIN_SEQ; a change means a fresh snapshot.
- **Alarm mode:** any event bit that is set in both `EVENT_FLAGS` and `EVENT_MASK` holds DRDY
  **solid LOW** until the flag is cleared or unmasked. The alarm asserts at the next publish
  tick and releases immediately on write-1-to-clear or unmask (NVM_FAULT re-asserts every publish
  while the fault persists). A host distinguishes pulse from alarm by level: still LOW after
  ~100 µs = alarm → read EVENT_FLAGS.

---

## EVENT_FLAGS (0x00C0) / EVENT_MASK (0x00C1)

`EVENT_FLAGS` bits are **sticky** and cleared by writing 1 to the bit (write-1-to-clear).
`EVENT_MASK` selects which of them assert the DRDY alarm. Both live in RAM — **defaults restore at
every module reset**, so a host re-arms its mask after any reboot it detects.

| Bit | Name | Set when |
|---|---|---|
| 0 | PERIOD_READY | a 15-minute summary was finalized |
| 1 | THRESH_I | any current channel exceeded `THRESH_I` |
| 2 | THRESH_P | \|total active power\| exceeded `THRESH_P` |
| 3 | ERROR | a command finished with a non-OK result |
| 4 | CONFIG_CHANGED | SAVE_CONFIG / SAVE_CAL succeeded |
| 5 | RESET_OCCURRED | set at every boot (the only flag set at boot) — a host that sees it knows the module restarted |
| 6 | NVM_FAULT | the energy store went FAULT/CORRUPT |

At boot `EVENT_FLAGS` = 0x20 (RESET_OCCURRED only) and both thresholds re-initialize to
`0xFFFFFFFF` (disabled).

Thresholds: `THRESH_I` (0x00C4, µA) triggers when **any** current channel goes above it;
`THRESH_P` (0x00C8, mW) compares the magnitude of the grand active power (direction-agnostic).
`0xFFFFFFFF` disables (default). Both RAM.

---

## Recommended host patterns

**Edge-driven realtime** (lowest latency, minimum bus traffic):
1. Configure a falling-edge interrupt on DRDY.
2. On the edge: read WIN_SEQ + the needed measurement fields in one transaction.
3. If WIN_SEQ stepped by more than 1 — you missed windows (not an error; data is current).

**Alarm wake-up** (host sleeps, module watches):
1. Write THRESH_I / THRESH_P, set the wanted bits in EVENT_MASK.
2. Sleep until DRDY goes (and stays) LOW.
3. Read EVENT_FLAGS, handle, write-1-to-clear. DRDY releases when no masked flag remains.
4. After any detected module reset (RESET_OCCURRED, BOOT_COUNT step, WIN_SEQ discontinuity):
   re-write thresholds and mask — they are RAM.

**Pure polling** (no spare GPIO): poll WIN_SEQ at ≤ the window rate; read data when it changes; poll
EVENT_FLAGS at whatever alarm latency is acceptable.

Modbus equivalents: EVENT_FLAGS holding 0x8120 (write-1-to-clear), EVENT_MASK 0x8121,
THRESH_I 0x8122–23, THRESH_P 0x8124–25. The DRDY line itself is transport-independent.
