# HamDecoder

Digital-mode decoder for ham radio, running on an **STM32F411 Black Pill** with a 2.4" ILI9341 TFT display, a rotary encoder and a 24LC512 EEPROM. Built with the Arduino IDE + STM32duino.

**Author:** Andrés Ortiz, EA7HQL  
**License:** MIT

---

## Features

- **SDR-style colour waterfall** — 1024-point FFT, 31 lines/s, 3 zoom spans (1/2/3 kHz), adjustable brightness and contrast, spectrum trace
- **Decoders:** CW (Morse), RTTY (Baudot), PSK31, PSK63, SSTV (13 modes, auto-VIS), WEFAX
- **Automatic Frequency Control (AFC)** for CW, RTTY and PSK
- **Image save to microSD card** — SSTV and WEFAX images saved as BMP
- **Persistent settings** in 24LC512 EEPROM (survives power cycles)
- **Bilingual UI** — English or Spanish selected at compile time (`UI_LANG` in `config.h`)
- **Large-font parameter bar** with two rows:
  - Row 1: frequency cursor, span, mode, decoder status (colour-coded)
  - Row 2: squelch + scrollable MENU for all secondary parameters
- **Rotary encoder navigation:**
  - Short press: cycle through essential parameters (FREQ → SPAN → MODE → SQL → MENU)
  - Rotate in MENU: browse secondary parameters
  - Short press in MENU: edit the selected parameter / confirm
  - Long press in MENU: return to essential parameters
  - Long press outside MENU: snap cursor to the strongest peak (text modes) or manual start/stop (image modes)

## Hardware

| Component | Part |
|-----------|------|
| MCU | STM32F411CEU6 "Black Pill" @ 96 MHz |
| Display | 2.4" ILI9341 TFT 320×240 (SPI1) |
| Encoder | Rotary encoder with push button |
| EEPROM | 24LC512 (I2C1, PB6/PB7) |
| microSD | Shared SPI1 (CS = PA4) |
| Audio in | 3.5 mm jack → PA0 (ADC1_IN0) |

### Pin assignment

| Signal | Pin |
|--------|-----|
| Audio input | PA0 (ADC1_IN0) |
| Encoder CLK | PA1 |
| Encoder DT | PA2 |
| Encoder SW | PA3 |
| SD CS | PA4 |
| SPI1 SCK | PA5 |
| SPI1 MISO | PA6 |
| SPI1 MOSI | PA7 |
| I2C1 SCL | PB6 |
| I2C1 SDA | PB7 |
| Mark LED (optional) | PB0 |
| Space LED (optional) | PB1 |

## Building

### Prerequisites

- **Arduino IDE** 2.x (or 1.8.x)
- **STM32duino** board package (Boards Manager → `https://github.com/stm32duino/BoardManagerFiles/raw/main/package_stmicroelectronics_index.json`)
- Libraries (via Library Manager):
  - **TFT_eSPI** (Bodmer) — configure with `User_Setup.h` included in this repo
  - **SdFat** (Bill Greiman) — version with `SHARED_SPI` support
  - **CMSIS-DSP** — included with STM32duino

### Board settings

```
Board:       Generic STM32F4 Series → STM32F411CEUx
Speed:       96 MHz
Optimize:    -Os (Smallest)
USB support: CDC (generic Serial, no restart needed)
Upload:      STM32CubeProgrammer (DFU) or SWD
```

### Compile-time options (`config.h`)

| Macro | Default | Description |
|-------|---------|-------------|
| `UI_LANG` | `LANG_ES` | On-screen language: `LANG_ES` (Spanish) or `LANG_EN` (English) |
| `WF_H_DEF` | `40` | Default waterfall height in pixels |
| `WF_AVG_DEF` | `0.7` | Waterfall time averaging (0 = instant, 1 = maximum) |
| `DEBUG_PERF` | `0` | Set to `1` to overlay timing measurements |
| `WF_TEST_SIGNAL` | `0` | Set to `1` to use a synthetic test signal instead of the ADC |
| `SD_SAVE_DEF` | `1` | Save received images to SD by default |
| `USE_PHYS_LEDS` | `0` | Enable mark/space LEDs on PB0/PB1 |
| `CFG_VERSION` | `6` | Increment to reset EEPROM settings to defaults |

## DSP pipeline

```
PA0 (ADC @ 32 kHz) → DMA circular buffer (512 samples)
    → FIR decimator ×4 (arm_fir_decimate_f32) → 8 kHz ring buffer (4096 samples)
    → Hann window → 1024-point rfft (CMSIS) → magnitude → waterfall line
                                                           → decoder (CW / RTTY / PSK / SSTV / WEFAX)
```

The FFT hop is 256 samples (31.25 lines/s). The waterfall DMA uses a private register-level driver on DMA2 Stream3 to avoid conflicts with HAL.

## Decoder modes

| Mode | Cursor marks | Squelch |
|------|-------------|---------|
| CW | Tone centre | Peak/noise ratio (dB) |
| RTTY | Centre between mark and space | SNR (dB) |
| PSK31 / PSK63 | Carrier | Signal quality (%) |
| SSTV | 1900 Hz (VIS leader) | — |
| WEFAX | 1900 Hz (centre) | — |

### RTTY

Selectable baud rates: 45, 45.45, 50, 75, 100 Bd. Shifts: 170, 200, 425, 450, 850 Hz. Normal and reversed polarity.

### SSTV

Supports 13 modes (Martin 1/2, Scottie 1/2/DX, Robot 36/72, PD90/120/160/180/240, Wrasse SC2-180). Mode detected automatically from VIS code; can also be forced.

### WEFAX

576 and 288 IOC. Speeds: 60, 90, 120, 240 LPM. Clock correction (PPM) and phase offset adjustable from the MENU.

## File structure

```
WaterfallSDR/
├── WaterfallSDR.ino    Main sketch: setup, loop, encoder, parameter bar
├── config.h            All compile-time constants
├── lang.h              TR(es,en) macro for bilingual texts
├── audio.h/cpp         ADC + DMA capture, decimation, ring buffer
├── waterfall.h/cpp     FFT, palette, TFT drawing, SPI DMA driver
├── cw_decoder.h/cpp    CW (Morse) decoder with AFC
├── rtty_decoder.h/cpp  RTTY (Baudot FSK) decoder with AFC
├── psk_decoder.h/cpp   PSK31/PSK63 BPSK decoder with AFC
├── sstv_decoder.h/cpp  SSTV image decoder
├── wefax_decoder.h/cpp WEFAX image decoder
├── fm_demod.h/cpp      FM narrow-band demodulator (shared by RTTY/SSTV/WEFAX)
├── image_view.h/cpp    TFT image rendering
├── sdcard.h/cpp        SdFat32 wrapper, BMP writer
├── settings.h/cpp      EEPROM persistence (24LC512)
├── encoder.h/cpp       Rotary encoder (TIM4 sampling)
├── console.h/cpp       Text output to TFT
├── decoder.h           Abstract Decoder interface
├── image_decoder.h     Abstract ImageDecoder interface
├── sstv_modes.h        SSTV mode table
├── dsp_coefs.h         FIR decimator coefficients
├── BaudotTable.h       Baudot code table
├── MorseTable.h        Morse code table
├── VaricodeTable.h     PSK varicode table
├── User_Setup.h        TFT_eSPI configuration
└── LICENSE             MIT licence
```

## Licence

MIT — see [LICENSE](LICENSE).  
Copyright © 2026 Andrés Ortiz, EA7HQL
