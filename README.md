---
type: Repository Guide
title: RP2350 as a Debugger
description: Guide that turns an RP2350 board into a CMSIS-DAP SWD probe for other microcontrollers.
status: stable
tags: [rp2350, xiao, cmsis-dap, swd, openocd, debugprobe]
generated:
  by: claude-code/opus-5
  at: 2026-08-16T00:30:00Z
supervised:
  by: human:ciprian-florin_ifrim
  at: 2026-10-02T17:54:03Z
edited:
  by: claude-code/opus-5.5
  at: 2026-10-02T18:04:52Z
---

# RP2350 as a Debugger

An RP2350 board becomes a CMSIS-DAP probe over USB. It then programs and debugs another
microcontroller through SWD, OpenOCD drives it, and the same cable carries a serial port for the
output of the target.

This is for the person who holds an RP2350 board and no ST-Link.

**There is no code in this repository.** The firmware is a build of
[raspberrypi/debugprobe](https://github.com/raspberrypi/debugprobe) with the stock `pico2` settings
and no change of any kind. What is here is the binary, the wiring, and the facts that cost a day to
establish.

## 1. Flash and Wire

Hold BOOTSEL, connect the USB-C cable, and release. A volume with the name `RP2350` appears.

    cp firmware/debugprobe_on_pico2.uf2 /Volumes/RP2350/     put the probe firmware on the board
    system_profiler SPUSBDataType | grep -iA6 "CMSIS-DAP"    the probe
    ls /dev/cu.usbmodem*                                     the serial bridge on the same cable

The board restarts itself when the copy finishes. **The volume disappears, and that is the signal of
success and not an error.**

Five signals reach the target, and the firmware fixes each pin:

    signal              RP2350      XIAO RP2350 pad
    SWCLK               GP2         D8
    SWDIO               GP3         D10
    nRESET              GP1         D7
    UART TX of probe    GP4         D9
    UART RX of probe    GP5         D3
    GND                 --          GND

**Three wires are enough to start.** SWDIO, SWCLK and GND connect to a part and halt it. Add the
reset and the serial port after that works, because three wires is a smaller place to look for a
fault than six.

**Give the target its own power.** The probe does not feed it. An unpowered part answers
`Error connecting DP: cannot read IDR`, and that message reads like a fault in the wiring.

## 2. Directory Tree

    firmware/     the built probe firmware, and the provenance that lets somebody rebuild the
                  same bytes. PROVENANCE.md holds the commit of each input, the flags, and the
                  checksum of the output.
    openocd/      the configuration of one target, as an example of the shape. See section 3.

**The scope of this repository is the probe.** A target microcontroller is out of scope. The
STM32H723 in `openocd/` is one worked example, and it is there to show the shape of a target
configuration and to prove that the probe reaches a real part.

To place a new file, ask which side of the USB cable it describes. A file about an RP2350 board
belongs here. A file about the part on the other end does not, and a repository that collects them
stops being about the RP2350 within a month.

## 3. Boards That Work

    board                       firmware                        state
    Seeed XIAO RP2350           the pico2 build, no change      verified on hardware
    Raspberry Pi Pico 2         the same binary                 the target of upstream
    Raspberry Pi Pico, RP2040   -DPICO_BOARD=pico               one flag away, not tested here

**The XIAO needs no board file, because it already matches the Pico 2 where it matters.** GP1 to GP5
each reach a pad on the castellated header, and the LED sits on GP25 for both. Upstream ships a
configuration for `pico` and for `pico2` and for nothing else, so a board that differs needs one.

To add an RP2350 board, read the schematic of the vendor for the net labels of the form
`GPIOn/Dm/function`. Confirm that the five signals of section 1 each reach a pad, and that none of
them already carries something the board needs.

## 4. The Worked Example: WeAct MiniSTM32H723

The board prints the port letter without the `P`. `A9` on the silkscreen is `PA9` in the datasheet,
`NR` is `NRST`, and only port A takes part.

    XIAO pad    WeAct silkscreen                STM32 pin
    D10         SWDIO, on the 4-pin header      PA13
    D8          SWCLK, on the 4-pin header      PA14
    GND         GND, on the 4-pin header        ground
    D7          NR, on the 22x2 header          NRST
    D9          A10                             PA10, the pin that receives
    D3          A9                              PA9, the pin that transmits

The 4-pin header reads `3.3V | SWDIO | SWCLK | GND`, and the `3.3V` pin stays open because the part
takes its own supply. The serial port crosses over, so the pin that transmits on the probe goes to
the pin that receives on the part.

Both SWD lines already carry 22 ohm in series on the board. Add nothing.

### 4.1 What This Reached

    SWD DPIDR                    0x6ba02477
    core                         Cortex-M7 r1p2
    part                         STM32H72x/73x, 1024 KiB of flash
    read of 128 KiB              237 KiB/s
    write of a 128 KiB sector    53 KiB/s
    program and verify           1.8 s, the connect included
    nRESET                       running -> reset -> running

The read is correct at every clock from 4 MHz to 25 MHz, and the three files carry one checksum.
**The rate stops to increase above 15 MHz**, because the limit is then the round trip of USB and the
overhead of a packet, and no longer the SWD clock. A higher number gives 3 KiB/s more and takes the
margin away, so the configuration file uses 15000.

## 5. Driving OpenOCD

    openocd -f interface/cmsis-dap.cfg -c "transport select swd" \
            -f target/stm32h7x.cfg -f openocd/stm32h723-weact.cfg \
            -c "init" -c "reset halt" -c "mdw 0x08000000 4" -c "shutdown"

To flash and to prove what arrived:

    openocd -f interface/cmsis-dap.cfg -c "transport select swd" \
            -f target/stm32h7x.cfg -f openocd/stm32h723-weact.cfg \
            -c "program firmware.elf verify reset exit"

**A loop that flashes many images keeps one process alive instead.** Each start costs about 0.3 s
for the connect, which is a third of the time of a 128 KiB transfer and most of the time of a small
one.

    openocd -f interface/cmsis-dap.cfg -c "transport select swd" \
            -f target/stm32h7x.cfg -f openocd/stm32h723-weact.cfg \
            -c "tcl_port 6666" -c "init" -c "reset halt"

Then drive port 6666. `mdw`, `mdb`, `dump_image`, `load_image`, `reset halt` and `resume` all answer
there, and no GDB is necessary.

## 6. What Cost Time

**`adapter speed` must come after the target file.** A vendor target file sets its own speed and
overrides anything before it. OpenOCD gives no warning. It prints the speed that it uses, so read
that line. A run that asked for 100 kHz went at 4000 and said so in one line that is easy to pass
over.

**Use `reset halt` and never a bare `halt`.** With `connect_assert_srst` the part comes up held in
reset. A bare `halt` then waits out its whole timeout and reports `external reset detected`.

**The pin report of the probe means nothing.** This line appears in every session:

    Info : SWCLK/TCK = 0 SWDIO/TMS = 0 TDI = 0 TDO = 0 nTRST = 0 nRESET = 0

`PIN_SWCLK_TCK_IN` and `PIN_SWDIO_TMS_IN` in `debugprobe/include/DAP_config.h` both return `0`. The
values are zero whatever the pins do. It is not a diagnostic and it looks exactly like one.

**`cannot read IDR` is usually the power of the target.** Test the cable of the target first, the
ground second, and the order of SWDIO and SWCLK third. The message names the debug port, so it sends
a reader to the wiring of the SWD lines, which is the third thing to test and not the first.

**The reset is an open drain, so it is safe against a supervisor.** `probe_assert_reset` drives the
pin low as an output, and returns it to an input to release it. It never drives the pin high. The
WeAct board puts 1.5 kilohm in series with its supervisor and shorts the same net to ground with the
reset button, so the design already expects a probe to hold that net low.

**Erratum RP2350-E9 looks fatal for the reset line, and it is not.** A released reset pin is an input
with the output disabled, which is the exact condition of the erratum. The text of the erratum names
its own answer: "The pad pull-up still works. If enabled it will pull the pad to IOVDD as it will
pull the input voltage out of the problematic range." The firmware enables that pull-up.

**Do not read memory during a run that you time.** OpenOCD halts the core to read it, and a cycle
counter does not count while the core is halted. Every figure after such a read is wrong, and the
run still finishes and still looks correct. Read the serial port instead, which this probe gives on
the same cable.

## 7. Limits

**There is no SWO.** `SWO_UART`, `SWO_MANCHESTER` and `SWO_STREAM` are each `0` in this firmware, so
there is no ITM trace. Use RTT over SWD, which needs no more pins, or the serial port.

**The transport is USB bulk only.** This is CMSIS-DAP v2 and the HID backend is absent. OpenOCD and
probe-rs accept it. A tool that speaks only HID does not.

## 8. Building the Firmware

You do not need to, because `firmware/` holds the same bytes. To repeat the build:

    git clone https://github.com/raspberrypi/debugprobe.git
    cd debugprobe && git submodule update --init --recursive
    mkdir build-pico2 && cd build-pico2
    cmake -G Ninja -DDEBUG_ON_PICO=1 -DPICO_BOARD=pico2 -DPICO_PLATFORM=rp2350 \
          -DPICO_TOOLCHAIN_PATH=/Applications/ArmGNUToolchain/15.3.rel1/arm-none-eabi ..
    ninja

The Pico SDK must be 2.0.0 or newer, and the RP2350 build needs the FreeRTOS submodule.

**The `arm-none-eabi-gcc` of Homebrew cannot link this.** That formula carries no newlib, so the
link stops with `cannot find -lc`. Give CMake the ArmGNUToolchain, as the line above does.

`firmware/PROVENANCE.md` holds the commit of each input and the checksum of the output.

## 9. License

The firmware is the work of Raspberry Pi and it keeps the MIT license of that project. See
[LICENSE-debugprobe](LICENSE-debugprobe).
