# CLAUDE.md — tact

tact is a pocket music player in the early-2000s style, built as a small single-board computer: a
discrete MC68SEC000 CPU, an iCE40HX4K FPGA as the "chipset", a VS1063A decoder, a 128×128 UC1617s
LCD, microSD with a USB Disk Mode. The repository is public (github.com/berrymore/tact) and open
source.

## Start here

1. [ROADMAP.md](ROADMAP.md): the current phase, what is done, what is next, and scope notes for each
   planned document.
2. [docs/README.md](docs/README.md): the document index and conventions.
3. [docs/decision-log.md](docs/decision-log.md): every decision with its reasoning.

The docs are the source of truth. Read the relevant ones before changing anything; never contradict
them silently.

## Working with the owner

- **Language:** chat in Russian; every repository artifact (docs, code, comments, commit messages)
  in English.
- The owner is a hobbyist with almost no PCB-layout experience. Claude does most of the schematic and
  layout work (D-022). When the owner has to act in a GUI, give step-by-step instructions.
- **Owner's tools:** a soldering iron (no hot air) and a True RMS multimeter; a cheap 8-channel FX2
  logic analyser is planned; no oscilloscope. No BGA. The owner hand-solders only the CPU and the
  AT42QT2120; JLCPCB assembles the rest. Everything must be debuggable without a scope (NFR-9).
- **Authenticity matters:** a real CPU and a chipset, in the spirit of a single-board computer. When a
  trade-off gives some of it up for pins, cost or simplicity, say so explicitly.
- **The owner makes the decisions.** If writing a spec requires deviating from or extending a `D-nnn`
  decision, add a `P-nnn` proposal to the decision log and raise it in chat instead of changing the
  design silently. Never delete an entry: mark a reversed one Superseded. For design questions with
  several real options, ask in numbered rounds, each question with a recommended answer.

## Documentation conventions

- Files are `docs/NN-name.md`, with a status line at the top (Planned / Draft / Review). When a
  document's status changes, update the table in `docs/README.md` and in `ROADMAP.md`.
- Hex uses the Motorola `$` prefix and big-endian byte order; active-low signals have a leading `/`.
- Cite IDs: `D-nnn`, `P-nnn`, `OQ-n`, `FR-*`, `NFR-*`.
- Mark unchecked datasheet facts **(verify)**, and check them against primary datasheets (a subagent
  with web access) before a doc leaves Draft.
- Keep 01 (architecture) ↔ 02 (memory map) ↔ 03 (registers) consistent. The emulator's C models and
  the RTL must implement doc 03 identically; shared conformance tests enforce this (NFR-7).
- The FPGA pin budget is ≈ 106 of 107 (01 §3). Any new FPGA signal has to be paid for from the
  reclaim list there. The EBR budget is in 01 §4 (4 of 20 blocks spare).

## Repository

- **Licences:** `docs/` and `hw/` are CERN-OHL-P-2.0; `rtl/`, `fw/`, `emu/`, `sim/`, `tools/` are
  MIT (see `LICENSE.md`).
- **Commits:** a scope prefix (`docs:`, `rtl:`, `fw:`, `emu:`, `sim:`, `hw:`, `tools:`), with a body
  that explains why.
- **Environment:** macOS, KiCad 10, `gh` authenticated. yosys is installed (Homebrew); nextpnr-ice40,
  icestorm, Verilator and an m68k-elf GCC are not installed yet.
