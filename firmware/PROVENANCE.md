---
type: Reference Table
title: Provenance of the Probe Firmware
description: The inputs, the flags and the checksum that produced debugprobe_on_pico2.uf2.
status: draft
tags: [rp2350, debugprobe, provenance, build]
generated:
  by: claude-code/opus-5
  at: 2026-08-16T00:30:00Z
supervised:
  by: human:ciprian-florin_ifrim
  at: 2026-08-16T00:30:00Z
---

# Provenance of the Probe Firmware

**A binary with no record of its inputs cannot be trusted and cannot be repeated.** This file is the
record. Somebody who reads it in a year can say what is in the file, and can build the same bytes
again.

## The Artifact

    file         debugprobe_on_pico2.uf2
    size         103424 bytes
    sha256       98b732e0397ebe892ec7374019019eb7c841ed1de6a5002c06e44cf020e5eebe
    built        2026-08-15
    host         macOS, arm64

## The Inputs

    debugprobe   3fff5b240ca8200c7ad538cb61c02dfc39bda831
    pico-sdk     98a542c1a62fb549ffb5d66a3e5892b06276b670    (tag 2.3.0)
    tinyusb      86ad6e56c1700e85f1c5678607a762cfe3aa2f47    (submodule of pico-sdk)
    toolchain    arm-none-eabi-gcc 15.3.rel1, ArmGNUToolchain

The FreeRTOS submodules of `debugprobe` come with the recursive checkout. The RP2350 needs the
downstream port, which is why the checkout must be recursive and not shallow of the top level alone.

## The Flags

    cmake -G Ninja \
          -DDEBUG_ON_PICO=1 \
          -DPICO_BOARD=pico2 \
          -DPICO_PLATFORM=rp2350 \
          -DPICO_TOOLCHAIN_PATH=/Applications/ArmGNUToolchain/15.3.rel1/arm-none-eabi

**`DEBUG_ON_PICO=1` selects the pin set, and it is the flag that matters.** It makes the build
include `include/board_pico_config.h` rather than `include/board_debug_probe_config.h`, and those two
files give different pins. The second one is for the Debug Probe product of Raspberry Pi and it
carries no reset pin at all.

    PROBE_PIN_OFFSET   2
    PROBE_PIN_SWCLK    2
    PROBE_PIN_SWDIO    3
    PROBE_PIN_RESET    1
    PROBE_UART_TX      4
    PROBE_UART_RX      5

**No source file was changed.** The XIAO RP2350 runs the stock `pico2` build because it matches the
Pico 2 on each of those five pins, and on the LED at GP25.

## To Repeat It

    git clone https://github.com/raspberrypi/debugprobe.git
    cd debugprobe
    git checkout 3fff5b240ca8200c7ad538cb61c02dfc39bda831
    git submodule update --init --recursive

    git clone https://github.com/raspberrypi/pico-sdk.git
    cd pico-sdk
    git checkout 98a542c1a62fb549ffb5d66a3e5892b06276b670
    git submodule update --init lib/tinyusb

    cd ../debugprobe && mkdir build-pico2 && cd build-pico2
    PICO_SDK_PATH=../../pico-sdk cmake -G Ninja \
        -DDEBUG_ON_PICO=1 -DPICO_BOARD=pico2 -DPICO_PLATFORM=rp2350 \
        -DPICO_TOOLCHAIN_PATH=/Applications/ArmGNUToolchain/15.3.rel1/arm-none-eabi ..
    ninja
    shasum -a 256 debugprobe_on_pico2.uf2

**The `arm-none-eabi-gcc` of Homebrew cannot link this.** That formula carries a compiler with no C
library, so the link stops at `cannot find -lc` and `cannot find -lg` in the boot stage. The
`PICO_TOOLCHAIN_PATH` above avoids it. The same trap catches any bare metal Arm build on that host,
and not this one alone.

The build fetches picotool from source if the host has none. That is a warning and not a fault.
