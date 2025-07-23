# LPC546xx MCU Overview

![FF_LPC546xx board](https://www.l-tek.com/wp-content/uploads/2018/04/L-Tek-FF-LPC546xx.png)

The LPC546xx is one of NXP's more direct successors to the classic line of LPC17xx microcontrollers released 8 years earlier. The LPC546xx keeps many of the best features of the LPC17xx, including the simple, no-nonsense datasheet, the wide 32-bit timers, and the capable ROM bootloader. However, it also modernizes the chip with much lower power usage (~5x better), a newer core (Cortex-M4) running at a higher core freq (~2x faster), lots more RAM, more flexible serial communications, and a wider variety of peripherals.

While I have not seen this family of chips on very many boards, I don't really understand why -- it seems like a worthy successor to the LPC17xx line, able to easily complete in functionality with popular MCU lines like STMicro STM32F4 and Freescale/NXP K64F. Perhaps adding support for more of its features in Mbed would give it more of a fair chance...

## Feature Overview
| CPU | Flash/Code Memory | RAM | Communication Peripherals | Other Features |
|---|---|---|---|---|
| Cortex-M4, up to 180/220MHz depending on model| 512 kiB flash (on most MCU models), 16 kiB EEPROM | 160 kiB main SRAM (on most MCU models), 32 kiB SRAMX bank (usable as extra heap), 8 kiB USB RAM | <ul><li>10x Flexcomm modules, which can become SPI, I2C, or UART (note: only some pinned out on dev boards)</li><li>1x QSPI</li><li>2x CAN (not supported by Mbed)</li><li>2x USB (not supported by Mbed)</li><li>1x Ethernet</li></ol> | <ul><li>1x ADC (AnalogIn), 12 multiplexed inputs</li><li>True RNG</li><li>5x 32-bit hardware timers (CTIMER1 used by Mbed)</li></ol>|

## FF_LPC546xx Dev Board Considerations
This board replicates the form factor of the Mbed LPC1678 board, and may work as a drop-in replacement in some applications. I have tested Mbed on this board, and it seems to work well. However, there are some considerations to be aware of: Mbed OS currently does not support USB, CAN, or PWM for LPC546xx, so these functions on the board are not available, despite being mentioned in the pinout. All of these are possible to implement, just not implemented in the current target layer. PRs would be gratefully accepted to fix this!

Additionally, this board has an onboard DataFlash memory chip, but this memory chip is not correctly connected to the MCU SPI peripheral, so it cannot be used.

## Upload Methods
Currently, the FF_LPC546xx board uses the Mbed USB disk upload method by default. This works for simple applications by default, but does not provide debugging. For those needing debugging support, the `PYOCD` and `LINKSERVER` upload methods are available. Both are functional, however PyOCD has issues with loading code or resetting the MCU during a debug session. For this reason, it's recommended to use LINKSERVER for the best debugging experience.

## Datasheets
- [FF_LPC546xx dev board schematic](https://www.l-tek.com/wp-content/uploads/2019/05/mbed_002_v1.1.pdf)
- [Electrical Datasheet](https://www.nxp.com/docs/en/data-sheet/LPC546XX.pdf)
- [Programmer's Reference Manual](https://www.nxp.com/webapp/Download?colCode=UM10912) (behind registration wall, but free)
- [Errata](https://www.nxp.com/docs/en/errata/ES_LPC546XX.pdf)