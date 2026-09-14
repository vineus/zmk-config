# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Personal ZMK firmware config for a **Halcyon Kyria rev4** split keyboard with the
splitkb wireless controllers. Created 2026-09-11 by migrating an existing QMK
keymap (see below) — the wireless controller is nRF52840 and runs ZMK, so QMK is
not an option for this hardware.

This is a personal side project, **not a Vibe work repo**: no Linear tickets, no
`repo:vibe-ad/...` tagging, no gitops/promotion release model.

## Relationship to the QMK repos

| Path | What it is |
| :--- | :--- |
| `~/dev-perso/qmk_userspace` | The predecessor. Branch `halcyon`, remote `vineus/qmk_userspace`. The keymap this repo was ported from is `keyboards/splitkb/halcyon/kyria/keymaps/vineus_hlc/keymap.c`. Still valid for the **wired** controller. |
| `~/dev-perso/qmk_firmware` | Personal QMK fork (`vineus/qmk_firmware`, branch `vineus_wip`), registered as `qmk config user.qmk_home`. Only relevant to the QMK repo above. |

Consult the old `keymap.c` when a binding here looks arbitrary — most of it is a
direct port. What deliberately did **not** come across:

- `rgb_matrix_indicators_advanced_user()` — per-key layer colors, the purple Hyper
  hints, the pink emoji key. ZMK exposes underglow effects only; there is no
  per-key indicator hook. Don't try to reimplement it.
- `keyboard_post_init_user()` — gold reactive startup. Nearest equivalent is
  `CONFIG_ZMK_RGB_UNDERGLOW_HUE_START` / `EFF_START` in a `.conf`.
- The TFT display graphics (`hlc_tft_display`) — replaced by the epaper module.

One bug was fixed rather than ported: QMK's `_ADJUST` layer was unreachable
(nothing bound `MO(_ADJUST)`, no tri-layer hook). Here `conditional_layers` maps
Lower+Raise onto it.

## Architecture

Boards, shields and drivers come from `zmk-halcyon-module`, pulled in by
`config/west.yml` (which also pins ZMK itself to `splitkb/zmk` @
`main+halcyon-fixes`, not upstream).

**`west.yml` points the module at `vineus/zmk-halcyon-module`, a fork**, branch
`feat/vibetv-epaper-image`. It carries two changes, one commit each so either can
be dropped independently:

1. the custom `vibetv` epaper image
2. the battery percentage on the peripheral status bar

Nothing else should accumulate there — anything achievable from this repo belongs
here, not in the fork. To drop it entirely, point the module back at the `splitkb`
remote @ `main` and pick a stock image in `build.yaml`.

The revision is a **branch, not a SHA**, so force-pushing or rebasing that branch
changes what CI builds. Rebase it when splitkb moves `main`.

Only three files are yours:

- `build.yaml` — the GitHub Actions build matrix. One entry per firmware image.
  Shields are composed as `<keyboard>_<side> <battery> <module>`.
- `config/halcyon_kyria.keymap` — layers, behaviors, encoder bindings. Overrides
  the module's default at `boards/shields/halcyon_kyria/halcyon_kyria.keymap`,
  which is the best starting point when adding something.
- `config/halcyon_kyria.conf` — Kconfig overrides.

The `config/*.json` files are physical layouts for ZMK Studio; `boards/shields/`
is an empty hook for local shields. Neither normally needs touching.

The three **stock** epaper images — `mod_display_epaper_mountain` / `_forest` /
`_cityscape` — are just shield names, so switching is a one-word `build.yaml` edit.
`CONFIG_HALCYON_EPAPER_WIDGET_INVERTED=y` flips light/dark on any of them.

A **custom** image is why the fork exists. The art lives as LVGL bitmap arrays in
the shield's `widgets/art.c`, and both `status.c` and `peripheral_status.c` pick
one at compile time via `#if IS_ENABLED(CONFIG_SHIELD_MOD_DISPLAY_EPAPER_*)` —
there is no runtime hook, and a config repo cannot add sources to a shield it does
not own.

Adding another custom image means touching **seven** files in the fork, all under
`boards/shields/mod_display_epaper/`: the array and descriptor in `widgets/art.c`,
a `def_bool $(shields_list_contains,...)` entry in `Kconfig.shield`, a `.conf` and
a `.overlay` (copy any existing pair — the confs are identical and the overlay is
a one-line include), a selection branch in **both** `widgets/status.c` and
`widgets/peripheral_status.c`, and a `siblings:` entry in the `.zmk.yml`. Patching
only `peripheral_status.c` works today because both halves are peripherals, but it
silently breaks if a half ever becomes central.

