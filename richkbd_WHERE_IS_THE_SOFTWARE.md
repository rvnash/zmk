> Copy of `~/pcb/richkbd_WHERE_IS_THE_SOFTWARE.md`. Directory paths below are relative to `~/pcb`.
> **You are in the LEGACY firmware directory.** Do not build here — see `richkbd-zmk-config`.

# Where is the richkbd software?

Written 2026-09-04, after moving the wireless keyboard's firmware to modern ZMK. There are two
firmware directories here and it is not obvious which one is live, so: **build from
`richkbd-zmk-config`.**

## Firmware

| directory | status | GitHub |
| --- | --- | --- |
| **`richkbd-zmk-config/`** | **CURRENT.** Build here. | `rvnash/richkbd-zmk-config` (private) |
| `richkbd_wireless_software/` | Legacy. Archived, do not build. | `rvnash/zmk` (public) |

**`richkbd-zmk-config/`** is an out-of-tree ZMK config module on current ZMK (Zephyr 4.1). Build
with `./build.sh`; output is `build/zephyr/zmk.uf2`. It is not a fork of ZMK — ZMK and Zephyr are
fetched into `deps/`. Read `STATUS.md` there first: it says what works, what does not (deep sleep
is disabled), and what to do next.

**`richkbd_wireless_software/`** is the 2023–2026 firmware: a full fork of ZMK on Zephyr 3.2,
built in Docker against a patched `zephyr/` checkout. Kept for two reasons — `firmware/` holds
flashable known-good uf2 files if the current firmware ever needs rolling back, and
`MIGRATION_TO_MODERN_ZMK.md` holds the migration record and the battery baseline log. Its build
instructions still work (`HOW_TO_BUILD_FROM_CLI.md`) but there is no reason to use them.

There is also a **zephyr fork** with no local clone: `rvnash/zephyr`, branch `richkbd-odr`. It is
`v4.1.0+zmk-fixes` plus three commits to the mcp23xxx GPIO driver that this keyboard needs.
`config/west.yml` in the config repo points at it. When bumping ZMK, that branch must be rebased
onto whatever zephyr revision ZMK then pins — see `STATUS.md`.

## Hardware

| directory | what | GitHub |
| --- | --- | --- |
| `richkbd_wireless/` | KiCad design for the wireless board (XIAO nRF52840 + 3× MCP23017). The one the firmware above runs on. | `rvnash/richkbd_wireless` |
| `richkbd/` | The earlier wired richkbd 2.0 — RP2040 Stamp, Neopixels, OLED. Different keyboard, different firmware (QMK). | `rvnash/richkbd` |

The wireless board's fab netlist lives at
`richkbd_wireless/pcb/production/Rich_Keyboard_2.0_2023-07-27_22-08-39/netlist.ipc`. It settled
several firmware questions during the migration and is the authority on how the expanders and the
shared interrupt line are wired. `STATUS.md` lists hardware issues found in it that are worth
fixing on a future board revision.

## Unrelated projects in this directory

`pi2040_tracker/` and `tracker-7/` have nothing to do with the keyboards.

## Flashing, wherever the uf2 came from

Plug in **directly, not through a USB hub** — through a hub the bootloader fails to enumerate in
time and boots straight back into the firmware. Hold the middle left thumb key and tap the
top-left key to enter the bootloader (the XIAO's LED fades in and out when it is there), then
`cp -X <file>.uf2 /Volumes/XIAO-SENSE`. An `Input/output error` from `cp` is success;
`Operation not permitted` means the drive is not mounted.
