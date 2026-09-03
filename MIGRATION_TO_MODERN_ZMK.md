# Migration plan: modern ZMK, no fork, (almost) no patches

**Status:** **in progress**, started 2026-09-03. Steps 1–5 are done and live in a new repo,
[rvnash/richkbd-zmk-config](https://github.com/rvnash/richkbd-zmk-config); the zephyr fork is
[rvnash/zephyr](https://github.com/rvnash/zephyr) branch `richkbd-odr`. What is **not** done is
everything that needs the physical keyboard: step 7 validation and the power comparison. Do not
flash until you have read §8.

The plan below is kept as written, with per-step notes on what actually happened.

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

Two more facts from the same netlist, both verified rather than inferred:

```
U1-4,  U2-4,  U3-4    = GPB7    single-pin net → unconnected
U1-24, U2-24, U3-24   = GPA7    single-pin net → unconnected
U1-14, U2-14, U3-14   = RESET   single-pin net → floating
```

The unconnected GPA7/GPB7 are exactly the two bits per chip that `iodir = 0x7F7F` turns into
outputs — see the end of this section. The floating `RESET` is out of spec per the MCP23017
datasheet (it wants to be biased) but has worked for years; it is **not** a migration concern, just
a note for a future board revision. It does mean the modern driver's optional `reset-gpios` is
unavailable to us — there is no GPIO on that net to name.

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

Our patch makes GPA7/GPB7 outputs on each chip — so unused inputs don't float. There is no
upstream equivalent.

This was a deliberate, hardware-informed choice, not stray tinkering: the netlist above shows
GPA7 and GPB7 physically unconnected on all three expanders, and they are precisely the two bits
`0x7F7F` clears (low byte = IODIRA/GPA0–7, high byte = IODIRB/GPB0–7). Do not "clean it up" without
replacing it.

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

### Step 0 — prep: do all of this *before* the migration weekend

None of it requires committing to the migration, and item 1 has to start weeks ahead.

**0.1 Start the power baseline now. This is the only perishable item.**

Risk R4 is the real danger here, and the old firmware's power behaviour can only be measured while
the old firmware is the one running. Without that number a post-migration regression is
unfalsifiable — exactly the position the 2023 bring-up ended in ("fixed the power consumption
problem, but not sure which change did it"). A few lines of data now replace a week of bisecting
later.

**What the baseline measures.** Tie it to a specific build or it means nothing:

| | |
| --- | --- |
| firmware | `build/zephyr/zmk.uf2`, md5 `9a81e8d92cede1058e1786bd5871bdc1` |
| built | 2026-09-03 11:00 |
| source | commit `3e94be47` + `patches/0001-mcp230xx-init-registers.patch` |
| archived as | `firmware/zmk-2026-09-03-known-good.uf2` (per 0.2) |
| relevant config | `ZMK_IDLE_TIMEOUT=1`, `ZMK_SLEEP=y`, `ZMK_IDLE_SLEEP_TIMEOUT=900000`, `ZMK_KSCAN_DIRECT_POLLING=n`, `&qspi` disabled |

**How to read the percentage.** By hand, from the Bluetooth menu-bar item or System Settings →
Bluetooth (ZMK reports it over BLE BAS). It cannot be scripted on this Mac: `ioreg -r -k
BatteryPercent`, `ioreg -l | grep -i batterylevel`, and `system_profiler SPBluetoothDataType` were
all checked on 2026-09-03 and none expose a level for `richkbd` (`DA:24:D1:79:29:D7`), even while
connected.

**Log.** Same time of day each reading, ideally after a normal day of typing:

| date | battery % | notes |
| --- | --- | --- |
| 2026-09-03 | 45% | baseline start — firmware above, freshly flashed; read from the Bluetooth panel |
| | | |
| | | |
| | | |
| | | |
| | | |
| | | |

**Derived figure:** ______ %/day over ______ days.

**Acceptance after migration:** re-measure over the same span and compare. Within ~20 %/day of the
baseline is noise; consistently worse is the R4 regression, and the first suspects are the missing
`iodir` unused-pin handling (§3) and `&qspi` re-enabling itself via ZMK's modern board dts (step 3).
Record the post-migration figure here too rather than in a new file, so the comparison stays in one
place.

**0.2 Make rollback real, then prove it works.**

"The uf2 in `build/`" is not a rollback plan — a `-p` build deletes that directory, which is how a
saved firmware was already lost once. Commit the artifacts:

```sh
mkdir -p firmware
cp build/zephyr/zmk.uf2 firmware/zmk-2026-09-03-known-good.uf2   # current, confirmed working
cp ~/Downloads/zmk.uf2  firmware/zmk-2024-08-15-devcontainer.uf2 # older devcontainer build
```

355 KB each; fine in git. Then **actually flash the 2024 one and flash back to current.** A
rollback path that has never been exercised is not a rollback path, and flashing on this board has
non-obvious failure modes (USB hubs — see `HOW_TO_BUILD_FROM_CLI.md`).

**0.3 De-risk the toolchain separately from the keyboard.**

The migration bundles two independent risks: *does modern ZMK build and flash for me at all*, and
*does richkbd work on it*. Separate them. In one evening:

1. Create the config repo from `unified-zmk-config-template` (§4 layout).
2. Copy the shield in as-is, no fixes.
3. Let GHA build it, download the artifact, flash it.

Expect the keyboard **not** to work — there is no `ODR` bit yet (§3). The point is to validate the
module layout, the `xiao_ble/nrf52840/zmk` board target, the workflow, artifact download, and
flashing a GHA build. Then the migration weekend is only about the keyboard.

**0.4 Stand up the zephyr fork while you are there.**

Fork `zmkfirmware/zephyr`, branch from `v4.1.0+zmk-fixes`, make the one-line change from §3 Option
A, push. Fifteen minutes, and it is the one piece of this migration with no upstream equivalent.

**0.5 Leave the current tree alone.**

Do **not** pre-fix `SHIELD_MY_BOARD`, the empty `seeeduino_xiao.overlay`, or the stale `intcon`
comment in the patch. That is churn against a working keyboard for no benefit; step 2 fixes them in
the new repo where they cost nothing.

**0.6 Docker volumes: keep until 0.2 is done, then discard.**

The volume copy of `gpio_mcp230xx.c` is now byte-identical to the patched tree
(`md5 78eac5d7bece7c745e5b869ab19dc755`, both sides), so `patches/0001-…` fully reproduces it. What
remains in those ~6 GB is a re-clonable older zephyr revision and 298 MB of ccache. Once you have
*proven* you can flash an archived uf2, they stop being a safety net — then §5 step 8 applies.

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
| `CONFIG_ZMK_IDLE_TIMEOUT=1` | keep — verified deliberate, not a typo. `app/Kconfig:319` is "Milliseconds of inactivity before entering idle state (OLED shutoff, etc)", default 30000. There is no OLED, and idle and deep-sleep timers both measure from last activity, so `1` does not hasten sleep — it just enters idle immediately, which is free when wake is interrupt-driven |
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
NL=~/pcb/richkbd_wireless/pcb/production/Rich_Keyboard_2.0_2023-07-27_22-08-39/netlist.ipc
grep -iE "INT" "$NL"

# every pin of an expander and the net it lands on (single-pin nets = unconnected).
# This is how GPA7/GPB7-unconnected and RESET-floating were established.
grep '^327' "$NL" | awk '$2=="U1"{printf "%s=%s ", $3, substr($1,4)}'

# idle-timeout semantics, in this tree
grep -n -A6 "config ZMK_IDLE_TIMEOUT" app/Kconfig
```