**Image format** — 164x88, `LV_COLOR_FORMAT_I1`, stride 21, an 8-byte palette
header (black, white; swapped under `CONFIG_HALCYON_EPAPER_WIDGET_INVERTED`), then
1848 bytes of packed bits, MSB first. Stored **rotated 90 degrees CCW** from what
the panel shows, so design at 88x164 portrait and rotate on the way in. The shield
README suggests LVGL's `LVGLImage.py`, but the format is simple enough to emit
directly — and doing so lets you round-trip the result back to a PNG to verify it
before ever flashing.

**Converting art** for a 1-bit 88x164 panel: unsharp-mask *before* downscaling,
then a plain threshold around 45%. Sharpening first keeps the thin white gaps
between black shapes from closing up. Floyd-Steinberg dithering is worse than
thresholding here — it adds noise where line art wants flat areas.

## Current hardware layout

- **Dongle**: central, ZMK Studio enabled. Holds the keymap and presents USB HID.
- **Left half**: peripheral, `mod_encoder_left`, `mod_battery_coincell`.
- **Right half**: peripheral, `mod_display_epaper_vibetv`, `mod_battery_coincell`.

Both halves carry `-DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n`. **Omitting it on a half is
not a no-op that merely leaves it dongleless** — the half becomes a second central,
advertises to hosts instead of joining the dongle, and so does nothing wirelessly
while still working over USB (a central has its own HID output). That USB-works /
wireless-doesn't split is the signature of this mistake; it cost an afternoon on
2026-09-11.

The encoder is on the **left** half (confirmed 2026-09-11). The dongleless variant
is written out and commented in `build.yaml`.

## Build and verify

There is no local toolchain. GitHub Actions builds on every push and PR:

```bash
gh run list --branch <branch> --limit 1
gh run watch <run-id> --exit-status --compact   # ~5 min per target
gh run view <run-id> --log-failed               # full log on failure
```

Artifacts land as a `firmware` zip on the run, one `.uf2` per build target.

Local builds are possible via the [ZMK local toolchain](https://zmk.dev/docs/development/local-toolchain/setup)
with `splitkb/zmk-halcyon-module` added as a module, but nothing here depends on it.

### Validating the keymap before pushing

Every layer must have exactly **60 bindings** (12 / 12 / 16 top rows and thumbs,
plus the 10-key Halcyon row). A miscount fails deep in devicetree with an unhelpful
message, so count first:

```bash
python3 - <<'PY'
import re
src = open('config/halcyon_kyria.keymap').read()
for name, body in re.findall(r'(\w+_layer)\s*\{(.*?)\n        \};', src, re.S):
    m = re.search(r'bindings = <(.*?)>;', body, re.S)
    if m:
        n = len(re.findall(r'&\w+', re.sub(r'//.*', '', m.group(1))))
        print(name, n, 'OK' if n == 60 else '<-- expected 60')
PY
```

## Gotchas

- **A failing build reports a misleading error.** When devicetree fails, `zephyr/.config`
  is never written, and the workflow's ZMK-compat check then reports *"The selected
  board is not set up for ZMK and there is a ZMK variant available"*. That is a red
  herring — `halcyon_wireless//zmk` is correct. Scroll up in the log for the real
  `devicetree error:` line.
- **Behaviors that exist upstream may not exist in this revision.** `inc_dec_ms` does
  not; `config/halcyon_kyria.keymap` defines `inc_dec_msc` from the generic
  `zmk,behavior-sensor-rotate-var` compatible wrapping `&msc`. Before using an
  unfamiliar behavior, check it is present:
  `gh api "repos/splitkb/zmk/contents/app/dts/behaviors?ref=main%2Bhalcyon-fixes" --jq '.[].name'`
- `&msc` requires `CONFIG_ZMK_POINTING=y` — that is the only reason the `.conf` exists.
- **`sensor-bindings` order is fixed**: left halcyon, right halcyon, left soldered,
  right soldered. All four are bound even though only one encoder is fitted, so
  swapping sides needs no encoder edit.
- **Flashing**: double-tap reset for the `HALCYON` mass-storage device, drag the
  `.uf2`, then **press the physical reset button** —
  [Adafruit bootloader #368](https://github.com/adafruit/Adafruit_nRF52_Bootloader/issues/368),
  skipping it drains the battery much faster.
- ZMK Studio (on the central half) edits the keymap live without a rebuild. Changes
  made there do not flow back into this repo — port them by hand.

## Conventions

Conventional commits, git-spice (`gs cc` / `gs ca --no-edit` / `gs ss`), PR rather
than pushing to `main`.
