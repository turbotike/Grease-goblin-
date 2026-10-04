# Claude Code Starter Prompt — Grease Goblin Muse Gadget (Hybrid Plan)

Paste this into Claude Code running in a local clone of
https://github.com/facebookincubator/muse-gadget-sdk (the `esp32/` directory).

---

I'm porting the Muse Gadget ESP32 firmware to my board and I want your help.
Read `esp32/README.md` and `esp32/devices/AGENTS.md` first, then follow the
board-porting recipe in AGENTS.md section 6 ("Boards with the full UI").

## My hardware

- Freenove ESP32-S3 development board with 3.5" 320x480 IPS touchscreen
  (classic CYD layout: SPI LCD + touch, portrait)
- Chip: ESP32-S3 (first-class target in this SDK — good)
- Board is NOT in the upstream supported list, so this is a custom board port

## Things we must verify before writing the board file

Pull these from Freenove's docs/schematics for this exact board revision —
do NOT guess pins (the AGENTS.md recipe insists on this too):

1. LCD panel controller chip (e.g. ILI9488 or similar) and its init sequence
2. LCD SPI bus pins: SCLK, MOSI, CS, DC, RST, backlight pin
3. Touch controller chip and its pins (I2C or SPI)
4. PSRAM: size and mode (most S3 CYD-style boards have octal PSRAM —
   confirm and set CONFIG_SPIRAM_MODE_OCT accordingly)
5. BOOT button GPIO (usually GPIO0 on S3) for the pairing-confirm step
6. USB serial VID/PID for `tools/muse/ports.py`

## The port

Following the recipe, create:

1. Kconfig board entry for the new board
2. `components/muse/boards/board_freenove_cyd.c` implementing `muse_board_t`
   (init, `display_start` bringing up the 320x480 SPI panel + LVGL + touch,
   `poll_buttons`, brightness control)
3. `devices/sdkconfig.muse-freenove-cyd` (target esp32s3, flash/PSRAM sizes,
   partition table, layering on `sdkconfig.muse` for the full UI)
4. Build-script alias in `tools/muse/board.sh` + USB VID/PID in
   `tools/muse/ports.py`

Use the closest existing S3 touchscreen boards as templates
(BOX-3, SenseCAP Watcher, Waveshare S3 AMOLEDs).

## Build environment

- ESP-IDF v6.0.1 ONLY (other versions unsupported)
- Build: `tools/board.sh freenove-cyd build`, flash: `idf.py -p PORT flash monitor`
- macOS or Linux machine with a data-capable USB cable

## The hybrid plan — this is the important part

This isn't just a stock Muse gadget. It's a HYBRID:

- **Idle face:** my Grease Goblin shop scene. A little green mechanic creature
  in blue overalls runs his own routine on the screen — drinks coffee, works
  on his John Deere 325G, takes breaks, scrolls his phone. I have all the
  sprite art ready (2D flipbook-style frames, 320x480). This is the default
  state — the thing sitting on my toolbox looking alive.
- **Muse mode:** when I tap the screen or hit push-to-talk, the scene slides
  aside and the real Muse agent takes over — I can ask for torque specs,
  part numbers, diagnostic help, and answers (plus images Muse sends) show
  on screen. When I'm done, the goblin comes back and carries on with his day.

So architecturally: the SDK firmware handles the Muse connection and its
LVGL UI layer, and my sprite scene becomes the idle screen that yields to
Muse mode on interaction and returns when idle. Think "screensaver with a
day job." The SDK even ships `tools/muse/avatar.py` for custom avatars —
we'll use the goblin art there too.

Phase 1 is the board port above (get it building, flashing, and pairing).
Phase 2 is wiring the hybrid idle/Muse-mode switching. Don't try to do
both at once — nail the port first.

## Ground rules

- Explain what you're about to change before changing it
- If a step needs my hardware (plugging in the board, pressing BOOT,
  reading a screen), stop and tell me exactly what to do
- If Freenove's docs contradict an assumption, flag it immediately
- Phase 1 first. Don't build the hybrid UI until the board port works.
