# USB Mass Storage SD Card Reader

A USB composite device for the **ATmega32U4** that exposes a physical SD card to a
host computer as a standard USB Mass Storage device, alongside a USB CDC virtual
serial port used for status logging and debugging. Built on the
[LUFA](https://github.com/abcminiuser/lufa) USB stack with a custom bare-metal SD
card driver, a FAT filesystem layer, and an OLED status display.

## Overview

The firmware enumerates as a **CDC + Mass Storage composite device**. When plugged
in, the host sees a removable drive backed by the SD card's raw blocks, plus a
virtual COM port for diagnostics. SCSI READ/WRITE commands from the host are mapped
directly onto SD card block reads and writes over SPI.

This project implements a SPI SD card driver from scratch implementing setup commands and read and write commands.

## Features

- **USB Mass Storage (BBB/SCSI)** — the SD card appears as a removable USB drive
- **USB CDC virtual serial port** — runtime status and debug output to the host
- **Custom SD/SPI driver** (`Lib/SDcard.c`) — implements the SPI-mode init sequence
  (CMD0 → CMD8 → ACMD41 → CMD58 → CMD16), SDHC/SDSC detection via the OCR register,
  and single/multi-block reads and writes
- **FAT filesystem access** via [Petit FatFs](http://elm-chan.org/fsw/ff/00index_p.html)
- **OLED status display** driven by [u8g2](https://github.com/olikraus/u8g2)
- **Watchdog reset fix** (`wdt_fix.c`) to recover cleanly from bootloader resets

## Hardware

| | |
|---|---|
| MCU | ATmega32U4 |
| Board profile | MICRO (LUFA `BOARD`) |
| Clock | 16 MHz |
| SD interface | SPI |
| Display | I²C OLED (u8g2-compatible) |

The SD card is wired to the hardware SPI pins with a dedicated chip-select line.
See `SD_SPINotes.txt` for notes on the SPI-mode entry sequence and SCSI read/write
flow.

There is also a PCB project which is linked to this firmware project aswell. It only adds a couple of buttons and LEDS to indicate the status of the board and for user input. 

## Architecture

```
Host (USB)
   │
   ├── CDC  ── virtual serial (logging/debug)
   │
   └── MSC  ── SCSI commands
                  │
              Lib/SCSI.c ──► Lib/DataflashManager.c
                                     │
                              Lib/SDcard.c  (SD protocol over SPI)
                                     │
                                Lib/SPI.c   (AVR8 hardware SPI)
```

Petit FatFs (`PetitFs/`) sits on top of the SD block driver for filesystem-level
access, and `Lib/display.c` renders status to the OLED.

## Project structure

```
main.c                  USB task loop + device configuration
Descriptors.c/.h        USB descriptors (CDC + Mass Storage composite)
Config/                 LUFA + application configuration headers
Lib/
  SDcard.c/.h           Custom SD card driver (SPI mode)
  SPI.c/.h              Low-level AVR8 SPI
  SCSI.c/.h             SCSI command handling for Mass Storage
  DataflashManager.c/.h Block-transfer glue between SCSI and the SD card
  display.c/.h          OLED status output (u8g2)
PetitFs/                Petit FatFs filesystem (third-party)
wdt_fix.c               Watchdog disable on startup
makefile                LUFA/DMBS build configuration
```

## Building

This project uses git submodules for LUFA and u8g2, and the AVR toolchain.

```bash
# Clone with submodules
git clone --recurse-submodules https://github.com/JYSCN/massmediadevice-proj.git
cd massmediadevice-proj

# Or, if already cloned:
git submodule update --init --recursive

# Install the AVR toolchain (Debian/Ubuntu/EndeavourOS examples)
#   Debian/Ubuntu:   sudo apt install gcc-avr avr-libc avrdude
#   Arch/EndeavourOS: sudo pacman -S avr-gcc avr-libc avrdude

# Build
make
```

This produces `main.hex` (and `main.elf`).

## Flashing

Flash with your programmer of choice. Example using `avrdude`:

```bash
# Adjust -c (programmer) and -P (port) to your setup
make avrdude
# or directly:
avrdude -c avr109 -p atmega32u4 -P /dev/ttyACM0 -U flash:w:main.hex
```

## Usage

1. Insert a FAT-formatted SD card.
2. Connect the device over USB.
3. The host mounts the SD card as a removable drive; a virtual serial port also
   appears for status output.

## Credits

- USB stack: [LUFA](https://github.com/abcminiuser/lufa) by Dean Camera
- Filesystem: [Petit FatFs](http://elm-chan.org/fsw/ff/00index_p.html) by ChaN
- Display: [u8g2](https://github.com/olikraus/u8g2) by olikraus
