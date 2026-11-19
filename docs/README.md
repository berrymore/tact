# tact documentation

Read in order. Status is per document.

| # | Document | Status |
|---|---|---|
| 00 | [Requirements](00-requirements.md) | Draft |
| 01 | [Architecture](01-architecture.md) | Draft |
| 02 | [Memory map and storage layouts](02-memory-map.md) | Draft |
| 03 | [Chipset register specification](03-chipset-registers.md) | Draft |
| 04 | 68000 bus interface and timing | Planned |
| 05 | Boot, image format and update | Planned |
| 06 | Power tree and power budget | Planned |
| 07 | FPGA pin budget and pinout | Planned |
| 08 | Bill of materials and sourcing | Planned |
| 09 | Firmware architecture | Planned |
| 10 | UI specification (screens and menus) | Planned |
| 11 | Library index format | Planned |
| 12 | Emulator and co-simulation | Planned |
| 13 | Bring-up plan | Planned |
| — | [Decision log](decision-log.md) | Living |

## Conventions

- Hex addresses use the Motorola `$` prefix; values are big-endian.
- Signal names: active-low signals have a leading `/` (`/DTACK`).
- `D-nnn` = decided, `P-nnn` = proposed and awaiting review, `OQ-n` = open question,
  **(verify)** = waiting for a datasheet check.
