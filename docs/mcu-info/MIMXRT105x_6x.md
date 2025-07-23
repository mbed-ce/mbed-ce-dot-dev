# MIMXRT105x/106x MCU Overview
This info page will cover NXP's MIMXRT105x and MIMXRT106x family of MCUs. These MCUs boast some of the highest clock rates of any Mbed-compatible processor on the market, as well as a huge array of capable peripherals. So, as long as you can handle their PCB requirements (complex power arrangement, external flash, large BGA package, etc) they're a very compelling option for your large and high-speed projects!

Mbed OS currently supports the following targets:

- `MIMXRT1050_EVK` [MIMXRT1050 EVK](https://www.nxp.com/design/development-boards/i-mx-evaluation-and-development-boards/i-mx-rt1050-evaluation-kit:MIMXRT1050-EVK), either revision (see below)
- `MIMXRT1060_EVK` [MIMXRT1060 EVK](https://www.nxp.com/design/development-boards/i-mx-evaluation-and-development-boards/i-mx-rt1060-evaluation-kit:MIMXRT1060-EVKB)
- `TEENSY_40` Teensy 4.0, which has an MIMXRT1062.
- `TEENSY_41` Teensy 4.1, which has an MIMXRT1062 + an Ethernet interface (which is fully supported by Mbed)

![MIMXRT1050 EVK Picture](https://www.nxp.com/assets/images/en/dev-board-image/IMX_RT1050-EVKB_TOP-LR.jpg)

## Feature Overview
| CPU | Flash/Code Memory | RAM | Communication Peripherals | Other Features |
|---|---|---|---|---|
| Cortex-M7, clocked at up to 600 MHz | **Total:** 4MiB (off of MCU) | **Total:** 32MiB (external SDRAM) + 512kiB (onboard) + additional 512kiB (onboard, MIMXRT106x only) | <ul><li>4x I2C</li><li>4x SPI</li><li>1x Ethernet (2x in MIMXRT106x)</li><li>8x UART</li><li>2x USB</li><li>2x CAN (3x in MIMXRT106x)</li><li>2x SDIO (currently not supported)</li></ol> | <ul><li>2x 16-input ADC (AnalogIn)</li><li>4x 4-output PWMs (all outputs of each PWM must share a frequency)</li><li>Hardware RNG</li><li>DMA (currently not supported)</li><li>Watchdog</li></ol>|

### Differences between the models
NXP has created a fairly large family of MCUs with these chips, with only some small differences between them. Unfortunately, their docs aren't super good at explaining this. Here's a comparison table I've put together:

| Model | On-Chip Memory | Ethernet Count | XIP Flash Interface <br>(FlexSPI) Count | CAN | LCD Controller, GPU, <br>and Camera Interface |
|---|---|---|---|---|---|
| MIMXRT1051 | 512MiB RAM | 1 | 1 | 2x Regular | No |
| MIMXRT1052 | 512MiB RAM | 1 | 1 | 2x Regular | Yes |
| MIMXRT1061 | 1MiB RAM | 2 | 2 | 2x Regular + 1x CAN FD | No |
| MIMXRT1062 | 1MiB RAM | 2 | 2 | 2x Regular + 1x CAN FD | Yes |
| MIMXRT1064 | 1MiB RAM + 4MiB Flash | 2 | 1 | 2x Regular + 1x CAN FD | Yes |

Basically, the MIMXRT1051 is the base model, and code compatible with it will run on any of the other chips. The other part numbers simply add features on top of that.

!!! note "Temperature Grades"
    There are three different grades of these MCUs: 10xxD for consumer products, 10xxC for industrial products, and 10xxX for extended industrial applications. The D grade allows the full 600MHz frequency but has a limited temperature range. The C grade down-clocks to 528MHz but has a wider temperature range. Similarly, the X grade down-clocks to 500MHz but has an even wider temperature range.

    Currently Mbed targets the "D" grade by default, but can be configured for a "C" grade by setting the `target.enable-overdrive-mode` JSON option to false. "X" grade is not currently supported.

## Memory Layout
The MIMXRT uses a fairly complex memory layout, and the way that the dev kit is set up only adds to this. However, you do gain the flexibility to use huge amounts of space in your Mbed program if you so desire!

### Internal Memory
Internally, the chip contains three RAM banks by default:

- 128kiB Instruction Tightly Coupled Memory (ITCM). This can be used to cache functions and execute them quicker than by running them out of flash. Functions can be placed here by preceding them with the `__attribute__((section("CodeQuickAccess")))` attribute. This will cause the function to be loaded from flash into ITCM at boot, and then executed out of ITCM.
- 256kiB Data Tightly Coupled Memory (DTCM). This can be used to store global data for the fastest possible access by the MCU core. This is currently used for all globals and for the first 
- 128kiB OCRAM. This is a general-purpose memory bank that can be used to store anything. This is used for additional heap and for storing crash data.

Notice that I said "by default". That's because this memory is actually [remappable](https://www.nxp.com/docs/en/application-note/AN12077.pdf)! By changing fuses, you can adjust how the underlying storage is allocated between these three banks. Just remember that if you change the fuses, you'll have to modify the [linker script](https://github.com/mbed-ce/mbed-os/blob/master/targets/TARGET_NXP/TARGET_MCUXpresso_MCUS/TARGET_MIMXRT1050/device/TOOLCHAIN_GCC_ARM/MIMXRT1052xxxxx.ld) and/or the [memory bank configuration](https://github.com/mbed-ce/mbed-os/blob/7fd6fdd79f977815fc78a2d2e8adf9e0dda20638/targets/cmsis_mcu_descriptions.json5#L2767) to compensate...

On top of these banks, the MIMXRT106x devices add an additional, fixed 512MiB OCRAM2 bank to give you even more working area. Mbed uses this instead of OCRAM as the second heap bank when it's available.

The MIMXRT1064, not content with that, includes a flash chip die within the processor package, giving you 4MiB of "internal" flash to work with.

### External Memory
The MIMXRT features lots of connectivity to external memories, and the dev kit does not disappoint in this respect.

As for RAM, the dev kit provides a 32MiB [IS42S16160J-6BLI](https://www.digikey.com/en/products/detail/issi-integrated-silicon-solution-inc/IS42S16160J-6BLI/5319786) RAM chip connected to the SEMC (Smart External Memory Controller) peripheral. Mbed maps this as RAM, providing you a massive amount of space to work with in code.

For flash, the MIMXRT broadly supports two options: standard QSPI NOR flashes, and less standardized OSPI flashes (also referred to as HyperFlashes). The dev kit provides both: a 64MiB OSPI flash, the [S26KS512SDPBHI02](https://www.digikey.com/en/products/detail/cypress-semiconductor-corp/S26KS512SDPBHI020/6057101), and an [IS25WP064AJBLE](https://www.digikey.com/en/products/detail/issi-integrated-silicon-solution-inc/IS25WP064A-JBLE/7221227) 8MiB QSPI flash. On the 1052 dev kit, the OSPI flash is connected by default to the MCU's FlexSPI peripheral, and if you want to switch you'll have to move some resistors around on the PCB. See the MIMXRT1050-EVKB User Guide for details. On the 1062 dev kit, the parts are the same but the QSPI flash is used by default instead.

!!! note "Note about Names"
    Don't get confused! FlexSPI is the MIMXRT's interface for talking to SPI flashes. LPSPI is the peripheral for talking to regular SPI devices. And, last but not least, FlexIO is a configurable peripheral that can emulate, among other things, an SPI bus.

#### SDRAM Issues (as of Jan 2025)
Due to #413, we are disabling the SDRAM on MIMXRT EVKs until we can debug memory corruption issues affecting it. It will still be mapped in memory starting at address 0x80000000, but the linker script will not store anything there.

Even once this issue is fixed, there is another potential problem affecting the SDRAM: it seems like the debugger has difficulty accessing thread stacks that are stored in SDRAM. This means that, at present, if you allocate a new thread whose stack ends up in SDRAM, you won't be able to view the stack trace or local variables.

### Memory Configuration
But what is it that makes these memory peripherals "flexible" and "smart", you ask? Why, that's because they can actually be reconfigured at runtime, before the code even boots!

The FlexSPI configuration data is stored in the first 512 bytes of the flash device (see MIMXRT reference manual section 9.6.1.2 for details). It contains the basic flash size and width information, as well as the command sequences that the hardware will use to start up the flash and read data from it. This structure is stored [here](https://github.com/mbed-ce/mbed-os/blob/master/targets/TARGET_NXP/TARGET_MCUXpresso_MCUS/TARGET_MIMXRT1050/TARGET_EVK/xip/evkbimxrt1050_flexspi_nor_config.c) in the Mbed code, and it is placed by the linker into the very start of flash.

On RT105x, if you want to switch your MCU to use the QSPI flash, you'll need to remove the `HYPERFLASH_BOOT` define using a custom target, which will cause the QSPI version of the flash config data to be used instead. On RT106x, the QSPI flash is used by default so you will need to define HYPERFLASH_BOOT in your project to use the HyperFlash.

The SEMC configuration, meanwhile, is stored in a different section of the program: the DCD, or Device Configuration Data. This structure basically specifies an arbitrary series of memory writes that the MCU will execute before starting the program, and is usually used to configure the RAM and/or do other low-level setup. The DCD is stored as C code [here](https://github.com/mbed-ce/mbed-os/blob/master/targets/TARGET_NXP/TARGET_MCUXpresso_MCUS/TARGET_MIMXRT1050/TARGET_EVK/xip/evkbimxrt1050_sdram_ini_dcd.c), and can be generated/modified using NXP's MCUXpresso Config Tools application. As before, the linker script is configured to place this data into the first part of flash so that the MCU can find and execute it.

## Deep Sleep Issues
On the MIMXRT, deep sleep prevents the CPU core from being accessed over the SWD/JTAG interface. This means that while the core is in deep sleep, the MCU cannot be flashed or debugged. If deep sleep were enabled, it would be possible to soft brick your board with code that deep sleeps for a while right after boot, like:
```
#include <mbed.h>

void main()
{
  rtos::ThisThread::sleep_for(10s);
  <do other stuff...>
}
```

Since PyOCD is configured to always reset the chip before trying to connect (to work around a _different_ issue), it will always be in deep sleep when the debugger tries to connect to it, leading to failure to connect.

Due to this issue providing an absolutely awful user experience, and the fact that the MIMXRT is not commonly used in low power applications anyway, deep sleep has been disabled by default on MIMXRT105x as of Jan 2025. If you wish to reenable deep sleep, and you understand that you will need to carefully control when deep sleep is enabled, you may do so by creating a custom target with the following in its JSON definition:
```js
    "device_has_add": [
      "LPTICKER"
    ],
```

## Soft Bricks and Recovery
It's very possible to get the MIMXRT into a state where code is doing something (e.g. trying to access an unpowered peripheral) that locks up the chip early in the boot process. This will prevent you from programming, debugging, or erasing the target board normally.

To get out of this state, you will need to set the chip to boot into Serial Downloader mode. On the official EVK boards, this can be done using the DIP switches on the board (see the EVK manual for the specific switches and positions). Once you reboot into serial downloader mode, the problematic code will no longer be running, so you can flash new code as desired.

## Other Notes
### Timer Usage
Mbed OS uses all four channels of the Periodic Interrupt Timer (PIT) peripheral to implement the microsecond ticker. It also uses the General Purpose Timer (GPT) 2 peripheral to implement the low-power ticker on devices that support it. These peripherals are therefore unavailable for user code to use.

### GPIO Naming
The MIMXRT uses a very unusual naming scheme for its GPIO pins, where pins are named according to their primary function:

- GPIO_AD_xx pins contain ADC inputs and the analog comparators
- GPIO_EMC_xx pins contain SEMC (the external RAM controller) inputs and outputs
- GPIO_SD_xx pins contain SDIO inputs and outputs
- GPIO_Bxx is the catch-all IO port and doesn't have a specific theme

I appreciate what they were going for here, but I feel like it can make things more confusing by forcing you to remember a more complicated name...

### Dev Board Differences
There are substantial differences between Revs A and B of the MIMXRT1050_EVKB dev board. Rev A uses an OpenSDA probe, which is CMSIS-DAP only and cannot be converted to a J-Link, while Rev B uses an LPC-Link which can be. 

Additionally, the mapping for the Arduino-pinout I2C pins (D14 and D15) changed. On the Rev A dev kit:

- D14 and D15 are mapped to GPIO_AD_B0_01 and GPIO_AD_B0_00, which do not support I2C. Oops!
- If you need I2C, you'll need to use D0=SCL and D1=SDA or A5=SCL and A4=SDA instead.

On the Rev B dev kit (and the MIMXRT1060 EVK):

- D14 and D15 are mapped to GPIO_AD_B1_01 and GPIO_AD_B1_00, which do support I2C, but are also the same signals as A4 and A5.
 - This mapping is what's in the Mbed headers

Note: in fall 2022 I ordered a brand new dev board from NXP, and received a Rev A, not a Rev B. So, the Rev As are definitely still out there...

## Datasheets
- MIMXRT1051/52 Datasheet: [IMXRT1050CEC.pdf](https://www.nxp.com/docs/en/data-sheet/IMXRT1050CEC.pdf)
- MIMXRT1061/1062 Datasheet: [IMXRT1060XCEC.pdf](https://www.nxp.com/docs/en/data-sheet/IMXRT1060XCEC.pdf)
- MIMXRT1064 Datasheet: [IMXRT1064CEC.pdf](https://www.nxp.com/docs/en/data-sheet/IMXRT1064CEC.pdf)
- MIMXRT105x Programmmer's Reference Manual: [MIMXRT1050RM.pdf](https://www.nxp.com/webapp/Download?colCode=IMXRT1050RM) (registration required)
- MIMXRT1050-EVKB User Guide: [MIMXRT1050EVKBHUG.pdf](https://www.nxp.com/webapp/Download?colCode=MIMXRT1050EVKBHUG) (registration required)
- MIMXRT1050-EVKB Design Files Rev B1 [RT1050EVKB-DESIGNFILES-B1.zip](https://www.nxp.com/webapp/Download?colCode=RT1050EVKB-DESIGNFILES-B1&appType=license)
- MIMXRT1050-EVKB Schematic Rev A1 [SPF-30168_A1.pdf](https://community.nxp.com/pwmxy87654/attachments/pwmxy87654/mcuxpresso-ide/8031/1/SPF-30168_A1.pdf)