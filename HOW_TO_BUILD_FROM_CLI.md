# How to build richkbd firmware from the command line

VSCode is **not** required. VSCode was only ever a convenience wrapper: `.vscode/tasks.json`
just runs `west build`, and `.devcontainer/` just starts the ZMK toolchain container. You can do
both yourself from a terminal.

There are two ways to drive the build, and they produce the same `build/zephyr/zmk.uf2`:

- [Option A: one-shot build](#option-a-one-shot-build) — a single `docker run` per build. Best
  for a quick rebuild after a keymap edit.
- [Option B: interactive shell in the container](#option-b-interactive-shell-in-the-container) —
  start a shell once, then run `west` commands inside it as many times as you like. This is
  exactly what the VSCode devcontainer gave you.

Flashing happens from the Mac side in either case (plain `cp` to the mounted bootloader drive).

Before your first CLI build, apply the [required zephyr patch](#required-patch-to-the-zephyr-tree)
— the build will not compile without it.

---

## Facts about this workspace

- Board: `seeeduino_xiao_ble` (XIAO nRF52840), shield: `richkbd`
- Zephyr v3.2.0+zmk-fixes; the container image pins Zephyr SDK 0.15.2
- The west workspace is already initialized (`.west/config`) and `zephyr/` + `modules/` are
  already checked out on disk, so **no `west init` / `west update` is needed** unless you
  intentionally re-sync (`west update` in the repo root). Re-syncing discards the zephyr patch —
  see [Required patch to the zephyr tree](#required-patch-to-the-zephyr-tree).
- `zephyr/` therefore carries one local modification on purpose. `git -C zephyr status` showing
  `M drivers/gpio/gpio_mcp230xx.c` is the *correct* state, not something to clean up.
- Keymap lives at `app/boards/shields/richkbd/richkbd.keymap`; config at `richkbd.conf`.

---

## Required patch to the zephyr tree

`patches/0001-mcp230xx-init-registers.patch` **must be applied to `zephyr/`** before richkbd will
build. Apply it from the repository root:

```sh
git -C zephyr apply "$PWD/patches/0001-mcp230xx-init-registers.patch"
```

Check whether it is already applied (exit status 0 means yes, it is):

```sh
git -C zephyr apply --check --reverse "$PWD/patches/0001-mcp230xx-init-registers.patch"
```

Re-apply it after **any `west update`**, which resets `zephyr/` to `manifest-rev` and silently
throws the patch away.

### What it does, and why it is not optional

The patch is Richard's, originally committed directly inside the zephyr checkout on 2023-08-10
("Updating mcp230xx code to initialize the registers"). It changes
`zephyr/drivers/gpio/gpio_mcp230xx.c` in two ways:

1. **It makes the file compile with more than one expander.** Upstream `v3.2.0+zmk-fixes` writes
   `mcp230xx_##inst##_drvdata` inside `GPIO_MCP230XX_DEVICE(n)`, whose parameter is `n` — so
   `inst` is never substituted and every instance emits the *same* literal symbol name. richkbd
   has three mcp23017s, so the second one collides. The symptom:

   ```
   gpio_mcp230xx.c:81:41: error: redefinition of 'mcp230xx_inst_drvdata'
   ```

   The patch renames the symbols to `mcp230xx_drvdata_##n` / `mcp230xx_config_##n`.

2. **It initializes the expanders for this keyboard.** It writes the register cache out to the
   chip in `mcp230xx_bus_is_ready()` and sets the defaults richkbd needs: `iodir = 0x7F7F`,
   `gpinten = 0xFFFF`, `iocon = 0b0100010001000100` (interrupt-on-change, mirrored INT).

Part 1 alone is what unblocks the compile, so resist the temptation to "just fix the typo" —
without part 2 you get firmware whose expanders are never configured.

`gpio_mcp23sxx.c` still carries the identical upstream `##inst##` bug. It is harmless here: SPI is
disabled in `richkbd.overlay`, so that driver is never built.

### Why this never came up under VSCode

`.devcontainer/devcontainer.json` mounts Docker *named volumes* over three directories:

| volume | mounted over |
| --- | --- |
| `zmk-zephyr` | `zephyr/` |
| `zmk-zephyr-modules` | `modules/` |
| `zmk-zephyr-tools` | `tools/` |

Every devcontainer build therefore compiled the zephyr tree inside the `zmk-zephyr` volume — where
that patch was committed — and the `zephyr/` directory in this repo was shadowed and never built.
The CLI recipes below bind-mount only the repo, with no volume overlay, so they use the real
`zephyr/` on disk. That is the whole reason the patch has to be applied explicitly now.

(The volume copy also sits on an older `zmk-fixes` revision — `6f349d0e4`, Jul 2023 — than this
checkout, which is at `1ae0eb5ce8`, Nov 2023. Both are depth-1 clones and share no history, so CLI
builds are not bit-for-bit identical to the last devcontainer-built firmware.)

---

## Option A: one-shot build

Run from the repository root (`/Users/rich/pcb/richkbd_wireless_software`):

```sh
docker run --rm -it \
  -v "$PWD":/workspaces/richkbd_wireless_software \
  -w /workspaces/richkbd_wireless_software \
  docker.io/zmkfirmware/zmk-build-arm:3.2 \
  west build -p -b seeeduino_xiao_ble -d build app -- -DSHIELD=richkbd
```

Result: `build/zephyr/zmk.uf2`.

`-p` is a pristine (clean) build. For a fast incremental rebuild after a keymap edit, drop it
along with the board and shield arguments — they are remembered in the build directory:

```sh
docker run --rm -it \
  -v "$PWD":/workspaces/richkbd_wireless_software \
  -w /workspaces/richkbd_wireless_software \
  docker.io/zmkfirmware/zmk-build-arm:3.2 \
  west build -d build app
```

---

## Option B: interactive shell in the container

Useful when you are iterating: you pay the container startup once, and each rebuild is just a
`west build` at the prompt.

**1. Start the shell.** From the repository root:

```sh
docker run --rm -it \
  -v "$PWD":/workspaces/richkbd_wireless_software \
  -w /workspaces/richkbd_wireless_software \
  docker.io/zmkfirmware/zmk-build-arm:3.2 \
  bash
```

You land at a root prompt inside `/workspaces/richkbd_wireless_software`, which is your repo
checkout — the same files, live. Confirm with `ls` if you like; you should see `app/`, `zephyr/`,
`modules/`.

**2. Build (pristine).** At the container prompt:

```sh
west build -p -b seeeduino_xiao_ble -d build app -- -DSHIELD=richkbd
```

**3. Rebuild (incremental).** After editing the keymap or config *on the Mac side* — the edit is
visible in the container immediately, no restart needed — just run:

```sh
west build -d build app
```

Repeat step 3 as often as you want. Only add `-p` back when you change something CMake needs to
reconfigure for (Kconfig options, new source files, a different shield).

**4. Leave the shell.**

```sh
exit
```

`--rm` throws the container away on exit, but nothing is lost: `build/` lives in the mounted repo,
so `build/zephyr/zmk.uf2` is sitting there on the Mac, ready to flash. Starting a fresh shell
later picks up the same build directory and still rebuilds incrementally.

---

## Notes that apply to both options

- **Mount at `/workspaces/richkbd_wireless_software` specifically.** The existing `build/`
  directory contains absolute paths from the old devcontainer (`/workspaces/...`). Using the same
  path means CMake's cache stays valid; a different path would force a reconfigure.
- This Mac is Apple Silicon and the ZMK images are x86_64. If Docker reports no matching
  manifest, add `--platform linux/amd64`; it will work but run slower under emulation (the
  devcontainer had the same cost).
- The image used by CI (`.github/workflows/build.yml`) is `zmk-build-arm:3.2`; the devcontainer
  used `zmk-dev-arm:3.2`. Either builds this firmware; `dev` also carries editor/debug extras.
- If CMake ever complains that it cannot find the Zephyr package, run `west zephyr-export` (in the
  container shell, or in place of `west build` in a one-shot run) before building.

---

## Flashing

1. Put the keyboard into bootloader mode. If the firmware is working: press **Nav +
   top-left-most key**. Otherwise double-tap the XIAO's reset button.
2. It mounts as `/Volumes/XIAO-SENSE`.
3. Copy the firmware — from the Mac, not from inside the container:

   ```sh
   cp -X build/zephyr/zmk.uf2 /Volumes/XIAO-SENSE
   ```

`cp` usually reports `cp: /Volumes/XIAO-SENSE/zmk.uf2: fcopyfile failed: Input/output error`.
That is expected and harmless — the bootloader reboots the board the moment the file lands, which
tears the volume out from under `cp`. The board flashing and rebooting is the real success signal.
