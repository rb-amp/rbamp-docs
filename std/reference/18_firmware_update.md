# Firmware update (host-side protocol)

Firmware updates run **over the working bus** — I²C or Modbus RTU — with no programmer and no
removal. The device keeps two firmware banks; the host writes the new image into the inactive bank,
the device verifies and *arms* it, and a separate activation reboots into it. A bad or crashing new
image **rolls back automatically** to the previous working one.

**Metering continues during the update.** The update yields to measurement (at most one page write
per ~250 ms); the few measurement frames lost to each flash write are gap-compensated in the energy
counters.

## Release files

A release ships **prepared bank images** — signed and encrypted on the build host. The transport
host **streams the file as-is**; it never builds, edits or re-wraps an image:

- Two files per release, one per bank (A / B). Read `FW_ACTIVE_BANK` (0x0156): 0 = A running →
  send the **B** file, and vice versa.
- Compare the release's board id with `BOARD_ID` (0x017C) first — an image built for another board
  is refused at verify (reason 15).
- A **strictly lower** version than the running one is refused (anti-downgrade, status 4); the same
  version is accepted. Returning to an older version is possible only through the device's own
  rollback path, not by pushing an old image.

---

## Registers

I²C (Modbus holding in brackets; u32 = 2 registers, high word first):

| Reg | Addr | Meaning |
|---|---|---|
| OTA_STATUS | 0x0140 (0x8300) | 0 idle · 1 ready/session open · 2 armed · 3 error · 4 downgrade refused |
| OTA_IDX | 0x0144 (0x8302–03, rw) | 256-B page index for the next page write |
| OTA_ACTIVE | 0x0148 (0x8306–07) | running bank base (opaque) |
| OTA_BANK_SIZE | 0x014C (0x8304–05) | bank size — device-reported; header page index = size/256 − 1 |
| FW_STAGED_VERSION | 0x0150 (0x8308–09) | version of the armed image (0 = none) |
| FW_ACTIVE_BANK | 0x0156 (0x8301) | 0 = A · 1 = B |
| OTA_PCRC_EXP | 0x0170 (0x8310–11, w) | optional expected CRC-32 of the next page |
| OTA_PCRC_RX | 0x0174 (0x8312–13) | CRC-32 of the page as received |
| OTA_KEYID | 0x0178 (0x8319–1A) | image key id this firmware accepts |
| OTA_DIAG | 0x0180 (0x8314: reason \| status<<8) | failure diagnostics |
| OTA_PAYLOAD | 0x0500, 256 B (0x8380–0x83FF byte pairs) | page buffer |
| FW_VER / SWAP_PENDING | 0x0114 / 0x011C (0x8315–16 / 0x8317–18) | running version / swap state 0 confirmed · 1 armed · 2 trial |

Commands (COMMAND 0x0060 / holding 0x8113): `0x10` BEGIN · `0x11` BLOCK_WR · `0x12` COMMIT ·
`0x13` ACTIVATE · `0x14` FW_ABORT. Over Modbus, BEGIN/COMMIT/ACTIVATE/FW_ABORT are UNLOCK-gated;
BLOCK_WR is not. After every command poll `CMD_STATE` (0x006A / 0x8116) until BUSY = 0, then read
`OTA_STATUS` / `ERROR`.

---

## Sequence

1. **Select.** Read `FW_ACTIVE_BANK` → pick the release file for the *other* bank. Check `BOARD_ID`
   against the file; optionally check `OTA_KEYID`.
2. **BEGIN (0x10).** Opens the session; `OTA_STATUS` = 1 immediately. Nothing is erased up front —
   flash sectors are erased as their first page arrives, so the bus and metering keep running.
   A BEGIN while an image is armed disarms it first.
