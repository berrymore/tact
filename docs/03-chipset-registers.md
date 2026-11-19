# 03 — Chipset Register Specification

Status: **Draft** · Updated: 2026-11-20

This document is the single source of truth for the chipset's programming model. The RTL (`rtl/`),
the emulator's C models (`emu/`) and the firmware headers (`fw/`) all implement exactly this; the
conformance tests check it (NFR-7). Change this document first, then the code.

## 1. Conventions

- Block base addresses are listed in [02-memory-map.md §2](02-memory-map.md#2-io-page-canonical-addresses).
  A register is named `BLOCK_NAME` and addressed as base + offset.
- All registers are 16 bits wide. A word access is the norm; byte-access rules are in the memory map.
- Access types:

  | Type | Meaning |
  |---|---|
  | `RW` | Read / write |
  | `RO` | Read-only; writes are ignored |
  | `WO` | Write-only; reads return 0 |
  | `W1C` | Read status; writing 1 to a bit clears it |
  | `PORT` | FIFO or auto-incrementing data port; an access has side effects |

- **Interrupts.** Every block with interrupts has a `_STAT` register (event bits, `W1C`) and an `_IE`
  register with the same bit layout. The block's interrupt request is `|(STAT & IE)`; `INTC` maps it
  to a fixed level.
- **Keys.** Registers that can reset or power off the system need a key value in the upper byte, so
  that a stray write cannot trigger them.
- **Stalls.** A `PORT` that is momentarily full or empty holds off `/DTACK` until it can complete,
  bounded by `SYS_BUSTO`. Ports are designed so that a legitimate stall is a few µs at most.
- Unlisted offsets in a block read as 0 and ignore writes.
- "SYSCLK" is 32 MHz (P-004).

## 2. `SYS` — $F00000

| Off | Name | Type | Reset | Description |
|---|---|---|---|---|
| $00 | `SYS_ID` | RO | `$7AC7` | Chipset identification constant. |
| $02 | `SYS_VER` | RO | — | `[15:8]` major, `[7:0]` minor version of the RTL. |
| $04 | `SYS_CFG` | RW | `$0005` | `[0] OVL` boot overlay (reads of `$0xxxxx` come from the boot ROM). `[1] SRAM1_EN`. `[2] UPROT` user-mode access to the I/O window → `/BERR`. |
| $06 | `SYS_CLKDIV` | RW | `$0000` | `[2:0]` CPU clock = SYSCLK / 2^(n+1): 0 = 16 MHz, 1 = 8, 2 = 4, 3 = 2, 4 = 1 MHz. The switch is glitch-free and takes effect at the next CPU clock edge. Values 5–7 are reserved. |
| $08 | `SYS_BUSTO` | RW | `$0040` | `[7:0]` bus timeout in CPU clocks; 0 disables it. If no device acknowledges in time → `/BERR`. |
| $0A | `SYS_RSTCAUSE` | W1C | — | `[0] POR` `[1] WATCHDOG` `[2] SOFT` `[3] DEBUG` (reset requested over the debug UART). |
| $0C | `SYS_RESET` | WO | — | Key `$5A` in `[15:8]`. `[0]` resets the CPU and chipset (FPGA stays configured). `[1]` reconfigures the FPGA from flash. |
| $0E | `SYS_SCRATCH` | RW | `$0000` (POR only) | Survives soft and watchdog resets; used to pass a boot request (e.g. "enter Disk Mode") across a reset. |
| $10 | `CRC_CTRL` | RW | `$0000` | `[0] INIT` (WO pulse: running CRC := `$FFFFFFFF`). `[1] FLASH_FEED`: bytes read through `FL_READ` also feed the CRC. |
| $12 | `CRC_DATA` | PORT (WO) | — | Feeds the CRC: a word write feeds two bytes (high first); a byte write to the odd address feeds one byte. |
| $14 | `CRC_HI` | RO | — | Final CRC-32 `[31:16]` (bit-inverted running value, i.e. matches zlib `crc32()`). Reading `CRC_HI` latches `CRC_LO`. |
| $16 | `CRC_LO` | RO | — | Final CRC-32 `[15:0]`. |

CRC-32 polynomial: IEEE 802.3 (`$04C11DB7`, reflected), as used by zlib and the tact image format.

## 3. `INTC` — $F00020

Interrupt sources and their fixed levels:

| Bit | Source | Level |
|---|---|---|
| 0 | `STREAM` | 6 |
| 1 | `SDC` | 5 |
| 2 | `TIMER` (tick) | 4 |
| 3 | `UART` | 3 |
| 4 | `INPUT` | 2 |
| 5 | `I2CM` (transfer done, or the shared external interrupt line: wheel, GPIO expander — VBUS / charger / Hold, fuel gauge) | 2 |
| 6 | Watchdog pre-timeout (`TIMER`) | 7 |
| 7 | Debug break (`UART` BREAK, when enabled) | 7 |
| 8–15 | reserved | — |

| Off | Name | Type | Reset | Description |
|---|---|---|---|---|
| $00 | `INT_PEND` | RO | — | Interrupt request of each source (level-sensitive; cleared at the source block). |
| $02 | `INT_EN` | RW | `$0000` | Per-source enable. |
| $04 | `INT_ACTIVE` | RO | — | `INT_PEND & INT_EN`. |
| $06 | `INT_LEVEL` | RO | — | `[2:0]` level currently presented on `/IPL2–0`. |
| $08 | `INT_SWI` | RW | `$0000` | Bits `[7:1]`: a software interrupt request at level 1–7, held until cleared. Level 1 is used only by software. |

Notes:
- The presented level is the highest level among `INT_ACTIVE` sources and `INT_SWI` bits.
- Interrupt acknowledge cycles (`FC = 111`) are answered with an autovector (vector 24 + level).
- Level 7 is edge-triggered in the 68000; a new level-7 interrupt is taken only after the level
  drops below 7 and rises again.

## 4. `TIMER` — $F00040

| Off | Name | Type | Reset | Description |
|---|---|---|---|---|
| $00 | `TMR_US_HI` | RO | 0 | Free-running microsecond counter `[31:16]`; reading latches `TMR_US_LO`. Wraps after ≈ 71.6 min. |
| $02 | `TMR_US_LO` | RO | 0 | `[15:0]`. |
| $04 | `TMR_TICK` | RW | `$0000` | Tick period in µs (1–65535); 0 disables. |
| $06 | `TMR_STAT` | W1C | 0 | `[0] TICK` `[1] WDT_PRE`. |
| $08 | `TMR_IE` | RW | 0 | Same layout. `TICK` → INTC bit 2; `WDT_PRE` → INTC bit 6. |
| $0A | `WDT_CTRL` | RW | 0 | `[0] EN`. Once set, it can only be cleared by a reset. |
| $0C | `WDT_TIMEOUT` | RW | `$03E8` | Timeout in ms (default 1000). Pre-timeout fires at half of it. |
| $0E | `WDT_KICK` | WO | — | Write `$C0DE` to restart the countdown. On timeout: CPU + chipset reset, `RSTCAUSE.WATCHDOG`. |

## 5. `UART` — $F00060

8N1, 256-byte TX and RX FIFOs.

| Off | Name | Type | Reset | Description |
|---|---|---|---|---|
| $00 | `UART_DATA` | PORT | — | Write: push `[7:0]` to the TX FIFO (stalls while full). Read: pop RX: `[8] VALID`, `[7:0]` data; returns 0 when empty. |
| $02 | `UART_STAT` | RO / W1C | — | `[0] RX_AVAIL` (RO) `[1] TX_FULL` (RO) `[2] TX_IDLE` (RO: FIFO and shifter empty) `[3] RX_OVERRUN` (W1C) `[4] FRAME_ERR` (W1C) `[5] BREAK` (W1C). |
| $04 | `UART_IE` | RW | 0 | `[0] RX_AVAIL` `[2] TX_IDLE` `[3] RX_OVERRUN` `[4] FRAME_ERR`. → INTC bit 3. |
| $06 | `UART_DIV` | RW | `277` | Baud = SYSCLK / (DIV + 1). 277 → 115 108 Bd (−0.08 %); 31 → 1 MBd; 15 → 2 MBd. |
| $08 | `UART_CTRL` | RW | `$0001` | `[0] BREAK_NMI`: a received BREAK raises the debug-break interrupt (INTC bit 7, level 7). `[1] LOOPBACK` (test). |
| $0A | `UART_RXCNT` | RO | 0 | Bytes waiting in the RX FIFO. |

## 6. `SDC` — $F00080

SD card controller, SPI mode. Two modes:

- **Raw**: the CPU shifts bytes itself and drives `/CS`. Used for card initialisation.
- **Block engine**: the controller runs single-block transactions (`CMD17` read, `CMD24` write)
  into or out of two 512-byte buffers (A, B). It serves the CPU and `STREAM`, one transaction at a
  time. STREAM gets priority while its FIFO is below `STR_URGENT`.

| Off | Name | Type | Reset | Description |
|---|---|---|---|---|
| $00 | `SDC_CTRL` | RW | `$2700` | `[0] CS` (raw mode: assert `/CS`). `[1] ENGINE` (0 = raw, 1 = block engine). `[2] HC` (card is SDHC/SDXC → block addressing; else the engine multiplies the LBA by 512). `[15:8] DIV`: SPI clock = SYSCLK / (2·(DIV+1)); reset value 39 → 400 kHz; 0 → 16 MHz. |
| $02 | `SDC_RAW` | PORT | — | Raw mode. Write: transmit `[7:0]` (stalls while a byte is shifting). Read: last received byte. |
| $04 | `SDC_STAT` | RO / W1C | — | `[0] BUSY` (RO). `[1] DONE` (W1C: CPU transaction finished). `[2] ERR` (W1C). `[3] STREAM_ACTIVE` (RO: engine is serving STREAM). `[11:8] ERRCODE` (RO): 1 command timeout, 2 R1 error, 3 data-token timeout, 4 data error token, 5 write rejected, 6 write-busy timeout. |
| $06 | `SDC_IE` | RW | 0 | `[1] DONE` `[2] ERR`. → INTC bit 1. |
| $08 | `SDC_LBA_HI` | RW | 0 | Block address `[31:16]` for the next CPU transaction. |
| $0A | `SDC_LBA_LO` | RW | 0 | `[15:0]`. |
| $0C | `SDC_CMD` | WO | — | Starts a CPU transaction. `[1:0]` op: 1 = read block → buffer, 2 = write buffer → block. `[4]` buffer (0 = A, 1 = B). Ignored, with `ERR` set, while a CPU transaction is pending. |
| $0E | `SDC_R1` | RO | — | Last R1 response byte (diagnostics). |
| $10 | `SDC_BUFSEL` | RW | 0 | `[0]` buffer accessed by the data port (0 = A, 1 = B). Writing resets the port pointer to 0. |
| $12 | `SDC_BUFPTR` | RW | 0 | `[7:0]` word index (0–255) of the data port. |
| $14 | `SDC_BUFDATA` | PORT | — | Read / write one 16-bit word of the selected buffer (first card byte = high byte); auto-increments `SDC_BUFPTR`. |

Two buffers allow the next block to be read while the CPU copies the previous one out.

## 7. `STREAM` — $F000A0

Moves compressed audio from the card to the VS1063A on its own (P-005).

**Descriptor** (written as four words through `STR_DESC`; the ring holds 64):

| Word | Bits | Field |
|---|---|---|
| 0 | `[15:0]` | Start LBA `[31:16]` |
| 1 | `[15:0]` | Start LBA `[15:0]` |
| 2 | `[15:0]` | Sector count (1–65535) |
| 3 | `[9:0]` | Valid bytes in the last sector (0 = all 512) |
| 3 | `[14]` | `IRQ`: set `DESC_DONE` when this descriptor's last byte has been sent to the decoder |
| 3 | `[15]` | `END`: end of track; set `END` after its last byte has been sent |

**Operation.** While `EN = 1` the fetcher reads the next sector of the current descriptor whenever the
FIFO has ≥ 512 bytes free. While `EN = 1` and `PAUSE = 0`, the feeder sends up to 32 bytes to the
decoder's SDI each time `DREQ` is high. A partial burst is sent only at the end of a descriptor.

| Off | Name | Type | Reset | Description |
|---|---|---|---|---|
| $00 | `STR_CTRL` | RW | 0 | `[0] EN`. `[1] PAUSE` (the fetcher keeps filling, the feeder stops). `[2] FLUSH` (WO pulse: empty the FIFO and the ring, abandon the current descriptor, zero `STR_BYTES`). `[3] FILL` (feeder sends `STR_FILL_COUNT` copies of `STR_FILL_BYTE` instead of FIFO data, regardless of `PAUSE`; clears itself when done). |
| $02 | `STR_STAT` | RO / W1C | — | `[0] RING_LOW` (W1C) `[1] DESC_DONE` (W1C) `[2] END` (W1C) `[3] ERR` (W1C: SD read error; the engine stops, `EN` is cleared) `[4] FIFO_EMPTY` (RO) `[5] UNDERRUN` (W1C: `DREQ` was high with no data while descriptors were pending) `[6] FILL_BUSY` (RO). |
| $04 | `STR_IE` | RW | 0 | Bits 0–3, 5. → INTC bit 0. |
| $06 | `STR_DESC` | PORT (WO) | — | Descriptor word port; the 4th word commits the descriptor. A commit to a full ring is dropped and sets `ERR`. |
| $08 | `STR_RING` | RO | 0 | `[6:0]` descriptors pending (including the current one); `[14:8]` free slots. |
| $0A | `STR_LOWAT` | RW | 2 | `RING_LOW` is set when pending ≤ this value (only after at least one descriptor was committed since `FLUSH`). |
| $0C | `STR_LEVEL` | RO | 0 | Bytes in the FIFO (0–4096). |
| $0E | `STR_BYTES_HI` | RO | 0 | Bytes delivered to the decoder since `FLUSH` `[31:16]`; reading latches `_LO`. Used for resume positions. |
| $10 | `STR_BYTES_LO` | RO | 0 | `[15:0]`. |
| $12 | `STR_FILL_BYTE` | RW | 0 | `[7:0]` fill byte (the decoder's `endFillByte`). |
| $14 | `STR_FILL_COUNT` | RW | 0 | Number of fill bytes to send. |
| $16 | `STR_URGENT` | RW | 2048 | While `STR_LEVEL` < this value, STREAM has priority on `SDC`. |

## 8. `VSC` — $F000C0

Access to the VS1063A's SCI registers. The SPI wires are shared with STREAM's SDI bursts (separate
`/XCS` and `/XDCS`; the decoder runs in "new mode"). An SCI transaction waits for the current burst to
finish.

| Off | Name | Type | Reset | Description |
|---|---|---|---|---|
| $00 | `VSC_SCIDIV` | RW | 9 | SCI SPI clock = SYSCLK / (2·(n+1)); 9 → 1.6 MHz (safe before `SC_CLOCKF` is raised). |
| $02 | `VSC_SDIDIV` | RW | 9 | SDI clock divider used by STREAM, same formula; raise to 1 (8 MHz) once the decoder clock multiplier is set. |
| $04 | `VSC_ADDR` | RW | 0 | `[3:0]` SCI register address. |
| $06 | `VSC_WDATA` | RW | 0 | Data for an SCI write. |
| $08 | `VSC_RDATA` | RO | 0 | Result of the last SCI read. |
| $0A | `VSC_CMD` | WO | — | 1 = SCI read, 2 = SCI write. |
| $0C | `VSC_STAT` | RO / W1C | — | `[0] BUSY` (RO) `[1] DONE` (W1C) `[2] DREQ` (RO: live pin level). Polled; no interrupt. |

The decoder's `XRESET` line is on the GPIO expander (P-003).

## 9. `LCDC` — $F000E0

Write-only LCD port with a 512-byte FIFO. Each FIFO entry carries the byte and its command / data flag,
so commands and pixel data stay in order. Whether the physical port is 8080 parallel or SPI is
**pending OQ-3**; the programming model is the same for both.

| Off | Name | Type | Reset | Description |
|---|---|---|---|---|
| $00 | `LCD_CTRL` | RW | 0 | `[0] EN`. `[15:8]` bus timing: write strobe width (parallel) or SPI divider, in SYSCLK cycles. |
| $02 | `LCD_CMD` | PORT (WO) | — | Queue `[7:0]` as a command byte. |
| $04 | `LCD_DATA` | PORT (WO) | — | Word write: queue two data bytes (high first). Byte write to the odd address: one data byte. Stalls while the FIFO is full (one byte frees every few hundred ns). |
| $06 | `LCD_STAT` | RO | — | `[0] IDLE` (FIFO empty and bus idle) `[1] FULL` `[2] CAP_SPI` (1 if this build drives the LCD over SPI). |
| $08 | `LCD_LEVEL` | RO | 0 | Bytes queued. |

The LCD's reset line is on the GPIO expander (P-003).

## 10. `I2CM` — $F00100

Byte-level I²C master plus the shared, active-low external interrupt line (wheel controller, GPIO
expander, fuel gauge; open-drain).

| Off | Name | Type | Reset | Description |
|---|---|---|---|---|
| $00 | `I2C_CTRL` | RW | `$4F00` | `[0] EN`. `[15:8] DIV`: SCL = SYSCLK / (4·(DIV+1)); 79 (`$4F`) → 100 kHz, 19 → 400 kHz. |
| $02 | `I2C_CMD` | WO | — | Bit flags, executed in this order: `[0] START`, `[2] WRITE` (`I2C_TX`), `[3] READ` → `I2C_RX`, `[4] NACK` (with `READ`: send NACK instead of ACK), `[1] STOP`. |
| $04 | `I2C_TX` | RW | 0 | `[7:0]` byte to send. |
| $06 | `I2C_RX` | RO | 0 | `[7:0]` last byte read. |
| $08 | `I2C_STAT` | RO / W1C | — | `[0] BUSY` (RO) `[1] DONE` (W1C) `[2] NACKED` (W1C: the target did not acknowledge) `[4] INT_LINE` (RO: 1 while the external line is asserted) `[5] INT` (W1C: assertion latched). |
| $0A | `I2C_IE` | RW | 0 | `[1] DONE` `[5] INT`. → INTC bit 5. |

## 11. `INPUT` — $F00120

Button bits: `[0] MENU` `[1] PREV` `[2] NEXT` `[3] PLAY` `[4] SELECT`. The Hold switch is read through the
GPIO expander (P-003).

| Off | Name | Type | Reset | Description |
|---|---|---|---|---|
| $00 | `IN_STATE` | RO | — | Debounced state, 1 = pressed. |
| $02 | `IN_EDGE` | W1C | 0 | `[4:0]` press edges, `[12:8]` release edges. |
| $04 | `IN_IE` | RW | 0 | Same layout as `IN_EDGE`. → INTC bit 4. |
| $06 | `IN_DEBOUNCE` | RW | 10 | Debounce time in ms (1–255). |
| $08 | `IN_RAW` | RO | — | Synchronised raw pin levels (1 = pressed). |

## 12. `PWR` — $F00140

| Off | Name | Type | Reset | Description |
|---|---|---|---|---|
| $00 | `PWR_STAT` | RO | — | `[0] HOLD` (the power-latch output; set by the chipset as soon as it is configured). |
| $02 | `PWR_OFF` | WO | — | Write `$DEAD` to release the power latch. If no button is pressed, the 3.3 V rail turns off within milliseconds. |

VBUS, charger status and battery alerts reach the CPU through the GPIO expander and the fuel gauge on
the shared I²C interrupt line.

## 13. `AUX` — $F00160

| Off | Name | Type | Reset | Description |
|---|---|---|---|---|
| $00 | `BEEP_HALF` | RW | 125 | Piezo tone half-period in µs (125 → 4 kHz). |
| $02 | `BEEP_LEN` | RW | 0 | Duration in units of 100 µs. Writing a non-zero value starts the tone; writing 0 stops it. A click ≈ 20 (2 ms). |
| $04 | `BEEP_STAT` | RO | — | `[0] BUSY`. |
| $06 | `BL_PWM` | RW | 0 | `[7:0]` backlight duty (0 = off, 255 = full). PWM frequency SYSCLK / 1024 = 31.25 kHz (inaudible). |

## 14. `FLASH` — $F00180

SPI master for the configuration flash (after FPGA configuration completes).

| Off | Name | Type | Reset | Description |
|---|---|---|---|---|
| $00 | `FL_CTRL` | RW | `$0000` | `[0] CS` (raw mode: assert `/CS`). `[1] AUTO` (auto-read mode). `[15:8] DIV`: SPI clock = SYSCLK / (2·(DIV+1)); 0 → 16 MHz. |
| $02 | `FL_RAW` | PORT | — | Raw mode. Write: transmit `[7:0]`. Read: last received byte. |
| $04 | `FL_STAT` | RO | — | `[0] BUSY`. |
| $06 | `FL_ADDR_HI` | RW | 0 | Auto-read start address `[23:16]` in `[7:0]`. |
| $08 | `FL_ADDR_LO` | RW | 0 | `[15:0]`. |
| $0A | `FL_READ` | PORT (RO) | — | Auto-read: each word read returns the next two flash bytes (big-endian). Setting `AUTO` (or writing an address while `AUTO = 1`) issues a fast-read command; the engine prefetches, so a sequential copy loop stalls at most ≈ 1 µs per word. Bytes also feed the CRC when `CRC_CTRL.FLASH_FEED = 1`. |

Stage-1 copy loop (illustrative):

```asm
        lea     $F00180,a0          ; FLASH block
        move.w  #$0010,6(a0)        ; FL_ADDR_HI = $10   -> $100000
        clr.w   8(a0)               ; FL_ADDR_LO = 0
        move.w  #$0002,(a0)         ; FL_CTRL: AUTO, 16 MHz
        lea     $0E0000,a1          ; destination
        move.w  #(LEN/2)-1,d0
.copy:  move.w  10(a0),(a1)+        ; FL_READ -> RAM
        dbra    d0,.copy
```

## 15. `DBG` — $F001A0

| Off | Name | Type | Reset | Description |
|---|---|---|---|---|
| $00 | `DBG_MUX` | RW | 0 | Signal group on the 8-pin LA/PMOD header: 0 off (inputs), 1 CPU strobes, 2 SD SPI, 3 decoder SPI, 4 LCD port, 5 I²C + interrupt line, 6 UART + interrupt levels, 7 GPIO (from `DBG_GPIO`). Exact signal lists live in the RTL documentation. |
| $02 | `DBG_GPIO` | RW | 0 | `[7:0]`: pin values driven in group 7; reads return the actual pin levels in every group. |
| $04 | `DBG_LED` | RW | 0 | `[1:0]` the two FPGA-driven LEDs. |
| $06 | `TRC_CTRL` | RW | 0 | `[0] ARM` (record into the ring). `[1] FROZEN` (RO). `[2] TRIG_ADDR` `[3] TRIG_BERR` `[4] TRIG_NMI` (trigger enables). `[5] TRIG_NOW` (WO: freeze now). `[15:8]` cycles to record after the trigger (0–127). |
| $08 | `TRC_TADDR` | RW | 0 | Trigger address as the chipset sees it: `[15:12]` = A23–A20, `[8:0]` = A9–A1. |
| $0A | `TRC_INDEX` | RW | 0 | `[6:0]` entry to read (0 = oldest); writing also resets the word counter. |
| $0C | `TRC_DATA` | PORT (RO) | — | Reads the selected entry as four words, then advances to the next entry. |
| $0E | `DBG_SCRATCH` | RW | 0 | Free for software (e.g. post codes visible in the bus trace). |

Trace entry (64 bits, word 0 first):

| Word | Bits | Field |
|---|---|---|
| 0 | `[15:0]` | CPU clocks since the previous entry (saturating) |
| 1 | `[15:12]` | A23–A20 |
| 1 | `[8:0]` | A9–A1 |
| 2 | `[15:0]` | D15–D0 (as driven or sampled) |
| 3 | `[2:0]` | FC2–FC0 |
| 3 | `[3]` | R/W (1 = read) |
| 3 | `[4]` | `UDS` active |
| 3 | `[5]` | `LDS` active |
| 3 | `[6]` | `/BERR` asserted |
| 3 | `[10:8]` | Wait states inserted (saturating at 7) |
| 3 | `[15]` | This is the trigger entry |

## 16. Open items

- `LCDC` physical port (OQ-3).
- Default `STR_URGENT` and `STR_LOWAT` to be tuned in co-simulation against the decoder's real
  stream-buffer size (verify).
- Bus-timing details (wait states, `/DTACK` timing) go into `04-bus-timing.md`.
