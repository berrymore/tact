# 02 — Memory Map and Storage Layouts

Status: **Draft** · Updated: 2026-11-20 · Depends on P-001 (two-stage boot) and P-002 (address
subset); see [decision-log.md](decision-log.md).

All multi-byte values are **big-endian** (68000 native).

## 1. CPU address space (24-bit, 16 MB)

The chipset decodes `A23–A20` (sixteen 1 MB regions) plus `A1–A9` inside the I/O region. `FC0–FC2`
qualify every access.

| Range | Size | Region | Notes |
|---|---|---|---|
| `$000000–$0FFFFF` | 1 MB | **SRAM0** | While `SYS_CFG.OVL = 1`, **reads** anywhere in this region return boot-ROM contents (mirrored); writes still go to SRAM0. |
| `$100000–$1FFFFF` | 1 MB | **SRAM1** (optional) | Decoded only when `SYS_CFG.SRAM1_EN = 1`; otherwise `/BERR`. |
| `$200000–$EFFFFF` | 13 MB | unmapped | `/BERR`. |
| `$F00000–$FFFFFF` | 1 MB | **I/O window** | Only `A1–A9` are decoded, so a 1 KB I/O page repeats throughout the region. Software must use the canonical addresses below. |

CPU-space cycles (`FC = 111`, i.e. interrupt acknowledge) are never treated as memory accesses: the
chipset answers them with an autovector.

### Access protection

- The I/O window is accessible only in supervisor mode (`FC = 101` data, `110` program). A user-mode
  access raises `/BERR` while `SYS_CFG.UPROT = 1` (reset default).
- Any access that is not acknowledged within the bus timeout (`SYS_BUSTO`, default 64 CPU clocks)
  raises `/BERR`. The bus tracer freezes on `/BERR` when armed to do so.

### Wait states

| Target | Wait states |
|---|---|
| SRAM0 / SRAM1 | 0 (target; confirmed in the bus-timing doc) |
| Chipset registers, boot ROM | ≥ 1 |
| FIFO ports when full / empty | `/DTACK` is held off until the FIFO can accept / supply data, bounded by the bus timeout |

## 2. I/O page (canonical addresses)

| Range | Block | Range | Block |
|---|---|---|---|
| `$F00000–$F0001F` | `SYS` | `$F00100–$F0011F` | `I2CM` |
| `$F00020–$F0003F` | `INTC` | `$F00120–$F0013F` | `INPUT` |
| `$F00040–$F0005F` | `TIMER` | `$F00140–$F0015F` | `PWR` |
| `$F00060–$F0007F` | `UART` | `$F00160–$F0017F` | `AUX` (piezo, backlight PWM) |
| `$F00080–$F0009F` | `SDC` | `$F00180–$F0019F` | `FLASH` |
| `$F000A0–$F000BF` | `STREAM` | `$F001A0–$F001BF` | `DBG` |
| `$F000C0–$F000DF` | `VSC` | `$F001C0–$F001FF` | reserved (`/BERR`) |
| `$F000E0–$F000FF` | `LCDC` | `$F00200–$F003FF` | **Boot ROM** (512 B, read-only) |

Each block has 16 word registers. Register layouts are in `03-chipset-registers.md`.

**Register access rules**

- Registers are 16 bits wide. Word access is the norm. A byte access to an even address touches bits
  15–8; one to an odd address touches bits 7–0.
- 32-bit quantities occupy a `_HI` / `_LO` register pair, so one `MOVE.L` reads or writes both. Reading
  `_HI` latches `_LO` so that counters read consistently.
- Reserved bits read as 0 and must be written as 0.

## 3. Exception vectors

The firmware owns a standard 68000 vector table at `$000000` in SRAM0 (after `OVL` is cleared).
Vectors used by the system:

| Vector | Offset | Use |
|---|---|---|
| 0, 1 | `$000`, `$004` | Reset SSP / PC (served by the boot ROM through the overlay at reset) |
| 2, 3 | `$008`, `$00C` | Bus error, address error → crash handler (logs to UART / screen) |
| 4–11 | `$010–$02C` | Illegal, divide-by-zero, CHK, TRAPV, privilege, trace, line-A, line-F |
| 24 | `$060` | Spurious interrupt |
| 25–31 | `$064–$07C` | Autovectors, levels 1–7 (sources in [01-architecture.md §7](01-architecture.md#7-interrupts)) |
| 32–47 | `$080–$0BC` | `TRAP #0–15`: reserved for a future system-call interface |

## 4. Software RAM map (SRAM0)

| Range | Size | Use |
|---|---|---|
| `$000000–$0003FF` | 1 KB | Vector table |
| `$000400–$0DFFFF` | ≈ 895 KB | Firmware image (`.text`, `.data`, `.bss`), then heap (library cache, framebuffer, file buffers) |
| `$0E0000–$0F7FFF` | 96 KB | tactboot while booting; free for the firmware afterwards |
| `$0F8000–$0FFFFF` | 32 KB | Supervisor stack (initial SSP = `$100000`) |

SRAM1, when fitted, is extra heap (for example a larger library cache).

## 5. Boot ROM (stage 1)

- 512 bytes at `$F00200`; the first two longwords are the reset vectors:
  - `$F00200`: initial SSP = `$00100000`
  - `$F00204`: initial PC = an entry point inside `$F00208–$F003FF`
- At reset the CPU reads these vectors from `$000000` / `$000004` through the overlay; execution then
  continues at the ROM's canonical address, and the ROM clears `OVL` before touching RAM.
- Job: copy tactboot from flash `$100000` to SRAM `$0E0000`, check its header and CRC-32 (the chipset
  computes the CRC on the fly while the `FLASH` read port streams data), jump to it. On failure: UART
  loader + LED blink code.

## 6. SPI flash layout (W25Q128JV, 16 MB)

| Range | Size | Contents |
|---|---|---|
| `$000000–$0FFFFF` | 1 MB | FPGA configuration area. Today one bitstream at `$000000`; reserved for a later multi-image (warm-boot) layout with a golden fallback. |
| `$100000–$13FFFF` | 256 KB | tactboot image (tact image header + payload) |
| `$140000–$17FFFF` | 256 KB | tactboot slot B (reserved for safe in-field updates) |
| `$180000–$FFEFFF` | ≈ 14.5 MB | Reserved (fonts, logs, …) |
| `$FFF000–$FFFFFF` | 4 KB | Board info: board revision, serial number, calibration constants |

The FT2232HL (Rev A) programs this flash while it holds the FPGA in reset (`CRESET_B` low). After
configuration the FPGA owns the flash through its `FLASH` block.

## 7. tact image format

Used for tactboot (in flash) and `FIRMWARE.BIN` (on the card). A 32-byte header is followed by the
payload.

| Offset | Size | Field |
|---|---|---|
| 0 | 4 | Magic `"TACT"` |
| 4 | 2 | Header version (= 1) |
| 6 | 2 | Image type: 1 = tactboot, 2 = firmware |
| 8 | 4 | Load address of the payload |
| 12 | 4 | Entry point |
| 16 | 4 | Payload size in bytes |
| 20 | 4 | Payload CRC-32 (IEEE 802.3 / zlib polynomial) |
| 24 | 4 | Version: major (1 B), minor (1 B), patch (2 B) |
| 28 | 4 | CRC-32 of header bytes 0–27 |

## 8. SD card layout

| Path | Contents |
|---|---|
| `/TACT/FIRMWARE.BIN` | Main firmware (tact image, type 2) |
| `/TACT/LIBRARY.DB` | Library index (format in its own doc) |
| `/TACT/STATE.DAT` | Settings and resume state; written only when not streaming (FR-ST-5) |
| `/TACT/LOG.TXT` | Optional crash / debug log |
| anything else | The user's music and playlists |

The card may be FAT32 or exFAT (P-006).