3. **BLOCK_WR (0x11) per page, ascending.** For each non-blank 256-B page k of the file:
   `OTA_IDX` = k → 256 B into `OTA_PAYLOAD` → *(optional)* `OTA_PCRC_EXP` = CRC-32 of the page →
   command 0x11 → poll BUSY to 0 (allow up to ~0.5 s).
   - Send pages **in ascending order**; **page 0 must be the first page of the session** (otherwise
     COMMIT refuses with reason 13); **skip pages that are all 0xFF**. The header page
     (`OTA_BANK_SIZE`/256 − 1) does not have to be last — ascending order sends it last naturally.
   - **After every page check `OTA_STATUS` is still 1.** `ERROR` 0xFC = payload arrived corrupted →
     rewrite the payload and repeat the page (an identical resend of an already-written page is
     accepted). `OTA_STATUS` 3 = the session failed (read `OTA_DIAG`; *different* data for an
     already-written page is one cause) — later page writes are ignored; start over with BEGIN.
   - Throttle: the device programs at most one page per ~250 ms (the command simply stays BUSY until
     its slot) — metering has priority. A full image takes ~1–3 minutes.
   - On I²C expect clock stretching up to ~9 ms and tolerate NACKs during the write window
     ([I²C transport](11_transport_i2c.md#timing-and-robustness)). Modbus is unaffected.
4. **COMMIT (0x12).** Poll BUSY with a timeout of **≥ 3 s**: the device erases any sectors no page
   touched, then verifies the image — integrity, authenticity, version, board id. Result:
   - `OTA_STATUS` **2** — verified and **armed**; `FW_STAGED_VERSION` = new version. The module
     still runs the old image. A power loss after COMMIT boots the armed image (safe — it is the
     intended one).
   - **3** — refused; `OTA_DIAG` says why (below).
   - **4** — version older than running; nothing changed.
5. **ACTIVATE (0x13)** when the installation is ready for a restart — **check `OTA_STATUS` = 2
   first** (not armed → the command does nothing, `ERROR` stays 0). Energy is flushed, the module
   resets and re-appears in ~4 s (over Modbus the reply to ACTIVATE may be absent or truncated).
   The new image boots **as a trial** under a hardware watchdog and confirms itself after ~8 s of
   healthy metering. **Any reset before it confirms rolls back** — do not reset the module for
   ~10 s after activation.
   Verify afterwards: `FW_ACTIVE_BANK` flipped, `FW_VER` = new version, `SWAP_PENDING` 2 → 0; the
   old `FW_VER` still running = it rolled back.
   **FW_ABORT (0x14)** instead of ACTIVATE drops the armed image and closes the session.

---

## Failure diagnostics (OTA_DIAG 0x0180)

Byte 0 = reason · byte 1 = internal status · then offset, page/length and check values.

| Reason | Meaning | Reason | Meaning |
|---|---|---|---|
| 1 | page index out of range | 9 | image integrity check failed |
| 2/3 | erase failed / erase verify failed | 10 | authentication failed |
| 4 | different data for an already-written page (session failed — restart from BEGIN) | 11 | version downgrade |
| 5/6 | program failed / verify failed | 12 | swap record error |
| 7 | bad image header | 13 | image not sealed / incomplete (e.g. page 0 or the header page never arrived) |
| 8 | header page never arrived | 14 | signature invalid |
| | | 15 | **image built for another board** (check value = the image's board word) |

All authentication failures report as `OTA_STATUS` 3 with the reason here — there are no separate
status values to handle. Over Modbus only the reason and internal status are mirrored (holding
0x8314); the offset/page/check details of `OTA_DIAG` are readable on I²C.

---

## Safety properties (what the host may rely on)

- Two-phase: **COMMIT never reboots**; the restart moment is the host's (maintenance window).
- Power loss at any point is safe: before COMMIT the old image runs; after COMMIT the armed image is
  the intended one; a failed trial rolls back.
- Calibration, configuration, energy counters, history, the event log and the device identity are
  stored outside the firmware banks — an update never touches them.
- A firmware commit is event-logged with the version transition.
