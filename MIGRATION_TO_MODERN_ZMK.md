# Migration plan: modern ZMK, no fork, (almost) no patches

**Status:** not started. Captured 2026-09-03. Nothing in this document has been executed.

**Goal:** move richkbd from this 2023-era ZMK fork (Zephyr 3.2) to current ZMK (Zephyr 4.1) as an
out-of-tree config module, retiring both local patches and the Docker-volume toolchain.

**Do not start this until the current CLI build is flashed and confirmed working**, so there is a
known-good baseline to fall back to.

---

## 1. Why bother

Today richkbd carries **two** local patches to code we do not own, and only one of them was ever
in git:

| patch | where it lives | in git? |
| --- | --- | --- |
| mcp230xx register init + macro fix | `patches/0001-mcp230xx-init-registers.patch`, applied to the gitignored `zephyr/` tree | now yes (as a patch file) |
| `interrupt-gpios` added to `zmk,kscan-gpio-direct` | `app/drivers/kscan/kscan_gpio_direct.c` + `app/drivers/zephyr/dts/bindings/kscan/zmk,kscan-gpio-direct.yaml` | yes |

Both exist for one reason: **the Zephyr 3.2 mcp23xxx driver cannot do interrupts.**
`mcp23xxx_pin_interrupt_configure()` is `return -ENOTSUP;`. So the chip had to be configured by
hand, and ZMK's kscan had to be taught to accept a single board-level interrupt line instead of
per-pin interrupts.

Zephyr **v3.4.0** added full interrupt support to that driver, and ZMK now ships Zephyr 4.1. On
modern ZMK:

- `pin_interrupt_configure()` sets `GPINTEN`/`INTCON`/`DEFVAL` per pin, on demand
- `IOCON.MIRROR` is set automatically at init when `int-gpios` is present and `ngpios == 16`
- a `k_work` handler reads `INTF`/`GPIO` and dispatches per-pin callbacks
- stock `zmk,kscan-gpio-direct` calls `gpio_pin_interrupt_configure_dt(gpio, GPIO_INT_LEVEL_ACTIVE)`
  on every input pin — exactly the call our driver refuses today

So both patches become unnecessary. What remains is **one bit** (see §3), and it is forced by the
PCB, not by software.

Secondary benefits: builds move to GitHub Actions (no Docker, no toolchain volumes), the ZMK fork
goes away entirely, and current ZMK features (Studio, modern behaviors, active support) become
available.

**Cost:** a weekend, and re-validating a keyboard that currently works. This is optional work.

---

## 2. Verified version facts

Established 2026-09-03 by reading the actual trees, not from memory. Re-verification commands are
in the appendix.

| fact | value |
| --- | --- |
| mcp23xxx interrupts absent | upstream Zephyr `v3.2.0`, `v3.3.0` |
| mcp23xxx interrupts present | upstream Zephyr **`v3.4.0`** and later |
| ZMK `main` zephyr pin | `zmkfirmware/zephyr` @ `v4.1.0+zmk-fixes` |
| board target (HWMv2) | **`xiao_ble/nrf52840/zmk`** (was `seeeduino_xiao_ble`) |
| board definition | `app/boards/seeed/xiao_ble/`, `board.yml` extends upstream `xiao_ble`, variant `zmk` |
| connector labels still valid | `&xiao_d`, `&xiao_i2c`, `&xiao_spi`, `&xiao_serial` (upstream `boards/seeed/xiao_ble/seeed_xiao_connector.dtsi`) |
| generic `microchip,mcp230xx` compatible | **gone** — replaced by per-model `microchip,mcp23017`, `mcp23018`, … |
| new DT properties on the expander | `int-gpios`, `reset-gpios` |
| `##inst##` compile bug | gone; macro is now `GPIO_MCP230XX_DEVICE(inst, num_gpios, open_drain, model)` |
| kscan `interrupt-gpios` | does not exist upstream (it was ours); modern kscan uses per-pin interrupts |
| debounce config | now DT properties `debounce-press-ms` / `debounce-release-ms`; the Kconfig globals still exist as overrides (default `-1` = defer to DT) |
| `CONFIG_ZMK_KSCAN_DIRECT_POLLING` | still exists (`app/module/drivers/kscan/Kconfig`) |
| `CONFIG_ZMK_KEYBOARD_NAME`, `ZMK_IDLE_TIMEOUT`, `ZMK_SLEEP`, `ZMK_IDLE_SLEEP_TIMEOUT` | all still exist |
| `matrix_transform.h` | still at `dt-bindings/zmk/matrix_transform.h` |

