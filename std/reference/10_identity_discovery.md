# Identity and discovery

How a host recognises an STD module, what it is, and what it can do. All registers in this chapter
are read-only unless stated otherwise. I²C addresses; Modbus discovery mirrors are listed at the end.

## Discovery sequence

On first contact a master reads the identity block at `0x0000` and gates everything else on it:

1. `ID` (0x0000, 2 bytes) must read `'S','T'` (0x53, 0x54) — anything else is not an STD module.
2. `VERSION` (0x0003) is the **register-map layout version** (current: `0x36`). Feature additions
   bump it; a master gates map features on it.
3. `CAPABILITIES` (0x0004, u16) — feature bitmap, see below. Bit 9 (16-bit addressing) is always set
   on STD.
4. `N_VOLTAGE` (0x0007) / `N_CURRENT` (0x0008) — how many channels are populated. **Channel blocks
   at index ≥ N are reserved; their value is unspecified.**
5. `VARIANT` (0x0009) — family member code.
6. `UID` (0x00F4, 12 bytes) — the chip's unique id. Addresses can be changed; the UID cannot: use it
   as the device identity in any database.
7. `FW_VER` (0x0114, u32) — the **running image version**, e.g. `0x00010077`.

## Identity registers

| Addr | Name | Type | Meaning |
|---|---|---|---|
| 0x0000 | ID | 2×u8 | `'S','T'` |
| 0x0002 | ERROR | u8 | result of the last command (see [Commands](14_commands.md)) |
| 0x0003 | VERSION | u8 | register-map version (`0x36`) |
| 0x0004 | CAPABILITIES | u16 | feature bitmap |
| 0x0006 | STATUS | u8 | live status bits (see [Diagnostics](19_diagnostics.md)) |
| 0x0007 | N_VOLTAGE | u8 | voltage channels, 0..8 |
| 0x0008 | N_CURRENT | u8 | current channels, 1..64 |
| 0x0009 | VARIANT | u8 | family member code |
| 0x000A | HW_REV | u8 | hardware revision |
| 0x000B | FW_VERSION | u16 | firmware **line marker** — not the image version |
| 0x00F4 | UID | 12×u8 | unique chip id |
| 0x0114 | FW_VER | u32 | running image version (from the image header) |
| 0x017C | BOARD_ID | u32 | board id of the running image (see below) |

> ⚠ Two "firmware version" registers exist for historical reasons. `FW_VERSION` (0x000B, u16) only
> marks the firmware line. The actual, monotonically increasing image version is **`FW_VER`
> (0x0114, u32)** — use it for inventory and update decisions.

### BOARD_ID (0x017C)

`BOARD_ID` identifies the board the running image was built for:
`("BD" << 16) | channel-count << 8 | VARIANT` = `0x4244nnvv` — e.g. `0x42440211` for STD-UI1.
Every release image carries the same word, covered by the image's
integrity and signature checks; a firmware update with a different board id is refused
(see [Firmware update](18_firmware_update.md)). **Compare `BOARD_ID` with the release file before
sending an update.**

## CAPABILITIES bitmap (0x0004)

| Bit | Name | Meaning |
|---|---|---|
| 0 | SIGNED | signed power/energy (import/export) |
| 1 | REACTIVE | Q and S published |
| 2 | THD | THD / displacement-PF fields live (0 → those fields read 0 on I²C, the sentinel on Modbus native) |
| 3 | HARMONICS | per-harmonic spectra (industrial tier) |
| 4 | INDUSTRIAL | industrial tier build |
| 5 | HUB_IFACE | hub-side interface |
| 6 | OTA | firmware update over the bus |
| 7 | ENERGY | per-window energy accumulators |
| 8 | MULTI_LATCH | snapshot banks present (0 on current firmware — see [Measurement](16_measurement_energy.md)) |
| 9 | ADDR16 | 16-bit register addressing (always 1 on STD) |
| 10 | MULTI_CHANNEL | more than one current channel |
| 11 | MODBUS_IFACE | Modbus RTU transport present |
| 12 | RTC | battery-backed on-device clock (0 on current modules — time comes from the master) |
| 13 | ONDEVICE_ENERGY | cumulative energy counters persisted on the device |
| 14 | CHMAP_PERSIST | per-channel CT-model map persists (reserved; 0 until implemented) |
| 15 | ENERGY_UWH | µWh-resolution energy counters published (image 0x7A+; reactive µvarh from image 0x7B) — see [Measurement & energy](16_measurement_energy.md#cumulative-counters) |

A master treats unknown bits as "feature absent" and never relies on reserved fields.

**Honest-capabilities rule:** a module reports only what the running build actually provides. For
example, current three-phase firmware reports THD = 0,
MULTI_LATCH = 0 (no snapshot banks; energy is read from cumulative counters) and RTC = 0 (the master
feeds the clock).

## VARIANT codes (0x0009)

| Code | Member | Channels | Status |
|---|---|---|---|
| 0x11 | STD-UI1 | 1 V + 1 I | built, validated |
| 0x63 | — | 3 V + 3 I (three-phase) | code reserved |
| 0x13 / 0x15 / 0x17 | — | 1 V + 3/5/7 I | codes reserved |
| 0x42 | — | 2 V + 2 I | code reserved |
| 0x66 | — | 3 V + 6 I | code reserved |

Other codes in the enum space are reserved and must not be assumed absent-forever; a master decides
by `N_VOLTAGE` / `N_CURRENT` + `CAPABILITIES`, not by hardcoding variant codes.

## Modbus discovery mirror

The same identity set is readable over Modbus as input registers at `0x6000`:
MAGIC 0x6000 · MAP_VERSION 0x6001 · FW_VERSION 0x6002 · HW_REV 0x6003 · CAPABILITIES 0x6004 ·
N_VOLTAGE 0x6005 · N_CURRENT 0x6006 · VARIANT 0x6007 · TIER 0x6008 · STATUS 0x6009 · ERROR 0x600A ·
UID 0x600B–0x6010 (12 bytes, big-endian by register) · FRAME_CNT 0x6011–12 · FREQ 0x6013 ·
WINDOW_MS 0x6014 · CMD_SEQ 0x6015 · RING_DROP 0x6016 · WIN_NSAMP 0x6017 · WIN_PERIOD 0x6018 ·
CPU diag 0x6019–0x601C · LOST_FRAMES 0x601D–1E · WIN_SEQ 0x601F. `BOARD_ID` is holding 0x831B–1C;
`FW_VER` is holding 0x8315–16.
