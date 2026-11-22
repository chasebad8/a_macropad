# a_macropad

A programmable USB macro pad with per-key RGB LEDs, rotary encoders, a slider and an OLED.
It follows on from [a_keyboard](https://github.com/chasebad8/a_keyboard), bare-metal ATmega32u4 + LUFA keyboard firmware.

## Decisions so far

- **MCU:** RP2040. Prototyping on a Raspberry Pi Pico, Zephyr board target `rpi_pico`.
  Whether the final board uses a Pico module or a bare RP2040 is still open (see PCB options).
  - A second Pico running `debugprobe` firmware acts as an SWD debugger (`west flash` / `west debug`).
- **Firmware:** Zephyr RTOS. Planned subsystems:
  - Input: `gpio-kbd-matrix` (key matrix), `gpio-qdec` (encoders)
  - USB device stack, HID: keyboard + consumer control + raw HID for the companion app
  - `worldsemi,ws2812-rpi_pico-pio` for the RGB LEDs
  - SSD1306 display driver (LVGL optional)
  - ADC for the slider
  - Settings over NVS to store keymaps and macros in flash
- **Companion app (later):** live remapping and macro recording over raw HID, per-app
  profile auto-switching, and sending data back to the OLED. The PC also sets volume
  from the slider's value, since HID has no absolute volume control.

## Layout

```
 ┌────────────────────────────────────────────────────────┐
 │  ( ◉ )      ┌──────────────────────────┐      ( ◉ )    │
 │  KNOB 1     │ ▌LAYER: OBS      🔊 72%  │      KNOB 2   │
 │  ⟲ vol ⟳    │ ▌SCENE: Camera   ● REC   │     ⟲ zoom ⟳  │
 │  push=mute  └──────────────────────────┘    push=reset │
 │                     OLED 128×64                        │
 │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐         ┌───┐     │
 │  │  ◇   │ │  ◇   │ │  ◇   │ │  ◇   │         │ ▲ │100% │
 │  │ MIC  │ │ CAM  │ │ REC  │ │ LIVE │         │   │     │
 │  └──────┘ └──────┘ └──────┘ └──────┘         │ █ │     │
 │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐         │ █ │     │
 │  │  ◇   │ │  ◇   │ │  ◇   │ │  ◇   │         │ ═ │◄ cap│
 │  │ COPY │ │PASTE │ │ UNDO │ │ REDO │         │   │     │
 │  └──────┘ └──────┘ └──────┘ └──────┘         │   │     │
 │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐         │   │     │
 │  │  ◇   │ │  ◇   │ │  ◇   │ │  ◇   │         │ ▼ │ 0%  │
 │  │ FN1  │ │ FN2  │ │ FN3  │ │LAYER │         └───┘     │
 │  └──────┘ └──────┘ └──────┘ └──────┘        SLIDER     │
 │                                                        │
 │ ◇ ◇ ◇ ◇ ◇ ◇ ◇ ◇ ◇ ◇ ◇ ◇ ◇ ◇ ◇ ◇ ◇ ◇ ◇  ← underglow /   │
 └──────────────────────────────────────── level bar ─────┘
   ◇ = addressable RGB LED (WS2812 / SK6812)      [USB-C] ▲ back edge
```

## Hardware

| Function | Part | Pins |
|---|---|---|
| Key matrix | 12 MX switches (4×3), hot-swap sockets | 7 |
| Encoders | 2× EC11 with push button | 6 |
| Slider | 10 kΩ linear slide pot (later: a motorized fader?) | 1 (ADC) |
| Display | SSD1306 128×64 OLED, I2C | 2 |
| Lighting | ~31× WS2812/SK6812 (12 per-key + ~19 underglow), single data line | 1 (PIO) |
| **Total** | | **17 of 26 GPIO** |

- Full-white LED current is ~1.9 A, more than USB supplies. Cap brightness in firmware (~30–40%).
- Add a 1000 µF bulk cap on the LED 5 V rail and a ~330 Ω resistor on the LED data line.

## PCB options (WIP)

The MCU is the RP2040. The open question is how it goes onto the macro pad PCB:

| Option | Pros | Cons |
|---|---|---|
| Pico module soldered on by its castellated edges | Simplest; regulator, flash, crystal and USB already done and tested | Micro-USB only (the D+/D− test pads TP2/TP3 are underneath, hard to reach once soldered down); takes more board space |
| Bare RP2040 on the PCB | USB-C directly on the board; smaller and lower profile; full control of pin placement (30 GPIO instead of the Pico's 26) | More parts and layout work; QFN-56 at 0.4 mm pitch to solder |

The RP2040 is one of the easier MCUs to put on a board: its 1.1 V core comes from an internal linear
regulator, so it only needs decoupling caps. The Pico's own schematic and Raspberry Pi's
"Hardware design with RP2040" guide are good references.

What the bare-chip option has to add (the Pico provides these):

- 3.3 V LDO from VBUS (e.g. AP2112K-3.3)
- 12 MHz crystal and its load caps
- QSPI flash, e.g. W25Q16JV (2 MB, same as the Pico) or W25Q128 (16 MB)
- USB-C: 5.1 kΩ CC pull-downs, 27 Ω series resistors on D+/D−, ESD protection (e.g. USBLC6-2)
- BOOTSEL button (on the flash CS line) and RUN/reset button
- SWD header for the debugprobe

Either option should also have a level shifter on the WS2812 data line (e.g. 74AHCT1G125):
3.3 V logic is borderline for 5 V LEDs.

Hand soldering a QFN-56 needs a stencil and hot plate, or the MCU section can be machine-assembled
(JLCPCB / PCBWay stock the RP2040).

Firmware: prototype on the Pico with all pins in devicetree. A bare-chip board then becomes its
own Zephyr board definition based on `rpi_pico`; app code is unchanged.

## Roadmap

1. Breadboard or hand-wire: keys + 1 encoder; USB HID keyboard + consumer control working.
2. Layers and macros, saved with settings/NVS.
3. Slider, OLED, RGB feedback (mute state, layer colours, level bar).
4. Raw HID protocol + a basic companion app.
5. Custom PCB (Pico module or bare RP2040, see PCB options) and a 3D-printed case.

## Parts sourcing

PiShop.ca (local pickup in Nepean, $2): Raspberry Pi Pico (RP2040), price to confirm.
