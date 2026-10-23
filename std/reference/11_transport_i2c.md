# I²C transport

The module is an I²C slave with the full register map. Default 7-bit address **0x50**
(configurable, see below). 100 kHz is safe on any build; the slave supports clock stretching and a
master must tolerate it (see limits below).

The active transport (I²C or Modbus RTU) is selected once per boot by the transport jumper — see
Hardware interface. On the I²C setting the Modbus register space is not served,
and vice versa; commands and data are the same on both.

---

## Register pointer

- **16-bit register pointer, big-endian on the wire** (high byte first), with **auto-increment**.
- **Write:** master sends 2 pointer bytes (hi, lo), then a data stream; each data byte lands at the
  pointer, which post-increments.
- **Read:** master writes the 2 pointer bytes, then issues a **repeated start** and reads; bytes
  stream from the pointer with auto-increment.
- **Multi-byte fields are little-endian** and byte-addressed — no alignment requirement on the bus;
  a master may read any field as N little-endian bytes from its address, in one transaction or
  several.
- **Unmapped addresses:** reads return `0xFF`, writes are ignored. Channel-block addresses at index
  ≥ `N_CURRENT` / `N_VOLTAGE` are unspecified — gate on the counts, do not interpret them.

There is no bus-level probe for 8- vs 16-bit addressing (an 8-bit device would misread the second
pointer byte as data). A master knows the convention by product line; the STD signature after a
16-bit read of `0x0000` is `'S','T'` + `VERSION ≥ 0x30` + CAPABILITIES bit 9.

---

## Timing and robustness

- **Readiness after reset:** the register interface comes up within milliseconds of a reset, before
  the measurement path — the module answers on the bus even while the sensor path is still starting.
  After a firmware activation reboot allow **~4 s** before the device re-appears.
- **Clock stretching:** normally negligible; during firmware-update flash operations the slave may
  hold SCL for up to **≈9 ms**. Configure the master's SCL timeout to **≥10 ms** or retries will be
  logged during OTA (data is not corrupted — the transaction simply waits).
- **During flash writes** triggered by commands (saving configuration/calibration, OTA page
  programming) the slave may also not-acknowledge briefly. Retry until the command's settle time
  elapses rather than aborting on the first NACK (see [Commands](14_commands.md) for settle times).
- **Coherency:** every read transaction is served from a coherent **128-byte snapshot** taken when
  the register address is set — fields read in **one transaction of up to 128 bytes** are from one
  publication and cannot tear (this is how to read i64 energy fields and whole blocks). Fields read
  in *separate* transactions may come from different windows. To detect a module restart between
  reads, check `FRAME_CNT` (0x0020) monotonicity and `BOOT_COUNT` (0x00CE).

---

## Changing the I²C address

Two paths:

**Direct (address known):**
1. Write the new 7-bit address (0x08..0x77) to `CFG_I2C_ADDR` (0x0073).
2. `COMMAND` = 0x04 (SAVE_CONFIG) — validated; an illegal address is refused with `ERROR` 0xFD and
   nothing is persisted. Saves are rate-limited: a repeat within ~5 s answers 0xF9 — retry later.
   Wait for BUSY = 0 and `ERROR` = 0 **before** the next step.
3. `COMMAND` = 0x02 (RESET). The address applies on the reboot; the device re-enumerates at the new
   address in ~1 s.

> ⚠ After the reset **every** master on the bus must switch to the new address; plan this as an
> installation operation, not a routine one.

**By UID (several modules on one default address):**
Write the target module's 12-byte UID to `ADDR_UID` (0x00E0), the new I²C address to `ADDR_UID_I2C`
(0x00EC, 0 = keep) and/or the new Modbus unit id to `ADDR_UID_UNIT` (0x00ED, 0 = keep), then
`COMMAND` = 0x16 (ADDR_BY_UID). Only the module whose UID matches persists the change — **together
with any other staged CFG_ values, like a SAVE_CONFIG** — and restarts; every other module ignores
the command (`ERROR` 0xFC there). Also accepted over a Modbus broadcast — see
[Modbus transport](12_transport_modbus.md). Both new-address fields zero → 0xFD; illegal address →
0xFD; save rate limit → 0xF9. Step-by-step: [Sequences D4](22_sequences.md#d4--address--serial-change).
