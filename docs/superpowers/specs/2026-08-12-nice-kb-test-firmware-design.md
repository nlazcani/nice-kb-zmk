# nice_kb assembly-test firmware (left + right, connected split)

**Date:** 2026-08-12
**Status:** approved

## Goal

Produce two flashable firmwares — `nice_kb_left` and `nice_kb_right` — that pair
as a split and let the builder verify the partially-assembled hardware:

- Left half: 26 keys. Rows 1-5 x cols 1-5 fully populated (25), plus a single
  key at row 5 / col 6.
- Left half: PMW3610 trackball.
- Right half: 3 keys at row 1 / cols 1-3. No trackball.
- ZMK Studio reachable on the left (central) over USB.

This is validation firmware, not the final layout. The final Cosmos matrix is
still unknown; the placeholder 5x12 grid transform stays.

## Findings that shaped the design

1. **The right half was building the trackball driver.** The PMW3610 lives in
   the shared `nice_kb.dtsi` because the bench-test shields need it there, so
   the peripheral compiled the driver and enabled the DT node for a sensor that
   is not populated. Confirmed in the CI log: run 29989390816 shows
   `Building C object modules/zmk-pmw3610-driver/.../pmw3610.c.obj` under the
   `nice_kb_right` job. This is the one real defect fixed here.

2. **`config/nice_kb.conf` was not dead.** ZMK strips the `_left` / `_right`
   suffix when resolving a split shield's config, so run 29989390816 logs
   `-- ZMK Config Kconfig: .../config/nice_kb.conf` for *both* halves. The left
   half already had `CONFIG_ZMK_POINTING`, `CONFIG_PMW3610_ALT`,
   `CONFIG_ZMK_STUDIO` and `CONFIG_NFCT_PINS_AS_GPIOS`. Splitting into per-half
   files is therefore a deliberate refactor, not a bug fix: its purpose is to
   let the peripheral drop the pointing stack.

3. **Keymap resolution was already correct for these shields.** Same suffix
   stripping applies — run 29989390816 logs
   `-- Using keymap file: .../config/nice_kb.keymap` for `nice_kb_left` with no
   `cmake-args` present. The `d28607c` glob problem was specific to the
   `nkbtest` shield name, which has no `nice_kb` prefix to strip. The explicit
   `-DKEYMAP_FILE` added here is defensive only.

## Design

### Matrix and transform

Unchanged: 5x12 placeholder transform. Left occupies transform cols 0-5; the
right overlay keeps `col-offset = <6>` so it lands on cols 6-11. Both halves
declare all 6 column GPIOs; unwired cells sit on pull-downs and never assert, so
no pin changes are needed for the partial build.

### Per-half Kconfig

| File | Contents |
| --- | --- |
| `config/nice_kb_left.conf` | Studio, SPI/input/pointing/PMW3610, NFC pins as GPIO |
| `config/nice_kb_right.conf` | Studio, NFC pins as GPIO. No pointing stack. |

`CONFIG_ZMK_STUDIO=y` is set on **both** halves. The two halves share one keymap
and that keymap binds `&studio_unlock`; the behavior must be compiled in on the
peripheral or the reference fails to link. The peripheral has no RPC transport
(no `studio-rpc-usb-uart` snippet), which is correct — Studio only ever talks to
the central.

`config/nice_kb.conf` is deleted; its content moved into the left file. Both
halves previously shared it via ZMK's `_left`/`_right` suffix stripping, so the
left half's effective config is unchanged — only the right half loses the
pointing stack.

### Right-half trackball teardown

`nice_kb_right.overlay` disables `&trackball_listener`, `&trackball`, and
`&spi0`. The `nkb*` bench shields are untouched.

### Left-half trackball

Keeps the tuned values already in `nice_kb.dtsi` (SPI 500 kHz, CPI 1000,
`invert-x`) and adds `force-awake` + `force-awake-4ms-mode`, the low-latency
settings the `nkbtest` bench shield validated.

### Test keymap (`config/nice_kb.keymap`)

Every wired key emits a **unique** character, so a matrix fault appears as a
specific missing or duplicated character in a text editor.

Layer 0 "Test", left half:

| | c1 | c2 | c3 | c4 | c5 | c6 |
| --- | --- | --- | --- | --- | --- | --- |
| r1 | 1 | 2 | 3 | 4 | 5 | - |
| r2 | Q | W | E | R | T | - |
| r3 | A | S | D | F | G | - |
| r4 | Z | X | C | V | B | - |
| r5 | 6 | 7 | 8 | 9 | 0 | `&lt 1 MINUS` |

Right half: r1c1-c3 = `Y U I`. All other cells `&none`.

The lone col6 key is `&lt 1 MINUS`: a tap types `-` (proving the key works), a
hold enters the Fn layer.

Layer 1 "Fn": `&studio_unlock`, `&bt BT_SEL 0..3`, `&bt BT_CLR`,
`&bt BT_CLR_ALL`, `&out OUT_TOG/OUT_USB/OUT_BLE`, `&bootloader`, `&sys_reset`.
The three right-half keys stay `&trans`, so they keep typing `Y U I` while the
layer is held — a live check that the split link survives layer changes.

Both layers are 60 bindings, matching the 5x12 transform and the 60-key physical
layout.

### build.yaml

`-DKEYMAP_FILE=.../config/nice_kb.keymap` added to the `nice_kb_left`,
`nice_kb_right`, and `nice_kb_left_debug` entries. Bench-shield entries are
unchanged.

## Verification

- Keymap binding count per layer == 60 (scripted check).
- `build.yaml` parses as YAML; the three `nice_kb*` entries carry `cmake-args`.
- Compile is done by the ZMK GitHub Actions cloud build; there is no local
  Zephyr toolchain in this repo.
- Run 31590448029: all 19 targets green. Both per-half confs confirmed merged
  (`Merged configuration '.../config/nice_kb_left.conf'` and the `_right`
  equivalent).
- Right half no longer compiles the sensor driver: the `nice_kb_right` job now
  logs `No SOURCES given to Zephyr library: ..__zmk-pmw3610-driver`, and the
  artifact shrank from 394,752 to 381,440 bytes. The left artifact's PMW3610
  build output is byte-for-byte unchanged in structure (same 48 log mentions),
  confirming no regression to the trackball half.

## Flashing

1. Flash `settings_reset` to **both** halves (clears stale bonds).
2. Flash `nice_kb_left`, then `nice_kb_right`.
3. Halves auto-pair on boot. Connect the left half over USB for Studio.

## Out of scope

The real Cosmos matrix, the final keymap, RGB underglow, and any change to the
`nkb*` bench-tuning shields.
