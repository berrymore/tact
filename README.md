# tact

A pocket music player in the spirit of 2001 — built as a tiny single-board computer.

- A real, discrete **Motorola 68000** (MC68SEC000) runs the firmware and is in charge of everything.
- An **iCE40 FPGA** plays the role of the custom chips (in the Amiga tradition): bus glue, interrupts,
  timers, an SD-to-decoder streaming engine, LCD and input controllers.
- A **VS1063A** hardware decoder plays MP3 / AAC / Ogg Vorbis / FLAC / WAV.
- A 2.1" **128×128 monochrome LCD** with a white backlight, a **capacitive touch wheel**, five clicky
  buttons and a **Hold** switch.
- **microSD** storage with a USB **Disk Mode** ("Do not disconnect").
- Single-cell LiPo, USB-C charging, 10+ hours of playback.

> **Status:** design phase. The documentation is being written first; then a full-system
> emulator and the firmware; hardware comes after the simulation is solid. Nothing has been built yet.

## Documentation

Start with [`docs/README.md`](docs/README.md). Key documents:

| Document | What it covers |
|---|---|
| [Requirements](docs/00-requirements.md) | What tact must do (v1 scope) |
| [Architecture](docs/01-architecture.md) | Blocks, data flows, clocks, power, boot, software layers |
| [Memory map](docs/02-memory-map.md) | 68000 address space, RAM / flash / SD card layouts |
| [Decision log](docs/decision-log.md) | Every design decision and why it was made |

## Repository layout

| Path | Contents | License |
|---|---|---|
| `docs/` | Specifications | CERN-OHL-P-2.0 |
| `hw/` | KiCad projects (Rev A, Rev B) | CERN-OHL-P-2.0 |
| `rtl/` | Chipset Verilog | MIT |
| `fw/` | 68000 firmware and boot loaders | MIT |
| `emu/` | Fast full-system emulator (Musashi + C models) | MIT |
| `sim/` | RTL co-simulation (Musashi + Verilator) and conformance tests | MIT |
| `tools/` | Host-side utilities | MIT |

Directories appear as the work reaches them. See [`LICENSE.md`](LICENSE.md).

## Not affiliated

tact is an independent hobby project. It is not affiliated with, endorsed by, or derived from any
commercial music player or its manufacturer; all names, logos and fonts are original.
