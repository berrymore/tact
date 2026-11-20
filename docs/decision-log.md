# Decision Log

Every design decision, with the reason it was made and what it replaced. Entries are never deleted:
a reversed decision is marked **Superseded** and points to its replacement.

- `D-nnn` — **decided** by the project owner (initial design review, 2026-11-20).
- `P-nnn` — **proposed** while writing the specifications, then reviewed by the owner. Accepted
  proposals keep their `P` number so that references in the docs stay valid.

## Decided

| ID | Decision | Why | Rejected alternatives |
|---|---|---|---|
| D-001 | Balance authenticity and usability; when they conflict, the device must work in a pocket. The CPU is a standalone chip, single-board-computer style. | The goal is a real retro computer that is also a usable player. | A pure "retro computer that happens to play music"; a pure appliance. |
| D-002 | Audio is decoded by a hardware decoder; the CPU orchestrates. | No period CPU can decode MP3 in software (needs ~20+ MIPS; a 68000 manages ~2–3). Period-correct: 1998-era players paired a controller with a hardware decoder. | PCM/ADPCM only through a DAC; FPGA decoding. |
| D-003 | CPU: MC68SEC000PB16 (16 MHz, 16-bit bus, TQFP-64 10 × 10 mm, 0.5 mm pitch). Buy 3–4 from Rochester Electronics (authorised reseller). | 32-bit internal, flat 16 MB address space, GCC; fully static 3.3 V part. | W65C02S / W65C816S (8 MHz at 3.3 V, cramped); Z80 (end-of-life 2024, 5 V); 8086 / V20 (NOS only, NMOS, 5 V). |
| D-004 | microSD + a USB card-reader bridge (GL823K) and an SD mux = Disk Mode. | Normal file copying from any computer; the period "Do not disconnect" experience. | Removing the card; USB mass storage implemented by the CPU (slow, complex). |
| D-005 | Monochrome, near-square, backlit LCD: Orient AMG128128PR (2.1", 128×128, UC1617, 4 grey). | Closest buyable match to the early-2000s look; positive FSTN; datasheet available. | Colour TFT; Sharp memory LCD (no backlight); OLED; 160×160 UC1698 (large, awkward pixel layout); 160×128 Maclight (inquiry-only). |
| D-006 | Capacitive wheel on the PCB + mechanical buttons (centre + 4) + Hold switch; controller AT42QT2120. | The look of a touch wheel with tactile clicks; native wheel support; TSSOP is hand-solderable. | Alps RKJXT mechanical wheel; buttons only; IQS7222A (QFN, partial NRND); FPGA-native sensing (hard to tune without a scope). |
| D-007 | Rev A = development board; Rev B = pocket form factor. Rev A is a single 100 × 100 mm, 4-layer PCB; the display hangs on its FPC. | A CPU + FPGA + SD + decoder board will need a respin; 100 × 100 is the cheapest JLCPCB size. | Going straight to the pocket form factor; a two-board Rev A sandwich. |
| D-008 | Power: 1S LiPo ≈ 1200 mAh, USB-C, BQ24074 power path, MAX17048 gauge, soft latch, ≥ 10 h target. **LDOs instead of a buck-boost.** | At ≈ 80 mA the LDO's efficiency is close to a buck-boost's; quieter audio, layout-tolerant, debuggable without a scope. Loses ~5–8 % of capacity. | Buck-boost 3.3 V; chargers without power path (TP4056, MCP73831). |
| D-009 | JLCPCB assembles every part it stocks; the owner hand-solders only the CPU and the touch controller; no BGA. | Owner has a soldering iron (no hot air) and limited experience. | Full hand assembly. |
| D-010 | Budget ≈ $100–150 per assembled Rev A unit, excluding tools. | Proposed in review; not objected to. | — |
| D-011 | Chipset FPGA: iCE40HX4K-TQ144 with the open toolchain. | Mature yosys/nextpnr/icestorm, native on macOS; TQFP is hand-solderable and JLCPCB-stocked. | Gowin GW1N-UV4 (less mature open flow); CPLDs (CoolRunner-II / XC9500XL discontinued, MAX II not for new designs). |
| D-012 | HDL: Verilog. | Best-supported language in yosys. | VHDL, Amaranth, SpinalHDL. |
| D-013 | 1 MB SRAM (IS62WV51216) plus a footprint for a second 1 MB chip. | Enough for firmware + a paged library; cheap upgrade path. | 512 KB; 2 MB TSOP-I-48 (harder to solder). |
| D-014 | Boot: the FPGA's boot ROM is overlaid at `$000000`; the firmware is loaded from the SD card; missing firmware → Disk Mode (unbrickable). | Firmware updates by copying a file; cannot brick. | Parallel NOR flash on the bus. Refined by P-001. |
| D-015 | The library is indexed on the device (tags → index file on the card), plus a Folders view. | No companion PC software needed. | A PC-side database tool; folders only. |
| D-016 | UI v1 scope as in the requirements; device name "tact"; original logo and fonts. | Keep v1 focused; avoid third-party trademarks. | — |
| D-017 | Rev A debug: FT2232HL (flash programming + UART), FPGA debug mux to an LA header, bus tracer, INA226, test points. | Bring-up without an oscilloscope; one USB cable for programming and console. | Full-bus logic-analyser headers (useless with an 8-channel analyser, cost board area). |
| D-018 | Simulation first, two levels running the same firmware: a fast emulator (Musashi + C models, real audio) and an RTL co-simulation (Musashi + Verilator), plus shared conformance tests. | Firmware can be developed before hardware exists; the gateware is proven before boards are ordered; the two models cannot drift apart silently. | Emulator only; RTL simulation only. |
| D-019 | Decoder: VS1063A (VS1053B kept as a fallback on the same footprint). | Native FLAC (≤ 48 kHz, ≤ 24-bit), better tone controls; VLSI calls it a "pin-compatible firmware upgrade to the VS1053b". | VS1053B (FLAC only via a patch); VS1003B (no AAC/Ogg/FLAC). |
| D-020 | Headphone path: decoder line output → TPA6132A2 → jack, sleeve on real ground. | The decoder's direct drive references the sleeve to ≈ 1.2–1.6 V (GBUF); grounding the sleeve through a car AUX input or a USB-powered amplifier would short it. | Direct GBUF drive. |
| D-021 | Open source: public GitHub repository; CERN-OHL-P-2.0 for hardware and docs, MIT for code. | Share freely. | Private repository; reciprocal licences. |
| D-022 | PCB workflow: Claude does most of the schematic and layout work (generated schematic, scripted placement, routing with cleanup, ERC/DRC in the loop); the owner does GUI checks with step-by-step instructions. | The owner has almost no layout experience. | The owner draws everything; fully autorouted. |
| D-023 | Work order: complete docs → simulation + firmware (the architecture may change) → hardware design → JLCPCB order → bring-up. | Settle the architecture cheaply before committing to copper. | Hardware first. |
| D-024 | Rev B vision: ≈ 102 × 62 × 20 mm; the PCB is the white front face with the display behind a window; wheel under a matte film; JLC3DP translucent back; jack + Hold on top, USB-C at the bottom; 603450 cell. Exact geometry after Rev A. | Early-2000s look, fully orderable from JLCPCB. | — |
| D-025 | Text: Latin + Cyrillic bitmap fonts, UTF-8 throughout, CP1251 option for legacy tags; other scripts later as font files. | The owner's library contains Cyrillic titles; old MP3s use CP1251 ID3v1. | Latin only. |
| D-026 | Validate peripherals on an iCE40 dev board with breakout modules (decoder, microSD, the real LCD) during the simulation phase. Optional. | Proves the LCD / SD / decoder RTL on real silicon before Rev A; tool practice. | Going straight from simulation to Rev A. |
| D-027 | Conversation in Russian; every repository artifact in English. | Public repository. | — |

## Proposals

All of P-001…P-009 were **accepted** on 2026-11-20 (the owner delegated the call). P-002 was revised
at acceptance: see the entry.

| ID | Proposal | Why | Affects |
|---|---|---|---|
| P-001 | **Two-stage boot.** Stage 1: a ≤ 512 B ROM in FPGA block RAM copies **tactboot** from the SPI configuration flash into SRAM. tactboot (C) then loads `FIRMWARE.BIN` from the card. Refines D-014. | The FPGA has only 10 KB of block RAM, and a FAT + SD + LCD loader does not fit. The flash has megabytes free. The device stays unbrickable for anything stored on the card. | Memory map, boot ROM, `FLASH` block |
| P-002 | The chipset sees the **whole** address bus `A1–A23` (inputs only) and decodes it fully; no bus mastering (`/BR` tied inactive). `A10–A19` are the last pins to reclaim if the FPGA pin budget runs short; the I/O page then mirrors inside `$Fxxxxx`. *Revised at acceptance; the first draft gave the chipset only `A1–A9` + `A20–A23`.* | Full addresses make the bus tracer show real addresses and act as a hardware breakpoint — the main debugging tool without an oscilloscope (NFR-9). The pin estimate (106 of 107) allows it once the LEDs move to the expander. DMA does not need the CPU bus because STREAM runs SD → decoder directly. | Memory map, `DBG`, schematic |
| P-003 | Slow signals (Hold, charger status, card detect, resets, enables, 2 LEDs) go through an I²C GPIO expander. | Frees ~10 FPGA pins. | Schematic, `I2CM` |
| P-004 | Clocking: 32 MHz oscillator = SYSCLK; CPU clock = SYSCLK / 2 (16 MHz), scalable at run time; no PLL needed. | The 68000 needs a ≈ 50 % duty cycle; generating it in the FPGA makes bus timing synchronous and known. | Bus timing, power |
| P-005 | STREAM engine: a descriptor ring of sector extents, 4 KB FIFO, `DREQ`-paced 32-byte SDI bursts; the firmware never writes to the card while streaming. | The CPU never touches audio bytes; the small FIFO tolerates read latency but not SD write busy times. | Registers, firmware |
| P-006 | Support exFAT as well as FAT32. | SDXC cards (≥ 64 GB) ship exFAT-formatted; FatFs supports it. | Firmware size |
| P-007 | Diagnostics mode selected by a boot-time button combination (display, input, audio sine, SD, battery, chipset self-tests). | Bring-up and field checks without tools. | tactboot, firmware |
| P-008 | Chipset registers are supervisor-only (user-mode access → `/BERR`); a bus timeout → `/BERR`; the bus tracer can freeze on `/BERR`. | Cheap protection; turns wild pointers into immediate, traceable faults. | `SYS`, `DBG` |
| P-009 | LCD on dedicated FPGA pins in 8080 parallel mode (11 pins: D0–D7, `CD`, `/CS0`, `/WR0`; read strobe tied off; reset via the expander; `PS1/PS0` set by solder jumpers on Rev A). The UC1617s SPI mode is a fallback that frees 7 pins. Resolves OQ-3. | The module datasheet documents only 8080 / 6800; keeping the LCD off the CPU bus protects bus signal integrity with the display on a cable; the pin budget allows it. | Schematic, `LCDC` |
