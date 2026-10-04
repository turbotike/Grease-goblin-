# Muse Gadget SDK — Technical Brief for the Freenove ESP32-S3 CYD
*Researched 2026-10-03 from the public repo docs. Repo: https://github.com/facebookincubator/muse-gadget-sdk*

## 1. What the ESP32 SDK actually does

The ESP32 Device SDK is firmware that turns an ESP32 board into a **peripheral for the cloud Muse agent** — the AI itself does not run on the board. Per the docs, every board "pairs with the Muse app, joins your Wi-Fi, and holds an encrypted session to Muse" (source: https://github.com/facebookincubator/muse-gadget-sdk/blob/main/esp32/devices/README.md).

How the device talks to Muse:
- The board connects to home Wi-Fi and maintains an encrypted session to Meta's cloud, where the Muse agent runs.
- Boards **with PSRAM** also get a **home-network tunnel**: "Muse can reach the devices you already own and anything you build with a local HTTP API" (source: https://github.com/facebookincubator/muse-gadget-sdk/tree/main/esp32).
- Boards **without PSRAM** (classic ESP32, ESP32-C6 in the list) run without the tunnel, but "Muse can still reach and control them once the control session is up."
- Replies from Muse are **text** by default. Push-to-talk sends a voice note, Muse transcribes it and answers in writing; boards with a screen show the answer as captions. Spoken replies are DIY: "Send each reply's text to a text-to-speech API of your choice" — the hook point is `start_tts` in `components/muse/muse_chat_session.cpp`, where "the MP3 decoder, speaker and volume are already wired up."
- Custom avatar art is explicitly supported: "To put your own avatar on a board's screen, plug in the board and run `python3 tools/muse/avatar.py`. It asks your Muse to redraw its avatar as the board's pixel avatar, checks the result, then builds and flashes it."

## 2. Supported ESP32 variants — is ESP32-S3 explicitly supported?

**Yes.** The ESP32-S3 is a first-class target. The documented build installs the toolchain for `esp32c5, esp32s3, esp32c6, esp32` (`install.sh esp32c5,esp32s3,esp32c6,esp32`). Supported S3 boards upstream include: Seeed SenseCAP Indicator, Seeed reTerminal E1001, Home Assistant Voice Preview Edition, Waveshare ESP32-S3-Touch-AMOLED-1.75C / 1.75, Espressif ESP32-S3-BOX-3, AIPI Lite, Seeed SenseCAP Watcher, M5Stack Cardputer ADV (experimental), M5Stack StickS3, M5Stack StopWatch.

The **Freenove ESP32-S3 CYD is NOT in the supported list** — it would need a custom board port (see §7). Note: an unofficial community fork (https://github.com/ledienbien-ai/muse-gadget-sdk) already adds ESP32-S3 display boards the upstream SDK doesn't cover yet, which proves the porting path works but also that upstream S3 display coverage is still growing.

## 3. Display support

- The full on-screen UI is **LVGL-based**: "an animated avatar, push-to-talk and settings." A desktop simulator runs "the production UI and avatar renderer" using LVGL + SDL (see `esp32/simulator/`).
- "Screens showing images" technically means: **boards that support images can show pictures Muse sends them** via a `display.draw_url` mechanism; `tools/image_for_display.py` prepares a picture for the screen size. Caveat from the docs: "the UI holds a whole image in PSRAM" — so image display effectively needs PSRAM (the one exception, the ideaspark board, draws straight to its screen without PSRAM).
- Display backends are per-board. Two integration paths are documented in `esp32/devices/AGENTS.md`:
  - **Status-screen kind**: a display backend in `main/` (see `main/led_status.c`) driving the panel through Espressif's `esp_lcd` component — panels on SPI/QSPI/RGB/I80 buses with pins, resolution, and backlight defined per board.
  - **Full-UI kind**: a `muse_board_t` implementation (`components/muse/boards/board_<id>.c`) whose `display_start` brings up the panel, LVGL and its task, plus touch input.
- **320×480 SPI touchscreen plausibility: yes, plausible.** A 320×480 SPI LCD (the classic CYD panel type) is exactly the kind of panel `esp_lcd` drives, and LVGL works on top of it. Nothing in the docs rules it out. The work is writing the board's display init (panel controller, SPI pins, backlight, touch controller) following the existing board files as templates.

## 4. Pairing flow

1. Get an SDK token from **gadgets.muse.ai** (Account > SDK tokens). "Every gadget needs a token to pair, including ones you build for yourself." Read the Gadget SDK Terms first.
2. Bake the token into firmware: `idf.py menuconfig` → "ESP32 Device SDK" → "Muse Gadgets SDK token". Note: "Your SDK token ships inside the firmware, so treat it as an identifier rather than a password. If it leaks, revoke it on gadgets.muse.ai, generate a new one, and rebuild."
3. Flash, then in the Muse app: **Settings > Devices → turn on Developer mode → Add Device** (+ icon, top right). The device advertises as `MuseGadget-XXXXXX` (BLE).
4. Status light breathes **orange** = ready; during pairing it breathes **blue** = press the BOOT button on the device to confirm; **green** = connected to Muse. (On a custom CYD port, the button/light roles must be mapped to the board's actual GPIO.)
5. "Pairing requires a press of the button on the device, and every setup creates a fresh encrypted session. Because these are community devices, pairing has no manufacturer verification and can't prevent an active man-in-the-middle attack. **Set it up on a network you trust.**"

## 5. Build system and flashing requirements

- **ESP-IDF v6.0.1, and only v6.0.1** — "Other versions aren't supported."
- Build with `idf.py build` (default target: ESP32-C5 DevKitC-1); per-board builds via `tools/board.sh <board> build`, which builds in a per-board `build-<board>` directory with the right chip and settings. Flash with `idf.py -p PORT flash monitor` (or `tools/board.sh <board> flash`).
- **Computer must run macOS or Linux.** The docs list prerequisites for macOS (`brew install cmake ninja dfu-util python3`) and Linux; **Windows is not mentioned** — flagging as unverified, not confirmed unsupported.
- You need a USB cable that carries data, the board's USB serial port, and (for the agentic flow) Muse Code: `curl -fsSL https://dev.meta.ai/install.sh | sh`, then run `muse --disable-sandbox` from `esp32/` so it can reach USB serial and download the ESP-IDF toolchain. Nat Friedman's suggested flow is literally "point your favorite coding agent at the GitHub repo."
- Each board is an `sdkconfig` overlay (`devices/sdkconfig.<board>`) on top of `sdkconfig.defaults`. Boards needing the full UI layer an additional overlay on `sdkconfig.muse` (e.g. `devices/sdkconfig.muse-<name>`).
- OTA updates are off by default (on for full-UI boards); builds are version `999.0.0`; builds are signed with a dev key, Secure Boot stays off, so you can reflash freely.

## 6. Stated limitations, security notes, caveats

- "Built by hackers, for hackers, just for fun. **Side effects of tinkering may include bricked boards, voided warranties, brownouts, or bankruptcies. Proceed at your own risk!**" (Both the root and ESP32 READMEs carry this.)
- Pairing has **no manufacturer verification** and can't stop an active MITM — use a trusted network.
- SDK token is baked into firmware — treat as identifier, revoke/regenerate on leak.
- **"We strongly recommend enabling NVS encryption** if your board supports it. NVS stores Wi-Fi credentials and device tokens in flash; without encryption, anyone with physical access to the board can read them." (`CONFIG_HOMEHUB_NVS_ENCRYPTION` in menuconfig.)
- No PSRAM → no home-network tunnel, no received images, voice notes get text-only replies.
- Replies are text unless you wire up TTS yourself.
- The Apache 2.0 license **does not cover the Jollybot avatar**.
- It is brand-new (repo created 2026-10-02, 9 commits at time of research) — expect rough edges; a community issue already documents pairing/mic breakage on an unofficial port (https://github.com/facebookincubator/muse-gadget-sdk/issues/9).

## 7. Honest assessment: adapting this to the Freenove ESP32-S3 CYD

**Verdict: very feasible, but it's a real porting project — not a flash-and-go.** The chip (ESP32-S3) is fully supported; the board is not. Concretely, the work is:

1. **Board port** following `esp32/devices/AGENTS.md` §6 ("Boards with the full UI"): add a Kconfig board entry, write `components/muse/boards/board_freenove_cyd.c` implementing `muse_board_t` (init, `display_start` for the 320×480 SPI panel + LVGL + touch, `poll_buttons`, brightness), write `devices/sdkconfig.muse-freenove-cyd` (target `esp32s3`, flash/PSRAM sizes, partition table), add a `tools/muse/board.sh` alias and USB VID/PID mapping in `tools/muse/ports.py`. The AGENTS.md recipe is explicit and the closest templates are the other S3 touchscreen boards (BOX-3, Watcher, Waveshare S3).
2. **Display bring-up** is the crux: the CYD's LCD controller, SPI pins, backlight pin, and touch controller must be wired into the board file. The docs insist: gather these from the vendor schematic/docs and "don't guess pins." (⚠️ Not verified in this research: the exact panel controller/touch chip/pinout of the Freenove 3.5" board — pull from Freenove's docs/repo before starting.)
3. **PSRAM check** (⚠️ also to verify against Freenove's specs): if the board has PSRAM, the full UI + images from Muse + home-network tunnel all work. Without PSRAM, you'd still get the UI avatar but lose received images and the tunnel. Most ESP32-S3 CYD-style boards ship with octal PSRAM — confirm, and set `CONFIG_SPIRAM_MODE_OCT=y` if so.
4. **Pairing UX**: map the "press BOOT to confirm" step to a real button/GPIO on the CYD (BOOT is GPIO0 on S3 — the recipe calls this out).

**Effort estimate:** for a builder comfortable with ESP-IDF and LVGL (or with Claude Code doing the port following AGENTS.md), this is on the order of days, not weeks — the recipe, templates, and simulator (`esp32/simulator/` runs the production UI on desktop for development without a board) all exist. The biggest risk is display/touch driver details for the exact Freenove panel revision.

**Strategic note for the Grease Goblin project:** this SDK changes the firmware plan. Instead of a purely scripted Tamagotchi (the current Claude-prompt plan), the CYD could run the actual Muse agent with the goblin as its on-screen avatar — Meta even ships a `tools/muse/avatar.py` flow for putting a custom avatar on the board. The scripted-sprite state machine could become a fallback or a "personality layer" on top. Worth deciding the architecture before writing firmware: full Muse gadget vs. scripted sprites vs. hybrid.

---
*Sources: repo root (https://github.com/facebookincubator/muse-gadget-sdk), ESP32 README (https://github.com/facebookincubator/muse-gadget-sdk/tree/main/esp32), devices README (https://github.com/facebookincubator/muse-gadget-sdk/blob/main/esp32/devices/README.md), devices AGENTS.md (https://github.com/facebookincubator/muse-gadget-sdk/blob/main/esp32/devices/AGENTS.md), community S3 fork (https://github.com/ledienbien-ai/muse-gadget-sdk), issue #9 (https://github.com/facebookincubator/muse-gadget-sdk/issues/9). Gadget portal: gadgets.muse.ai (referenced in-repo; portal itself not browsed).*
