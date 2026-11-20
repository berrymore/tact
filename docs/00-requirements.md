# 00 — Product Requirements (v1)

Status: **Draft** · Updated: 2026-11-20 · Decisions referenced as `D-nnn` / `P-nnn` are in
[decision-log.md](decision-log.md).

## 1. Vision

tact is a pocket music player with the look and feel of an early-2000s player: a near-square
monochrome screen with a white backlight, a touch wheel, a Hold switch and a plain list-based UI.

Under the hood it is a small single-board computer. A discrete Motorola 68000 executes the firmware
and controls everything; an FPGA plays the role of custom chips; every peripheral on the board exists
only to play music.

**Design priority (D-001).** Authenticity and usability are balanced. When they conflict, the device
must remain something you actually carry in a pocket.

## 2. Goals and non-goals

### Goals

| ID | Goal |
|---|---|
| G1 | A real, standalone CPU (68000) executes the firmware from RAM; the chipset lives in an FPGA. |
| G2 | Plays a typical personal library from microSD through headphones without conversion. |
| G3 | Browse by Playlists / Artists / Albums / Songs / Folders using the wheel. |
| G4 | USB Disk Mode for loading music and firmware. |
| G5 | ≥ 10 h of playback per charge (target ≈ 15 h). |
| G6 | Buildable by a hobbyist: JLCPCB assembly plus hand soldering of a few non-BGA parts. |
| G7 | Fully open: hardware, gateware, firmware, tools and documentation. |

### Non-goals for v1

Bluetooth / Wi-Fi, recording, photos / video, colour, games, clock and alarms, DRM, USB audio class,
on-device file management (delete / rename), sample rates above 48 kHz. Gapless playback is a
stretch goal (OQ-4).

## 3. Usage scenarios

| ID | Scenario |
|---|---|
| S1 | **Listen on the go.** Device in a pocket, Hold on, headphones in. |
| S2 | **Load music.** Plug USB-C into a computer → Disk Mode → copy files → eject → the device updates its library. |
| S3 | **Update firmware.** Copy `FIRMWARE.BIN` into `/TACT/` in Disk Mode, unplug, the device reboots into it. |
| S4 | **Charge.** Any USB-C charger; the device stays usable while charging. |
| S5 | **Hack (Rev A).** Connect the debug USB port, load firmware over UART, watch the console, capture bus traces. |

## 4. Functional requirements

### 4.1 Playback

| ID | Requirement |
|---|---|
| FR-PB-1 | Formats: MP3 (MPEG-1/2 Layer III, CBR/VBR ≤ 320 kbps), AAC-LC (`.m4a`, `.aac`), Ogg Vorbis, FLAC (16/24-bit, ≤ 48 kHz, stereo), WAV (16-bit PCM). |
| FR-PB-2 | Play / pause, next, previous (within the first 3 s → previous track, otherwise restart), scrub within a track. |
| FR-PB-3 | Shuffle: Off / Songs / Albums. Repeat: Off / One / All. |
| FR-PB-4 | Volume is digital (in the decoder), set with the wheel on Now Playing, ≥ 32 steps. |
| FR-PB-5 | Tone: bass and treble controls plus a few presets (exact capability per VS1063A, OQ-2). |
| FR-PB-6 | Resume: after power-on the last track and position are restored, paused. |
| FR-PB-7 | From selecting a track to hearing it: ≤ 0.5 s (target). |
| FR-PB-8 | No audible glitches while navigating menus or browsing the library during playback. |

### 4.2 Library and browsing

| ID | Requirement |
|---|---|
| FR-LB-1 | Main menu: Music, Shuffle Songs, Settings, About. Music: Playlists, Artists, Albums, Songs, Folders. |
| FR-LB-2 | Metadata: ID3v1, ID3v2.2–2.4 (UTF-8 / UTF-16 / ISO-8859-1), Vorbis comments (Ogg, FLAC), MP4 atoms (best effort). File name is the fallback title. |
| FR-LB-3 | Setting "Legacy tags": how 8-bit tag text is decoded — Latin-1 (default) or CP1251. |
| FR-LB-4 | The library index is built on the device and stored on the card (`/TACT/LIBRARY.DB`). After Disk Mode it is updated incrementally with an "Updating library…" progress screen. |
| FR-LB-5 | Scale: ≥ 5,000 tracks with responsive browsing (target 10,000). |
| FR-LB-6 | Sorting is case-insensitive for Latin and Cyrillic; a leading "The " is ignored for artists. |
| FR-LB-7 | Playlists: `.m3u` / `.m3u8` anywhere on the card; entries relative to the playlist or absolute from the card root. |
| FR-LB-8 | Folders: browse the card's directory tree; play a folder in order. |

### 4.3 User interface

| ID | Requirement |
|---|---|
| FR-UI-1 | Display 128×128, 4 grey levels, white LED backlight; backlight timer Off / 2 / 5 / 10 / 20 s / Always. |
| FR-UI-2 | Layout: title bar (screen title, play state, battery) above a scrolling list; the selected row is inverted. |
| FR-UI-3 | Now Playing: title, artist, album, "n of m", progress bar, elapsed / remaining time. Wheel = volume; Select toggles scrub mode. |
| FR-UI-4 | Inputs: Menu (back), Previous, Next, Play/Pause, Select (centre), wheel with acceleration, Hold switch. While Hold is on, inputs are ignored and a lock icon appears. |
| FR-UI-5 | Clicker: an audible click per wheel step from a piezo; can be turned off. |
| FR-UI-6 | Text: original bitmap font(s) covering Latin and Cyrillic; UTF-8 internally; missing glyphs drawn as a box. |
| FR-UI-7 | Settings: Shuffle, Repeat, Tone, Backlight, Contrast, Clicker, Legacy tags, Auto-off, About (firmware version, track count, card capacity / free), Reset settings. |
| FR-UI-8 | Identity: name "tact", original logo and fonts; no third-party trademarks, logos or fonts. |
| FR-UI-9 | Responsiveness: input → visible feedback ≤ 50 ms; list scrolling ≥ 25 fps. |

