# Archived firmware

Flashable rollback artifacts, kept here because `build/` is deleted by any `west build -p`.

| file | built | source | md5 |
| --- | --- | --- | --- |
| `zmk-2026-09-03-known-good.uf2` | 2026-09-03 11:00 | `3e94be47` + `patches/0001-mcp230xx-init-registers.patch` | `9a81e8d92cede1058e1786bd5871bdc1` |
| `zmk-2024-08-15-devcontainer.uf2` | 2024-08-15 16:00 | built in the old VSCode devcontainer, from the `zmk-zephyr` Docker volume tree | `a2857ef4112f7ed8d08734dc7cdfb66f` |

The 2024 build predates commits `431bcac8`, `0aefc901`, `3e94be47` (the shift/Nav rearrangement),
so flashing it loses those keymap changes but is otherwise a known-good keyboard.

Flashing: see `../HOW_TO_BUILD_FROM_CLI.md`. Plug in **directly**, not through a hub.