---

## 3. The one thing that does not migrate: open-drain INT

**This is the crux of the plan. Read it before anything else.**

From the fabrication netlist
(`richkbd_wireless/pcb/production/Rich_Keyboard_2.0_2023-07-27_22-08-39/netlist.ipc`), the net
named `INT` has these members:

```
U1-15, U1-16    INTB, INTA of expander 1   (MCP23017, QFN-28)
U2-15, U2-16    INTB, INTA of expander 2
U3-15, U3-16    INTB, INTA of expander 3
U4-1            XIAO D0
```

**Six interrupt pins on one wire, and no pull-up resistor on the net.** The pull-up is the
nRF52840's internal one, which is why the overlay says
`interrupt-gpios = <&xiao_d 0 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>`.

For that to work at all, every one of those six pins must be **open-drain**. That is
`IOCON.ODR` (bit 2) — one of the two bits in our `iocon = 0x4444`. Note the INTA/INTB pairs are
physically tied *within* each chip too, so without open-drain a single chip's two INT pins fight
each other, never mind across chips.

The modern driver writes **`write_iocon(dev, REG_IOCON_MIRROR)` — bit 6 only.** There is no `ODR`
define in `gpio_mcp23xxx.h`, no `ODR` write anywhere in the driver, and no binding property for
it. The `open_drain` argument in the device macro is a per-model constant (`false` for mcp23017,
`true` for mcp23018) used **only to validate pin configuration flags** — it never touches INT
behavior.

So modern ZMK on this hardware needs `IOCON.ODR` set by something. Three ways:

### Option A — one-commit fork of `zmkfirmware/zephyr`, pinned in `west.yml` *(recommended)*

Add to `gpio_mcp23xxx.h`:

```c
#define REG_IOCON_ODR BIT(2)
```

and in `gpio_mcp23xxx.c`, at the INT setup block (~line 531 of the 4.1 file):

```c
-		err = write_iocon(dev, REG_IOCON_MIRROR);
+		err = write_iocon(dev, REG_IOCON_MIRROR | REG_IOCON_ODR);
```

Then point the config repo's `config/west.yml` at your fork's branch instead of
`v4.1.0+zmk-fixes`. Fifty-five lines of patch become one, it lives in a real git remote that GHA
can fetch, and nothing is hidden in a Docker volume. The downside is maintaining a zephyr fork
across ZMK upgrades — but it is a one-line rebase.

Unconditionally setting `ODR` is safe for anyone with a pull-up, which is every MIRROR user; it is
not obviously wrong upstream either, which leads to Option C.

### Option B — set it from the module, no fork

A ~30-line C file in the config module with a `SYS_INIT` at an init priority *after*
`CONFIG_GPIO_MCP230XX_INIT_PRIORITY` (75), doing a raw `i2c_reg_write_byte` of `0x44` to `IOCON`
on `0x20`, `0x21`, `0x22`.

Avoids forking zephyr entirely. Two caveats: it writes behind the driver's back, so
`drv_data->reg_cache.iocon` is stale (harmless in practice — `write_iocon` is only ever called at
init); and it duplicates the chip addresses outside the devicetree.

### Option C — upstream a binding property

Add `int-open-drain` (or reuse `GPIO_LINE_OPEN_DRAIN` semantics on the `int-gpios` flags) to
`microchip,mcp23xxx.yaml` and honor it in `write_iocon`. Correct, benefits others, and removes the
fork permanently. Slow, and doesn't help until it lands in a ZMK-pinned Zephyr.

**Plan:** do **A** to get running, then **C** at leisure, then drop the fork.

### Also not migrating: `iodir = 0x7F7F`

Our patch makes GPA7/GPB7 outputs on each chip — the two pins per expander that aren't in
`input-gpios` — so unused inputs don't float. There is no upstream equivalent.

With per-pin interrupt configuration this matters much less: the modern driver only sets `GPINTEN`
for pins ZMK actually configures, so a floating unused pin can no longer generate interrupts. What
remains is a small leakage current on 6 floating pins. **Measure before adding anything.** If it
does matter, the fix belongs in Option A/B alongside the `ODR` write, or as a `gpio-hog`-style
output declaration in the overlay.

