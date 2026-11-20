# Roadmap

Updated: 2026-11-20. Work order (D-023): **specifications → emulator + RTL + firmware → (optional)
dev-board validation → Rev A hardware → order and bring-up → Rev B.** The simulation phase may change
the architecture; every change goes through [docs/decision-log.md](docs/decision-log.md).

## Current position

Phase 1, specifications. Docs 00–03 and the decision log are drafted; P-001…P-009 are accepted.
**Next: doc 04 (bus interface and timing).**

## Phase 1 — Specifications *(in progress)*

| # | Document | State | Scope notes for the author |
|---|---|---|---|
| 00 | Requirements | Draft | — |
| 01 | Architecture | Draft | Includes the FPGA pin-budget estimate (§3) and the EBR budget (§4). |
| 02 | Memory map | Draft | — |
| 03 | Chipset registers | Draft | — |
| 04 | Bus interface and timing | **Next** | 68000 bus states S0–S7 at CPUCLK = SYSCLK / 2. Zero-wait SRAM read/write against IS62WV51216 (55 ns) timing. `/DTACK` and `/BERR` generation, register access wait states, FIFO stall rules, bus timeout. IACK / autovector cycle. Reset sequence (`/RESET`+`/HALT`), boot-ROM overlay, glitch-free CPU clock divider. iCE40 I/O timing budget. Waveform diagrams. |
| 05 | Boot, image format, update | Planned | Stage-1 ROM, tactboot, image header (already in 02 §7), flash slot B. UART upload protocol (FR-DV-2). Recovery screen, boot-time button combinations, diagnostics mode (P-007). Updating tactboot / bitstream. |
| 06 | Power tree and budget | Planned | BQ24074 settings. LDOs: 3.3 V, 1.2 V FPGA core, 1.8 V VS1063A CVDD, AVDD 2.6–3.6 V, iCE40 `VPP_2V5`. LCD VDD ≤ 3.3 V, so use an accurate 3.3 V LDO. Backlight: a constant-current sink from VSYS (VF 3.1–3.3 V, 45 mA max), PWM from the FPGA. Soft-latch circuit (buttons wake through diodes; FPGA hold line). USB-C CC resistors. Off-state drain ≤ 50 µA. INA226 shunt. Sequencing. Current table and battery life. Close OQ-1 (USB host vs charger). |
| 07 | FPGA pinout | Planned | Verify the 107-PIO count from the Lattice TQ144 package file. Exact bank and pin assignment, `GBIN` for the 32 MHz clock, config-SPI pins. Reconcile with 01 §3; use the reclaim order if short. |
| 08 | BOM and sourcing | Planned | LCSC numbers (basic vs extended). Hand-solder list (CPU, QT2120). Rochester / DigiKey / Orient sourcing, cost per Rev A unit (≤ $100–150), purchase timing. |
| 09 | Firmware architecture | Planned | m68k-elf-gcc toolchain, layers (chipset HAL, drivers, FatFs, tag parsers, library, player, UI). Interrupt handlers, main loop, crash handler, logging, debug monitor using the tracer breakpoints. Build system. Close OQ-2 (VS1063A tone / EQ). |
| 10 | UI specification | Planned | Every screen and menu, fonts, wheel acceleration, clicker, Now Playing, Settings, Disk Mode / recovery / diagnostics screens. 128×128 mock-ups. |
| 11 | Library index format | Planned | `LIBRARY.DB` and `STATE.DAT` layouts, sorting (FR-LB-6), incremental update, paging into 1 MB RAM. |
| 12 | Emulator and co-simulation | Planned | `emu/` (Musashi + C models + SDL video / audio, SD card image). `sim/` (Musashi + Verilator RTL). Conformance-test framework and CI. Close OQ-4 (gapless). |
| 13 | Bring-up plan | Planned | Step-by-step with a multimeter and an 8-channel LA only: rails → FPGA config → chipset self-test without the CPU → CPU run loop → SRAM → UART → SD → decoder sine test → LCD → input → battery. |

**Exit criteria:** every doc at least Draft and internally consistent (01 ↔ 02 ↔ 03 ↔ 04); the owner
has read through them.

## Phase 2 — Emulator, RTL, firmware

- Install the toolchain. yosys is already installed; still needed: nextpnr-ice40, icestorm,
  Verilator, an m68k-elf GCC, SDL2; vendored libraries: Musashi, dr_mp3, dr_flac, FatFs.
- `emu/`: a fast full-system emulator in which the firmware boots and plays music through the host's
  audio.
- `fw/`: stage-1 boot ROM (assembly), tactboot, main firmware.
- `rtl/`: chipset Verilog; `sim/`: co-simulation running the same firmware binary; shared
  conformance tests (NFR-7).
- **Purchases at the start of this phase:** 3 × Orient AMG128128PR LCD (small vendor stock), an
  iCE40 dev board, a VS1053/VS1063 breakout, a microSD breakout, a cheap FX2 8-channel logic analyser.
- **Exit:** the firmware runs identically in emu and co-sim; the RTL fits the HX4K with timing margin
  at 32 MHz; architecture changes are recorded in the decision log.

## Phase 3 — Dev-board validation *(optional, D-026)*

The LCD, SD and decoder RTL on the iCE40 dev board with the breakout modules and the real LCD.

## Phase 4 — Rev A hardware

KiCad 10 project in `hw/rev-a/`. Claude produces most of the schematic and layout (D-022) with ERC /
DRC in the loop; the owner checks it in the GUI. Outputs are JLCPCB fabrication files, BOM and CPL.
**Purchases:** MC68SEC000PB16 × 4 (Rochester) and AT42QT2120 (DigiKey).

## Phase 5 — Order and bring-up

JLCPCB PCBA order; hand-solder the CPU and the touch controller; follow doc 13.

## Phase 6 — Rev B (pocket form factor)

D-024; geometry from the Rev A lessons (OQ-5).
