# MORPHEUS (matrix-fm1)

Bare-metal firmware for the **M-VAVE FM-1** (JieLi AC791N RISC-V SoC, 240×240 LCD): an 8-track
ReBirth-style groovebox (two 303s, 808, 909, breakbeat slicer, FM lead, sub synth, MIDI aux) with a
Reason Matrix-style pattern sequencer, plus a browser emulator in `web/`. Built on the Dinghy / Felucca /
SLOOP platform. GPL-3.0. Current version 1.0.0, package identity `FM-1_8000013` (FM-1 Open platform).

## Layout

- `firmware/src/` – app code. Entry point is `morpheus_app.c`; `matrix_core.*` = sequencer (8 tracks,
  `NUM_TRACKS`), `matrix_synth.*` = engine mixing, `matrix_ui.*` = LCD renderer, `dsp/` = 303 / 808 / 909 /
  breaks / FX bus / master. `dinghy.c` is the original platform app, kept for reference.
- `firmware/hal/` – the **only** place allowed to touch hardware registers.
- `firmware/loader/`, `firmware/sdk_bins/` – update loader and JieLi SDK blobs.
- `web/` – emulator (`index.html`, `emulator.js`, `style.css`), Web MIDI flasher (`install.html`),
  user manual (`manual.html`). Static assets are mirrored into `web/assets/` from `assets/`.
- `tools/` – `build.py` (firmware build + `.fwsc` packaging), asset generators (`gen_splash.py`,
  `gen_svg_logo.py`, `gen_drum_samples.py`), `get_toolchain.sh`.
- `tests/` – host simulator (`host.c`, `run_host.sh`, `morpheus_scenario.txt`).
- `docs/RULES.md` – **read before touching firmware code** (chip constraints, summarized below).

## Commands

- Host simulator (fast, runs on macOS with plain `cc`, no toolchain needed):
  `sh tests/run_host.sh` → `build/host/morpheus.wav` and `build/host/morpheus-*.png` screenshots.
  Run it after any firmware change and look at the PNGs for UI changes.
- Firmware build: `./build.sh [--release X.Y]` → `morpheus.fwsc`. Needs the JieLi toolchain
  (`tools/get_toolchain.sh`, default `~/.jieli/toolchain`), Pillow, and Docker running on macOS.
- Web emulator: `cd web && python3 -m http.server 8000`, open http://localhost:8000.
- CI: `.github/workflows/build.yml` runs the host tests, builds firmware, and deploys `web/` to GitHub Pages
  (https://benbadio.github.io/morpheus-fm1/) on push to `main`.

## Firmware rules (from docs/RULES.md)

- **No `float`/`double`** – no FPU, link fails. Fixed point: Q15 levels, Q12 gains, 32-bit oscillator phase.
- **No 64-bit `/` or `%`** (no `__udivdi3`). 64-bit multiply is fine.
- Division by zero silently returns garbage – guard divisors.
- Precompute constants offline and write them in as numbers.
- No constant that lands in the ROM range 0xFFC00000..0xFFCFFFFF (e.g. `-32767 << 7`); the build rejects it.
- Hardware registers / volatile casts / inline asm only in `firmware/hal/`.
- `app_audio()`: 256 frames in 5.8 ms, aim for < 2.9 ms. Per-frame work only; envelopes/coeffs/LFOs per
  16–32 frame block. No flash, screen, `platform_poll()` or waiting inside it. Share state via `volatile`,
  one writer each, ordered so a half-written update is safe.
- Main loop: `platform_poll()` every few ms; redraw only what changed; flash erases only when silent.
- Crash debugging: red screen shows vector / PC / return address – look the PC up in `build/dinghy.dis`.
  Rescue mode: hold **OCT-** at power-on.

## Conventions

- Keep the firmware UI and the web emulator visually and behaviorally in sync: a UI or control-mapping change
  in `matrix_ui.c` / `morpheus_app.c` usually needs the matching change in `web/emulator.js`, and in
  `web/manual.html` / `README.md` if user-facing.
- Branding: 🐇 rabbit emoji logo, "MORPHEUS", `VERSION 1.0.0`; palette matrix green `RGB(0,255,102)`,
  amber `RGB(255,184,0)`, cyan `RGB(0,240,255)` on black.
- Keep the SPDX GPL-3.0 headers on source files.
