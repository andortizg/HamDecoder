# HamDecoder

![HamDecoder splash screen](tools/splash.png)

Stand-alone digital-mode decoder for ham radio. It listens to the audio of your receiver and shows a colour **waterfall** while decoding **CW, RTTY, PSK31, PSK63, SSTV and WEFAX** (images can be saved to a microSD card). Everything is driven by a single rotary encoder.

Runs on an **STM32F411 "Black Pill"** with a 2.4" **ILI9341** TFT, a rotary encoder, a 24LC512 EEPROM and an optional microSD. Built with the Arduino IDE + STM32duino.

**Author:** Andrés Ortiz, EA7HQL · **License:** [MIT](LICENSE)

---

## Contents

1. [Features](#features)
2. [Hardware](#hardware)
3. [Building and flashing](#building-and-flashing)
4. [Configuration (`config.h`)](#configuration-configh)
5. [User guide](#user-guide)
6. [How it works](#how-it-works)
7. [Troubleshooting](#troubleshooting)
8. [Project layout](#project-layout)
9. [License](#license)

---

## Features

- **Colour waterfall** (SDR style): 1024-point FFT, about 31 lines/s, spans of 3, 2 or 1 kHz, adjustable threshold, brightness, contrast, averaging and height, plus a spectrum trace and a frequency scale.
- **Decoders**
  - **CW**: adaptive speed (WPM), selectable filter width, squelch, AFC.
  - **RTTY**: 45, 45.45, 50, 75 and 100 Bd; shifts of 170, 200, 425, 450 and 850 Hz; normal/reversed; USOS; squelch (SNR), AFC.
  - **PSK31 / PSK63** (BPSK): quality indicator, squelch, AFC.
  - **SSTV**: 11 modes (Martin M1/M2, Scottie S1/S2/DX, Robot 36/72, PD50/90/120/180), automatic detection from the VIS code or forced mode.
  - **WEFAX**: 60/90/120/240 lines per minute, IOC 576 or 288, clock (PPM) and phase correction.
- **SD card**: received SSTV/WEFAX images are saved as BMP files.
- **Settings stored in EEPROM** (24LC512): they survive power cycles; the cursor frequency is remembered per mode.
- **One-knob interface**: four essential parameters always on screen, everything else in a menu.
- **Bilingual**: all on-screen text in Spanish or English, chosen at compile time.
- **Start-up splash screen** (optional, can be left out of the build).
- Audio capture and decimation in hardware (timer + ADC + DMA), so the CPU keeps about 14 % headroom at 96 MHz.

---

## Hardware

| Part | Notes |
|------|-------|
| MCU | STM32F411CE "Black Pill", 96 MHz |
| Display | ILI9341 2.4" TFT, 320×240, SPI1 |
| Encoder | Rotary encoder with push button |
| EEPROM | 24LC512 on I2C1 (address 0x50) |
| microSD | Optional, on the same SPI1 bus as the display |
| Audio | Receiver audio into PA0 (ADC1_IN0) |

### Pin assignment

| Function | Pin |
|----------|-----|
| Audio input (ADC) | PA0 |
| Encoder A (CLK) | PA1 |
| Encoder B (DT) | PA2 |
| Encoder push button | PA3 |
| SD card CS | PA4 |
| SPI1 SCK (TFT + SD) | PA5 |
| SPI1 MISO (TFT + SD) | PA6 |
| SPI1 MOSI (TFT + SD) | PA7 |
| TFT CS | PB12 |
| TFT DC | PB13 |
| TFT RST | PB14 |
| I2C1 SCL (EEPROM) | PB6 |
| I2C1 SDA (EEPROM) | PB7 |
| On-board LED | PC13 |
| Mark / space LEDs (optional, off by default) | PB0 / PB1 |

### Audio input

PA0 is an ADC input (0 to 3.3 V). Feed it with the receiver's audio through a coupling capacitor and a bias network that centres the signal at about 1.65 V (two resistors to 3.3 V and ground), with a series resistor and clamp diodes if you want to protect the pin. Set the receiver volume so that the signal does not clip; the decoders do not need a high level, and a slightly low level is better than a distorted one.

---

## Building and flashing

### Requirements

- **Arduino IDE** (2.x recommended)
- **STM32duino** core ("STM32 MCU based boards"), installed from the Boards Manager with this URL:
  `https://github.com/stm32duino/BoardManagerFiles/raw/main/package_stmicroelectronics_index.json`
- Libraries (Library Manager):
  - **TFT_eSPI** (Bodmer)
  - **SdFat** (Bill Greiman), version 2.x
- CMSIS-DSP comes with the STM32duino core.

### TFT_eSPI setup

The project needs its own display configuration. Copy `User_Setup.h` from this repository over the one in the TFT_eSPI library folder (or select it in `User_Setup_Select.h`). It defines the pins above, SPI1 at 50 MHz and the fonts used by the interface.

### Board settings

| Setting | Value |
|---------|-------|
| Board | Generic STM32F4 series |
| Board part number | BlackPill F411CE |
| USB support | CDC (generic "Serial" supersedes U(S)ART) |
| Upload method | Your programmer: ST-Link (SWD) or DFU |

Open `HamDecoder.ino`, compile and upload. With the splash screen included the sketch uses about 67 % of the flash (37 % without it) and 58 % of the RAM.

---

## Configuration (`config.h`)

All compile-time options are in `config.h`. The most useful ones:

| Macro | Default | Description |
|-------|---------|-------------|
| `UI_LANG` | `LANG_ES` | On-screen language: `LANG_ES` (Spanish) or `LANG_EN` (English) |
| `SPLASH_ENABLE` | `1` | Start-up splash screen. `0` leaves it (and its ~150 KB image) out of the build |
| `SPLASH_IMAGE` | `"splash_img.h"` | Header with the splash image |
| `SPLASH_MS` | `2500` | Minimum time the splash stays on screen (ms) |
| `MODE_DEF` | `MODE_CW` | Mode at first boot |
| `WF_H_DEF` | `40` | Waterfall height in pixels (20 to 60) |
| `WF_AVG_DEF` | `0.7` | Weight of the new line in the waterfall averaging (1.0 = none) |
| `CW_SQL_DB_DEF` | `11` | Default CW squelch (dB) |
| `RTTY_BAUD_IDX_DEF` | `1` | Default RTTY speed (index into the baud list: 45.45 Bd) |
| `RTTY_USOS` | `1` | Back to letters after a space |
| `IMG_SHOW_SPECTRUM` | `1` | Keep the scale and spectrum above SSTV/WEFAX images (172 px image area; with `0` the area is 206 px) |
| `SD_SAVE_DEF` | `1` | Save received images to the SD card by default |
| `SD_SPI_MHZ` | `12` | SD clock; lower it to 4 if the card gives errors |
| `ENC_DIR` | `1` | Use `-1` to reverse the encoder direction |
| `ENC_LONG_MS` | `600` | Long-press time (ms) |
| `CFG_VERSION` | `6` | Increment it to discard the settings stored in the EEPROM and load the defaults |
| `WF_TEST_SIGNAL` | `0` | `1` = synthetic test signals instead of the ADC (no radio needed) |
| `DEBUG_PERF` | `0` | `1` = show timing measurements in the text window |

Settings stored in the EEPROM take priority over `config.h`. After changing a default, increment `CFG_VERSION` so it takes effect on the next boot.


## User guide

### 1. First start

1. Connect the receiver audio to PA0 and power the board. The splash screen appears for a couple of seconds.
2. The screen shows the parameter bar, the frequency scale, the waterfall and, below it, the text window. The text window prints `Settings loaded from EEPROM` (or says that it is using defaults).
3. Tune the receiver to a signal. You should see it as a vertical trace in the waterfall.

### 2. Screen layout

```
┌──────────────────────────────────────────┐
│ F 1550 Hz  SPAN 3k  CW      22 WPM       │  row 1: essential parameters + status
│ SQL 11     Filter BW        100 Hz       │  row 2: SQL + menu (name and value)
├──────────────────────────────────────────┤
│ frequency scale with tuning mark         │
│ spectrum trace                           │
│ waterfall                                │
├──────────────────────────────────────────┤
│ decoded text / console                   │
└──────────────────────────────────────────┘
```

- **F**: tuning cursor (the audio frequency being decoded), in Hz.
- **SPAN**: width of the visible band: 3k, 2k or 1k.
- **Mode**: WF, CW, RTTY, PSK31, PSK63, SSTV or WEFAX.
- **SQL**: squelch (only in CW, RTTY and PSK).
- **Status** (top right): decoder information, for example `22 WPM` (CW), `SNR 12 dB` (RTTY), `Q 85%` (PSK), `M1 120/256` (SSTV) or `L 85` (WEFAX).
- **Status colour** is the tuning indicator: grey = no signal, orange = signal present, green = decoding, yellow = temporary message.

In SSTV and WEFAX the lower part of the screen shows the received image instead of the text window.

### 3. Controls

The interface uses one rotary encoder with a push button. The highlighted (yellow) field is the one that the knob changes.

| Action | Effect |
|--------|--------|
| **Turn** | Change the selected value (or move through the menu) |
| **Short press** | Select the next essential parameter: FREQ → SPAN → MODE → SQL → MENU → FREQ |
| **Long press** (outside the menu) | Text modes: snap the cursor to the strongest signal. Image modes: manual start / end of the image |
| **Short press in MENU** | Start editing the shown value; press again to accept and go back to the list |
| **Long press in MENU** | Leave the menu and return to FREQ |

Turning quickly accelerates the change (×3 and ×10).

**FREQ** step depends on the span: 10 Hz with 3 kHz, about 7 Hz with 2 kHz and about 3 Hz with 1 kHz. The cursor can go from 100 to 3900 Hz. The waterfall scrolls its view automatically when the cursor gets close to the edge.

### 4. The menu

Select **MENU** with short presses. The second row then shows the name of a parameter and its value. Turn the knob to browse the list, press to edit the value (the value is highlighted), turn to change it, press again to accept. A long press leaves the menu.

The list depends on the mode. The parameters of the active decoder come first, then those of the waterfall.

| Parameter | Range | Modes |
|-----------|-------|-------|
| Filter BW | 50, 100, 200, 400 Hz | CW |
| Baud rate | 45, 45.45, 50, 75, 100 Bd | RTTY |
| Shift | 170, 200, 425, 450, 850 Hz | RTTY |
| Reverse | NO / YES | RTTY |
| AFC | ON / OFF | CW, RTTY, PSK, SSTV |
| SSTV mode | AUTO, M1, M2, S1, S2, SDX, R36, R72, PD50, PD90, PD120, PD180 | SSTV |
| Save to SD | ON / OFF | SSTV, WEFAX |
| Lines/min | 60, 90, 120, 240 | WEFAX |
| IOC | 576, 288 | WEFAX |
| Clock PPM | -3000 to +3000 | WEFAX |
| Phase | 0 to 99 % | WEFAX |
| Threshold | off, 0 to 30 dB | all |
| Reference | -30 to +30 dB | all |
| Range | 10 to 90 dB | all |
| Averaging | 0.1 to 1.0 | all |
| WF height | 20 to 60 px | all |

The **SQL** parameter is an essential one (not in the menu): 0 to 30 dB in CW and RTTY, 0 to 100 % in PSK.

### 5. Waterfall adjustment

These parameters are in the menu and apply to all modes:

- **Threshold**: hides everything weaker than this many dB above the noise floor. `--` disables it. Useful to clean the display.
- **Reference**: brightness. Lower values make weak signals brighter.
- **Range**: contrast, the number of dB covered by the whole colour palette.
- **Averaging**: temporal smoothing. 1.0 means none; lower values smooth the noise but blur short signals.
- **WF height**: height of the waterfall in pixels (the text window gets the rest).

In **WF** mode only the waterfall is active and no decoder runs.

### 6. Operating the decoders

#### CW

1. Select **CW** with MODE.
2. Tune the cursor onto the CW tone. The easiest way: with FREQ selected, press and hold the button to snap the cursor to the strongest signal.
3. Choose the filter width in the menu (**Filter BW**): narrow for crowded bands, wide for drifting signals.
4. Adjust **SQL**: raise it if you see random letters on noise, lower it for weak signals.
5. The decoder adapts to the sending speed; the status shows the measured WPM. With **AFC** on, the cursor follows small drifts.

#### RTTY

1. Select **RTTY**. The cursor marks the **centre** between mark and space (default 2210 Hz for 2125/2295 Hz).
2. In the menu set **Baud rate** (45.45 Bd is the usual amateur speed) and **Shift** (170 Hz usual).
3. If the text is garbage, set **Reverse** to YES (mark and space swapped, depends on the sideband).
4. The status shows the SNR of the two tones; the colour turns green when valid characters arrive.

#### PSK31 / PSK63

1. Select **PSK31** or **PSK63**. The cursor marks the **carrier** (default 1500 Hz).
2. Put the cursor on the signal trace. With **AFC** on the decoder keeps locked to the carrier.
3. The status shows the signal quality `Q`. Text appears when it goes above the **SQL** value (in %).

#### SSTV

1. Select **SSTV**. The cursor marks 1900 Hz (the VIS leader tone), so the signal has to be centred around it.
2. Leave **SSTV mode** on AUTO: the decoder detects the mode from the VIS header (`VIS...` in the status). Force a mode only if the VIS is missed.
3. The status shows `WAITING`, then `M1 120/256` (mode and line count) while receiving. With **AFC** on, the mistuning measured in the header corrects the cursor by itself.
4. If you tune in the middle of an image, press and hold to start manually; hold again to end.
5. When the image is finished the message `SAVED n L` appears (if SD saving is on).

#### WEFAX

1. Select **WEFAX**. The cursor marks 1900 Hz (centre of the 1500 to 2300 Hz range).
2. Set **Lines/min** (120 is the usual weather-chart speed) and **IOC** (576 for charts).
3. The decoder waits for the start tone (`START`), then the phasing signal (`PHASING`), then receives (`L 85`).
4. **Slant correction**: if vertical lines lean to the left going down, lower **Clock PPM**; if they lean to the right, raise it. **Phase** shifts the image horizontally.
5. Press and hold to start or end the image manually.

### 7. Saving images to the SD card

- Format the card as FAT32. Images are saved as BMP in `/SSTV` and `/WEFAX`: `SSTV0001.BMP`, `WFAX0001.BMP`, and so on. The counter is kept in the EEPROM so names do not repeat.
- SSTV images are saved as 24-bit colour, WEFAX as 8-bit greyscale. The text `REC SSTV0001` appears when a file is opened.
- The card is only initialised when you enter SSTV or WEFAX with **Save to SD** ON, and the message `SD READY` or `NO SD CARD` appears. The rest of the time the SD does not touch the bus.
- Switching mode in the middle of an image closes and saves the file.

### 8. Settings are remembered

Every change is stored in the EEPROM about 3 seconds after you stop touching the controls. This includes the mode, the cursor frequency of each mode, the waterfall parameters and all decoder parameters. To return to the factory values, increment `CFG_VERSION` in `config.h` and upload again.

### 9. Serial port

The USB serial port (115200 baud) prints the status messages and, every couple of seconds, a line with performance measurements (lines per second, SPI speed, CPU share of each task). It is useful for diagnostics and can be ignored otherwise.

---

## How it works

```
PA0 ── ADC1 @ 32 kHz (triggered by TIM2) ──DMA2 S0 (circular)──┐
                                                               ▼
              FIR decimator ×4  ──►  8 kHz ring buffer (4096 samples)
                                          │
                  ┌───────────────────────┴──────────────────────────┐
                  ▼                                                  ▼
   Hann window + 1024-point real FFT                      Active decoder
   (hop 256 = ~31 lines/s)                      CW / RTTY / PSK / SSTV / WEFAX
                  ▼                                                  ▼
   Colour palette ──DMA2 S3──► ILI9341 (SPI1 @ 48 MHz)       Text window / image
```

- **Capture**: the ADC is triggered by timer 2 at 32 kHz and fills a buffer by DMA. Each half is decimated by 4 in the interrupt, giving a clean 8 kHz signal. The CPU is never involved in sampling.
- **Waterfall**: the FFT result is converted to a 256-colour palette (stored already in the display byte order) and sent to the display by a private DMA stream, in blocks of 4 rows with double buffering.
- **Decoders**: they read the same 8 kHz ring buffer. CW uses a tone filter with adaptive threshold; RTTY and PSK use matched filters with symbol timing recovery; SSTV and WEFAX use an FM demodulator (mixer, complex FIR and phase-difference frequency measurement) followed by line synchronisation.
- **Shared SPI**: display and SD share SPI1. The SD uses the same SPI object as TFT_eSPI so that neither changes the other's bus speed.
- **Settings**: a packed structure with checksum in the 24LC512, written in small chunks so the writes never block the decoding.

---

## Troubleshooting

| Symptom | What to check |
|---------|---------------|
| Random letters in CW or RTTY with no signal | Raise **SQL**. Make sure the cursor is on the signal: use a long press to snap to it. |
| Nothing decoded but the signal is visible | Cursor not on the signal; wrong filter width, baud rate or shift; RTTY may need **Reverse**. |
| Waterfall is very dark or very bright | Adjust **Reference** and **Range**; lower the receiver volume if the whole display is saturated. |
| Waterfall too busy with noise | Enable **Threshold** (for example 6 to 10 dB) or lower **Averaging**. |
| Distorted decoding on strong signals | Reduce the audio level: the ADC may be clipping. |
| `NO SD CARD` | FAT32 format, card seated, `SD_SPI_MHZ` lowered to 4. |
| `EEPROM not responding: defaults` | Check the I2C wiring (PB6/PB7) and the pull-up resistors. |
| WEFAX image slanted | Adjust **Clock PPM** (see the WEFAX section). |
| SSTV mode not detected | The VIS header was missed: force the mode in **SSTV mode**, or start manually with a long press. |
| Menu texts in the wrong language | Change `UI_LANG` in `config.h`. |
| Encoder turns the wrong way | Set `ENC_DIR` to `-1`. |

---

## Project layout

```
HamDecoder/
├── HamDecoder.ino       Main sketch: setup/loop, parameter bar, encoder handling
├── config.h             All compile-time constants and options
├── lang.h               TR(es, en) macro for the bilingual texts
├── splash.h / .cpp      Start-up splash screen (optional)
├── splash_img.h         Splash image, RGB565 (generated)
├── audio.h / .cpp       ADC + DMA capture, decimation, ring buffer
├── waterfall.h / .cpp   FFT, palette, drawing, DMA transfer to the display
├── decoder.h            Common decoder interface
├── cw_decoder.*         CW decoder
├── rtty_decoder.*       RTTY decoder
├── psk_decoder.*        PSK31/PSK63 decoder
├── fm_demod.*           FM demodulator for the image modes
├── image_decoder.h      Common interface of the image decoders
├── sstv_decoder.*       SSTV decoder
├── sstv_modes.h         SSTV mode table
├── wefax_decoder.*      WEFAX decoder
├── image_view.*         Image drawing on the display
├── image_tx.h           SSTV/WEFAX test-signal generators (for WF_TEST_SIGNAL)
├── sdcard.*             SD card and BMP writer
├── settings.*           EEPROM storage
├── encoder.*            Rotary encoder and button
├── console.*            Decoded-text window
├── dsp_coefs.h          FIR and window coefficients
├── BaudotTable.h        Baudot code table
├── MorseTable.h         Morse code table
├── VaricodeTable.h      PSK varicode table
├── User_Setup.h         TFT_eSPI configuration
├── tools/
│   ├── make_logo.py     Draws the splash logo
│   ├── png2splash.py    Converts an image into splash_img.h
│   └── splash.png       Splash logo
└── LICENSE
```

---

## License

Released under the [MIT License](LICENSE).
Copyright © 2026 Andrés Ortiz, EA7HQL