### 4.4 Storage and USB

| ID | Requirement |
|---|---|
| FR-ST-1 | microSD (SDSC / SDHC / SDXC) formatted FAT32 or exFAT, long file names (P-006). |
| FR-ST-2 | Disk Mode: the card is handed to a USB mass-storage bridge; the screen shows "Do not disconnect"; firmware does not touch the card. |
| FR-ST-3 | Disk Mode starts when a USB host is detected (detection method: OQ-1) or by a boot-time button combination. |
| FR-ST-4 | On USB disconnect the card returns to the system, is remounted, and the library is checked for changes. |
| FR-ST-5 | Firmware never writes to the card while audio is streaming; writes happen only when paused / stopped / shutting down (P-005). |

### 4.5 Power

| ID | Requirement |
|---|---|
| FR-PW-1 | One Li-ion/LiPo cell; USB-C 5 V sink-only charging (5.1 kΩ CC pull-downs); operation while charging (power path). |
| FR-PW-2 | Any button (Hold off) turns the device on; holding Play/Pause for 2 s turns it off; auto-off after a configurable idle time while paused (default 5 min). |
| FR-PW-3 | Off-state battery drain ≤ 50 µA (target ≤ 20 µA). |
| FR-PW-4 | Battery percentage from a fuel-gauge IC; warning at 10 %; state saved and safe shutdown at ≈ 3 %. |
| FR-PW-5 | ≥ 10 h playback with the backlight mostly off (target ≈ 15 h with a ≈ 1200 mAh cell). |
| FR-PW-6 | Charging state shown on screen (and on an LED on Rev A). |

### 4.6 Boot and update

| ID | Requirement |
|---|---|
| FR-BT-1 | From button press to the main menu ≤ 3 s (target), excluding a library update. |
| FR-BT-2 | The main firmware is loaded from `/TACT/FIRMWARE.BIN` and verified (CRC-32) before it runs. |
| FR-BT-3 | Unbrickable: missing or corrupt firmware → recovery screen and Disk Mode, so a good image can be copied. |
| FR-BT-4 | Boot-time button combinations: Disk Mode, Diagnostics. |
| FR-BT-5 | Diagnostics mode: display, input, audio (sine test), SD, battery and chipset self-tests. |

### 4.7 Development and debug (Rev A)

| ID | Requirement |
|---|---|
| FR-DV-1 | Debug USB-C (FT2232HL): FPGA configuration-flash programming (iceprog-compatible) and a UART console. |
| FR-DV-2 | UART firmware upload into RAM and run, without touching the card. |
| FR-DV-3 | FPGA debug mux to an 8-signal logic-analyser / PMOD header; bus tracer holding recent 68000 bus cycles with full addresses; hardware breakpoint / watchpoint through the tracer trigger. |
| FR-DV-4 | Battery current monitor (INA226) readable by firmware. |
| FR-DV-5 | Test points on every power rail. |

## 5. Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-1 | Licences: CERN-OHL-P-2.0 (hardware, docs), MIT (code). Third-party code must be permissively licensed. |
| NFR-2 | Parts are purchasable new from authorised channels; prefer the LCSC / JLCPCB library; no BGA. |
| NFR-3 | Hand-soldered parts are limited to the CPU and the touch controller; everything else is JLCPCB-assembled. |
| NFR-4 | Cost ≤ $100–150 per assembled Rev A unit, excluding tools (D-010). |
| NFR-5 | Rev A: one PCB ≤ 100 × 100 mm, 4 layers, single-sided assembly. |
| NFR-6 | Rev B: ≈ 102 × 62 × 20 mm. |
| NFR-7 | Every chipset register is specified in the register spec and implemented identically by the emulator's C models and by the RTL; shared conformance tests enforce this. |
| NFR-8 | Headphone output is ground-referenced (sleeve = GND), drives 16–32 Ω, and has no audible hiss with sensitive IEMs at idle. |
| NFR-9 | Debuggable without an oscilloscope: logic analyser + multimeter + built-in instrumentation are enough for bring-up. |
| NFR-10 | All repository artifacts are in English. |

## 6. Open questions

| ID | Question | Blocks |
|---|---|---|
| OQ-1 | How to tell a USB host from a plain charger to enter Disk Mode automatically (bridge status pin, D+/D− sensing, or ask the user)? | FR-ST-3 |
| OQ-2 | VS1063A tone / EQ capabilities (FLAC limits resolved: ≤ 48 kHz, ≤ 24-bit). | FR-PB-5 |
| OQ-3 | ~~LCD module bus mode~~ — resolved: 8080 parallel (P-009). | — |
| OQ-4 | Is gapless playback feasible with the streaming engine? | Stretch |
| OQ-5 | Rev B mechanics (wheel geometry, button caps, display mounting) — after Rev A testing. | Rev B |