---

## 4. Target structure

Fork nothing. Use a config module based on
[`zmkfirmware/unified-zmk-config-template`](https://github.com/zmkfirmware/unified-zmk-config-template),
whose layout is:

```
richkbd-zmk-config/            ← new repo (or reuse this one, see §5 step 1)
├── .github/workflows/build.yml   stock ZMK GHA workflow
├── build.yaml                    board+shield matrix
├── config/
│   └── west.yml                  pins zmk (and our zephyr fork, per §3 Option A)
├── boards/shields/richkbd/
│   ├── Kconfig.shield
│   ├── Kconfig.defconfig
│   ├── richkbd.overlay
│   ├── richkbd.conf
│   ├── richkbd.keymap
│   └── richkbd.zmk.yml
└── zephyr/module.yml             build.settings.board_root: .
```

`build.yaml`:

```yaml
include:
  - board: xiao_ble/nrf52840/zmk
    shield: richkbd
```

`zephyr/module.yml`:

```yaml
build:
  settings:
    board_root: .
```

Firmware then comes from GHA artifacts. Local Docker builds remain possible (see
`HOW_TO_BUILD_FROM_CLI.md`) but stop being the only way.

---

## 5. Steps

### Step 0 — baseline and inventory

1. Flash and confirm today's CLI-built firmware. Do not start from a broken keyboard.
2. Archive a known-good uf2 **outside** `build/` (see the lesson in `HOW_TO_BUILD_FROM_CLI.md`).
3. Record current behaviour to compare against later: idle current draw, wake-from-sleep latency,
   any missed-keypress behaviour under fast typing.

### Step 1 — create the config module

Generate a repo from `unified-zmk-config-template`. Decide whether this repo becomes it (rename,
delete the ZMK fork content) or a fresh repo is cleaner. **Fresh repo is cleaner**; keep this one
archived as the historical fork — it is the only record of the 2023 bring-up.

### Step 2 — move the shield, fix the naming bugs

Copy `app/boards/shields/richkbd/*` to `boards/shields/richkbd/`, then fix two latent bugs found
while writing this plan:

- `Kconfig.shield` says `config SHIELD_MY_BOARD` / `def_bool $(shields_list_contains,my_board)` —
  never renamed from the template. Since we build `-DSHIELD=richkbd`, this symbol is always `n`.
- Consequently `Kconfig.defconfig`'s `if SHIELD_MY_BOARD` block never applies, which is why
  `richkbd.conf` sets `CONFIG_ZMK_KEYBOARD_NAME` explicitly.

Correct versions:

```kconfig
# Kconfig.shield
config SHIELD_RICHKBD
    def_bool $(shields_list_contains,richkbd)
```

```kconfig
# Kconfig.defconfig
if SHIELD_RICHKBD

config ZMK_KEYBOARD_NAME
    default "richkbd"

endif
```

Then drop `CONFIG_ZMK_KEYBOARD_NAME` from `richkbd.conf`.

Also leave behind, deliberately: `app/boards/arm/seeeduino_xiao/seeeduino_xiao.overlay` (an empty
file we added), the whitespace-only change to `app/boards/seeeduino_xiao_ble.conf`, and
`richkbd_prethumb shift.keymap` (an old keymap backup — move it to a `keymaps-archive/` folder if
it's worth keeping).

### Step 3 — rewrite the overlay

The `input-gpios` list of 42 entries and the whole `matrix_transform` block **carry over
verbatim**. What changes is the expander declarations and the removal of our custom kscan
property:

```dts
&xiao_i2c {
    status = "okay";
    clock-frequency = <I2C_BITRATE_FAST>;

    io_0: mcp23017@20 {
        compatible = "microchip,mcp23017";      /* was "microchip,mcp230xx" */
        status = "okay";
        gpio-controller;
        reg = <0x20>;
        #gpio-cells = <2>;
        int-gpios = <&xiao_d 0 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;   /* NEW: shared INT net */
        /* ngpios dropped — 16 is implied by the model */
    };

    io_1: mcp23017@21 { /* ... same, reg = <0x21> ... */ };
    io_2: mcp23017@22 { /* ... same, reg = <0x22> ... */ };
};

kscan0: kscan {
    compatible = "zmk,kscan-gpio-direct";
    /* interrupt-gpios REMOVED — that property was our own patch */
    debounce-press-ms = <0>;      /* was CONFIG_ZMK_KSCAN_DEBOUNCE_PRESS_MS=0   */
    debounce-release-ms = <5>;    /* was CONFIG_ZMK_KSCAN_DEBOUNCE_RELEASE_MS=5 */
    input-gpios = < ...unchanged 42 entries... >;
};
```

Keep `&xiao_spi { status = "disabled"; }`. Note ZMK's modern `xiao_ble_zmk.dts` already disables
`&xiao_serial` and **enables `&qspi`** with the P25Q16H flash — our overlay currently disables
`&qspi` because it prevented sleep. Keep `&qspi { status = "disabled"; }` initially, then test
whether it's still needed; ZMK may handle QSPI power-down properly now.

### Step 4 — translate `richkbd.conf`

| current line | action |
| --- | --- |
| `CONFIG_ZMK_KEYBOARD_NAME="richkbd"` | drop — moves to `Kconfig.defconfig` (step 2) |
| `CONFIG_I2C=y` | keep |
| `CONFIG_ZMK_KSCAN_DEBOUNCE_PRESS_MS=0` | drop — becomes `debounce-press-ms` in DT |
| `CONFIG_ZMK_KSCAN_DEBOUNCE_RELEASE_MS=5` | drop — becomes `debounce-release-ms` in DT |
| `CONFIG_BT=y` | keep (likely redundant, ZMK defaults it) |
| `CONFIG_ZMK_IDLE_TIMEOUT=1` | keep — **but verify**, `1` ms is suspicious; probably meant to be aggressive, may be a typo for `1000` |
| `CONFIG_ZMK_SLEEP=y`, `CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=900000` | keep |
| `CONFIG_ZMK_KSCAN_DIRECT_POLLING=n` | keep as an explicit statement of intent (still a valid symbol) |
| `CONFIG_ZMK_USB=n`, `CONFIG_ZMK_USB_LOGGING=n` | keep |
| `CONFIG_MPU_ALLOW_FLASH_WRITE`, `CONFIG_NVS`, `CONFIG_SETTINGS_NVS`, `CONFIG_FLASH*` | try removing — modern ZMK sets these for nRF52840 by default; re-add only if the build or settings persistence breaks |

### Step 5 — apply the `ODR` fix

Per §3 Option A: fork `zmkfirmware/zephyr` at `v4.1.0+zmk-fixes`, make the one-line change, push a
branch, and point `config/west.yml` at it.

### Step 6 — build

Push and let GHA build. For local iteration, the Docker recipes in `HOW_TO_BUILD_FROM_CLI.md`
still apply with the new board target:

```sh
west build -p -b xiao_ble/nrf52840/zmk -d build config -- -DSHIELD=richkbd
```

(Exact `-d`/source-dir arguments differ for a config-module layout — check ZMK's current
local-toolchain docs at build time.)

### Step 7 — validate on hardware

This is the part that actually costs time. Test in this order, because each depends on the last:

1. **Boots and enumerates over BLE.** Keyboard name is `richkbd`.
2. **All 42 keys register.** Every expander is talking — if only 14 keys work, one chip isn't
   initialized, which historically meant a shared-drvdata bug.
3. **Interrupt path works, not just polling.** Confirm `CONFIG_ZMK_KSCAN_DIRECT_POLLING=n` and that
   keys still respond. If keys only work with polling on, the `int-gpios` path is broken —
   suspect `ODR` (§3) first.
4. **Fast typing / chords don't drop keys.** See risk R1 below.
5. **Idle current draw** vs. the Step 0 baseline. This is where a regression would hide, and it's
   the reason the 2023 patch set `iodir` and disabled `&qspi`.
6. **Sleep and wake.** `CONFIG_ZMK_SLEEP=y` with a 15-minute idle timeout; confirm a keypress
   wakes it and the first keystroke isn't lost.

### Step 8 — decommission

Only after several days of real use:

- `docker rm db125e0b4174` (the 2023 devcontainer), then
  `docker volume rm zmk-zephyr zmk-zephyr-modules zmk-zephyr-tools zmk-root-user zmk-config`
  — frees ~6 GB. **Do not do this earlier**: those volumes are the last copy of the exact tree the
  known-good firmware was built from.
- Archive this repo. Delete `.devcontainer/` from the new one.

---

## 6. Risks

**R1 — shared INT line with edge detection can mask an event.** The nRF pin is configured
`GPIO_INT_EDGE_TO_ACTIVE`. With six open-drain pins wire-ORed, if chip B is holding INT low
(unread) and a key on chip A goes down, there is no new falling edge. Mitigation is structural:
ZMK's kscan-direct, on any interrupt, scans **all** input pins, so reading the matrix clears every
chip's INT and the line releases together. Expected to be self-healing, but this is exactly what
fast-typing testing (step 7.4) is for. Our current design sidesteps the issue by construction —
one board-level interrupt, one full scan — so a regression here is plausible.

**R2 — three devices, one interrupt pin.** Each expander independently calls
`gpio_add_callback()` and `gpio_pin_interrupt_configure_dt()` on the *same* nRF pin. Legal
(separate callback structs, idempotent configuration) but unusual; upstream likely never tested
three devices sharing one `int-gpios`. Watch for the third device's init failing.

**R3 — level-triggered pin interrupts on an expander.** Modern kscan asks for
`GPIO_INT_LEVEL_ACTIVE` per pin, which the driver emulates with `INTCON`/`DEFVAL` comparison
against a reference value rather than true level detection. Semantics may differ subtly from a
native GPIO. Untested on this hardware.

**R4 — power regression.** The 2023 bring-up fought a power problem and won by unclear means
(commit `c3f151bf`: "Fixed the power consumption problem, but not sure which change did it"). Some
of that fix is in the patch we are deleting (`iodir`) and some in the overlay (`&qspi` disabled).
Budget time to re-find it. Baseline measurements in step 0 are what make this tractable.

**R5 — Zephyr 3.2 → 4.1 is three years of churn.** Devicetree, HWMv2 board naming, kscan API,
behavior definitions. The keymap is the least likely thing to break; the overlay and conf are the
most.

---

## 7. Rollback

At every point before step 8, rollback is: reflash the archived known-good uf2 from step 0. The
old fork keeps working as long as this repo, `patches/0001-mcp230xx-init-registers.patch`, and the
`zmk-zephyr*` Docker volumes exist. Keeping all three until validation is complete is the whole
point of step 8's ordering.

---

## Appendix: how to re-verify these facts

Every claim in §2 came from one of these:

```sh
# when interrupt support landed
for v in v3.2.0 v3.3.0 v3.4.0 v3.5.0 v4.1.0; do
  printf '%-8s ' "$v"
  curl -sSfL "https://raw.githubusercontent.com/zephyrproject-rtos/zephyr/$v/drivers/gpio/gpio_mcp23xxx.c" \
    | grep -c gpio_int
done

# what ZMK pins today
curl -sSfL https://raw.githubusercontent.com/zmkfirmware/zmk/main/app/west.yml

# the driver ZMK would give us, and whether it sets ODR
curl -sSfL "https://raw.githubusercontent.com/zmkfirmware/zephyr/v4.1.0%2Bzmk-fixes/drivers/gpio/gpio_mcp23xxx.c" \
  | grep -nE "write_iocon|REG_IOCON|ODR"

# the expander binding
curl -sSfL "https://raw.githubusercontent.com/zmkfirmware/zephyr/v4.1.0%2Bzmk-fixes/dts/bindings/gpio/microchip,mcp23xxx.yaml"

# board target
curl -sSfL https://raw.githubusercontent.com/zmkfirmware/zmk/main/app/boards/seeed/xiao_ble/board.yml

# modern kscan: per-pin interrupts, DT debounce
curl -sSfL "https://raw.githubusercontent.com/zmkfirmware/zmk/main/app/module/dts/bindings/kscan/zmk%2Ckscan-gpio-direct.yaml"
curl -sSfL https://raw.githubusercontent.com/zmkfirmware/zmk/main/app/module/drivers/kscan/kscan_gpio_direct.c \
  | grep -n "interrupt_configure"

# the INT net, from the fab netlist
grep -iE "INT" \
  ~/pcb/richkbd_wireless/pcb/production/Rich_Keyboard_2.0_2023-07-27_22-08-39/netlist.ipc
```