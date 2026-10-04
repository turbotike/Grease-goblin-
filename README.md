# Grease Goblin Gadget

Hybrid ESP32-S3 project: a Grease Goblin shop buddy living on a Freenove
CYD touchscreen, powered by Meta's open-source Muse gadget firmware.

## The idea

Most of the day, the goblin runs his own shop routine on the screen —
coffee, wrenching on the John Deere 325G, breaks, the works. Tap the screen
or hit push-to-talk and the real Muse agent takes over: torque specs, part
number lookups, diagnostic help. Done asking, the goblin comes back.

## Files

- `gadget-sdk-brief.md` — technical brief on the Muse gadget ESP32 SDK:
  what it does, S3 support, display/pairing/build details, and an honest
  assessment of porting it to the Freenove CYD.
- `claude-code-gadget-starter.md` — ready-to-paste prompt for Claude Code:
  clone the SDK repo, run Claude Code in `esp32/`, paste this in, and it
  walks through the custom board port (phase 1) then the hybrid idle/Muse
  UI (phase 2).

## Hardware

- Freenove ESP32-S3 CYD, 3.5" 320x480 IPS touchscreen (portrait)
- SDK: https://github.com/facebookincubator/muse-gadget-sdk (Apache 2.0)
- Build: ESP-IDF v6.0.1 only, macOS or Linux, USB flashing

## Sprite art

The goblin character art and shop scene assets live in the project's
Google Drive folder ("Grease Goblin") — they get wired in during phase 2
via the SDK's custom avatar flow (`tools/muse/avatar.py`).
