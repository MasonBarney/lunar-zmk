# Lunar Dust — ZMK config

ZMK firmware config for a wireless split keyboard ("Lunar Dust"), built from the
`paw` shield (derived from Bastardkb paw) on nice!nano hardware.

- Build targets: `nice_nano//zmk` + `paw_left` / `paw_right` (see `build.yaml`)
- Split over BLE; **left half is central** (`ZMK_SPLIT_BLE_ROLE_CENTRAL` in `Kconfig.defconfig`)
- Extra west modules: `zmk-driver-paw3222` (sekigon), `zmk-pmw3610-driver` (badjeff)

## Layout of this repo

| Path | What it is |
|---|---|
| `config/paw.keymap` | keymap — 2 layers, 44 bindings each |
| `config/boards/shields/paw/` | the real shield: matrix, encoders, trackball, display |
| `config/boards/shields/nice_view_gem/` | custom nice!view status screen |
| `config/boards/shields/lpm_view/` | alternate low-power display shield |

## Hardware as the firmware actually defines it

Source of truth is `paw.dtsi` + `paw_{left,right}.overlay`, not any diagram.

- **Matrix**: 4 rows × 12 cols total → **4 rows × 6 cols per half**, 44 keys (22/half)
- **Diode direction**: `row2col`
- **Rows** (both halves): `P0.31`, `P0.29`, `P0.02`, `P1.15`
- **Cols left**: `P0.22`, `P0.24`, `P1.00`, `P0.11`, `P1.04`, `P1.06`
- **Cols right**: same six pins in reverse order; right adds `col-offset = <6>`
- **Trackball**: PAW3222 on **SPI2** — SCK `P1.13`, MOSI+MISO both `P0.20` (3-wire SDIO), CS `P0.17`, IRQ `P0.09`
- **Encoders**: EC11, A `P0.06` / B `P0.08`; `disabled` in `paw.dtsi`, enabled per-half in the overlays
- **nice!view**: SPI3 — SCK `P0.10`, MOSI `P1.01` (underside pad), CS `P1.11`
  (Sharp `ls0xx` memory LCD — chip-select/clock/data only, **no D/C line**)
- **RGB underglow**: WS2812 on SPI1 — SCK `P0.19`, MOSI `P0.21`; chain 28 (left) / 25 (right)

## nice!nano v2 pin naming

The board silkscreens **raw nRF52840 port names** (`P0.22`, `P1.15`), not Pro Micro
D-numbers. Both dialects address the same physical pin and both appear in ZMK configs
(`&gpio0 22` vs `&pro_micro 4`). This repo uses `&gpioN` throughout — keep it that way.

Header mapping, in physical order:

| Silkscreen | Pro Micro | | Silkscreen | Pro Micro |
|---|---|---|---|---|
| `P0.06` | 1 / TXO | | RAW | — |
| `P0.08` | 0 / RXI | | GND | — |
| GND | — | | RST | — |
| GND | — | | VCC | — |
| `P0.17` | 2 | | `P0.31` | A3 / 21 |
| `P0.20` | 3 | | `P0.29` | A2 / 20 |
| `P0.22` | 4 | | `P0.02` | A1 / 19 |
| `P0.24` | 5 | | `P1.15` | A0 / 18 |
| `P1.00` | 6 | | `P1.13` | SCK / 15 |
| `P0.11` | 7 | | `P1.11` | MISO / 14 |
| `P1.04` | 8 | | `P0.10` | MOSI / 16 |
| `P1.06` | 9 | | `P0.09` | 10 |

Also broken out on the underside: `P1.01`, `P1.02`, `P1.07` (spare GPIO), plus SWD/SWC
and the `B+`/`B−` battery pads.

**Orientation**: the nice!nano prints its labels on the **bottom**. Read a pinout with
the labels facing you and USB-C pointing away — RAW then falls on the left, `P0.06` on
the right. Viewed from the component side the two columns are mirrored.

`P0.09` and `P0.10` are the nRF52840's NFC pins; they only work as GPIO with
`CONFIG_NFCT_PINS_AS_GPIO=y`. ZMK's nice!nano board definition sets it, so stock builds
are fine — recheck if the board definition ever changes. `P0.09` is the trackball IRQ here.

## Published artifacts

- [nice!nano Wiring Reference](https://claude.ai/code/artifact/4d036039-86e7-446b-abcc-93ed0737649a) — hand-wiring pinouts, bottom view
- [velvet_v3_ui — Current Keymap](https://claude.ai/code/artifact/968bf346-db09-4f35-ac2e-a68893c8de75)
- [Two-Board Wired Split](https://claude.ai/code/artifact/8e76b9d1-fb07-4564-a8e3-c07f71ce7310) — superseded UART-link design

> **The wiring reference artifact describes a different build from this firmware.**
> It documents a 5×6 COL2ROW matrix with an Azoteq TPS65 trackpad on I²C; this repo is
> 4×6 `row2col` with a PAW3222 trackball on SPI, and rows/columns are transposed relative
> to the diagram. Same physical pins, different roles. Treat the repo as authoritative and
> reconcile the diagram before wiring anything.

## Open questions

- `paw_left.overlay` `&spi1` sets `pinctrl-1` twice and never sets `pinctrl-0`; the right
  overlay sets `pinctrl-0` correctly. Looks like a typo in the left half.
- **SPI1 (underglow) still routes to `P0.19` and `P0.21`, which are not broken out on a
  nice!nano.** SPI3 had the same problem and was repointed to `P0.10` + `P1.01`; SPI1 was left
  alone pending a decision. The only pins still free are the `P1.02` and `P1.07` pads.
- The header is fully committed: all 18 pins are assigned, and `P1.01`/`P1.02` are now in use.
  Any new peripheral has to take `P1.07` or displace something.
