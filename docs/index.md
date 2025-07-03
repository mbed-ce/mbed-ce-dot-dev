# Mbed OS Community Edition
## What is Mbed OS?

Mbed OS is an embedded real-time operating system (RTOS) and hardware abstraction layer (HAL) written in C/C++ that runs across a wide range of ARM microcontrollers (MCUs). Mbed OS can run on chips as small as 16k RAM / 64k flash, but also scales up to ones with megabytes of RAM and flash. Mbed OS is designed for Internet of Things (IoT) applications, and as such supports wired and wireless network connectivity, Internet protocols, and network encryption.

What does that mean in plain English? Well, Mbed OS is a software framework to make it easier & faster to program microcontrollers. It is designed to make the basics (e.g. initializing the chip, talking to other chips, running multiple threads, and using the network) as simple as possible, without sacrificing the ability to access advanced features. Were you to start from nothing (or with the limited SDKs the manufacturers provide), it can take weeks or months of work to get to the point where you can blink an LED, print out a message, or send a network packet. Mbed OS is designed to handle that stuff for you, so you can proceed with your project with a minimum of fuss.

Mbed OS uses clean C++ for its user-facing API (and efficient C code for the majority of its internals). This provides an elegant way to model things that have internal state and life cycles, like processor peripherals, network sockets, and external chips. You'll find no device tree files or homegrown configuration languages here -- your project is simply defined by which hardware objects it creates and what pins it passes to them (though there's also a sprinkling of JSON for configuring global options and defining new targets). In this developer's opinion, it's the cleanest way that anyone has ever figured out to make complex peripherals and hardware usable from simple code.

## Features of Mbed OS
### RTOS (or not)
Mbed OS includes the [Keil (pronounced "Kyle") RTX5 RTOS](https://github.com/ARM-software/CMSIS-RTX), wrapped with a C++ API that makes it easy to use. This lets you use threads, mutexes, semaphores, condition variables, and inter-thread queues just as easily, or even more so, than if you were using desktop C++.

If you don't need the RTOS, no problem! Just set
```js
"target.application-profile": "bare-metal"
```
in your mbed_app.json5 file and the RTOS goes away, freeing up additional RAM for your application to use.

_Note that more advanced functionality of Mbed, such as networking and bluetooth, requires the RTOS._

### Microcontroller Peripherals
Mbed OS provides drivers for most common peripherals of most microcontrollers, including:

#### Communication
- I²C Bus
    - Single-byte and transactional blocking transfers
    - Asynchronous transfers via interrupts
    - Most ports support only 100kHz and 400kHz speeds
- SPI Bus
    - Single-byte and transactional blocking transfers
    - Asynchronous transfers via interrupts and/or DMA
- UART (aka serial port)
    - Blocking (unbuffered) and interrupt-based (buffered) versions available
    - Hardware flow control (CTS and/or RTR) supported
- Quad and Octal SPI
- CAN
    - No CAN FD support at this time
- USB Device
    - No USB host support at this time
- Ethernet
    - 10/100Mbit supported

#### Timing
- Microsecond clock
- Low power (& low precision) clock
- PWM
- Watchdog timer
- Real-time clock

#### Analog
- ADC (AnalogIn)
    - Basic support only, no batched or triggered conversions
- DAC (AnalogOut)

#### Security
- True RNG
- CRC
- Hardware encryption offload (for PSA Certified MCUs)
- Memory Protection Unit (MPU)

#### Wireless Comms
_Note that the Mbed CE maintainers currently cannot provide maintenance/development for wireless networking features due to lack of expertise and bandwidth_

- Bluetooth Low Energy
- 802.15.4 mesh networking (Wi-SUN, Thread, LoWPAN)
- Wi-fi adapters
- AT cellular modules

#### ...and more!

For the full list of supported peripheral drivers, and a list of which targets each peripheral is available on, see [here](https://mbed-ce.github.io/mbed-ce-test-tools/drivers/).

Note that using Mbed OS in no way stops you from implementing your own peripheral drivers if needed. Want to take advantage of some advanced timer feature or communication peripheral that isn't available in Mbed? No problem! Just write your own chip-specific code for that part using the manufacturer SDK, and leave all the rest of your project (boot, I/O, storage, etc) up to Mbed! It's still way less work than implementing the entire project from scratch.

### Low-Power Operation

Mbed is designed for low-power MCUs with sleep support. Mbed will automatically send the MCU to sleep when all threads are sleeping (or when the application is sleeping if the RTOS is disabled). Both regular sleep and deep sleep are supported, if available on your microcontroller. Mbed will intelligently decide whether to use deep sleep or not based on:

- How long until the next wakeup (vs how long does it take to enter and exit deep sleep)
- Whether any background operations (e.g. Ethernet Rx) have locked out deep sleep
- Whether debug mode is in use (deep sleeping can cause problems with the debugger so it is not used in Debug mode)


### Network Connectivity

Mbed OS supports connecting to the Internet via Ethernet, Wi-Fi, cellular modules, and 802.15.4 mesh networks. Two IP stacks are available: [LwIP](https://savannah.nongnu.org/projects/lwip/) and Mbed's homegrown Socket Abstraction Layer (SAL) Nanostack.

After joining a network, Mbed applications can use static IPs, DHCP, or address autoconfiguration to get an IP address. Then, they can use DNS to resolve websites' hostnames and create UDP, TCP, and TLS sockets to talk to them!

### I/O and Storage

Many embedded applications need to store persistent data locally, and Mbed provides easy, effective ways to do this. There are two layers to Mbed's storage API: block devices and filesystems.

#### Block Devices
Block devices allow access to the raw bytes stored in a memory device, without any idea of the structure of said bytes. Block devices provide APIs to read, program, and erase a memory device, as well as a way to check its size and attributes.

Mbed OS includes block devices for standardized memory devices such as SPI, QSPI, and OSPI flashes, I²C EEPROMs, MicroSD cards. It also includes "virtual" block devices, such as `SlicingBlockDevice` and `MBRBlockDevice`, which divide one physical block device into smaller regions. Or, if you don't mind slow writes that stop the CPU from executing anything, there is `FlashIAPBlockDevice`, which allows you to reprogram the microcontroller's own flash memory!

#### File Systems

To make use of block devices, Mbed provides several file system drivers which operate using a block device. `LittleFileSystem` 1 and 2 are both available, and optimized for storing files in embedded flash memory. Also, to interoperate with memory formatted on other OSs, `FATFileSystem` can be used to read and write FAT32-formatted devices. Or, if you need something more like a database than a file system, the `KVStore` library can provide robust key-value storage using a block device.

### C Library Hooks

Mbed OS integrates with the Newlib C library so that many standard C functions will work how you're used to, even on embedded.

File I/O is implemented, on both filesystem files and streams (e.g. serial ports). Simply use the `fdopen()` function to get a `FILE *` from a file descriptor or `FileHandle` object, and then you can use it with all of the standard C I/O functions like fwrite, fread, fprintf, and fscanf. Filesystem functions are also available, so you can use `opendir()`/`readdir()` to walk through directories and `open()` to open files.

Time functions are also supported -- the C `clock()` function will work as expected via the us ticker, and if your chip has a real-time clock, you can set its time via `settimeofday()` and access it via `gettimeofday()` or `time()`. Mbed also integrates deeply with the C++ `chrono` library -- more docs on this will be posted soon.

### Development Environments

While, as a volunteer project, we cannot support anything like the online compiler, we do support several powerful development environments. Many users enjoy our VS Code integration -- the Mbed OS project generates all of the right config files needed for VS Code to build, flash, and debug code out of the box. We also integrate with JetBrains CLion, a paid option with very capable code analysis. But don't feel left out if you don't like IDEs -- we also support development via the command line, so you can flash and debug without needing a GUI at all.

### Bootloader

Most serious firmware projects will need some way to update themselves in the field without a debug probe connected. Mbed CE provides this via the [MCUBoot bootloader](https://github.com/mbed-ce/mbed-mcuboot-bootloader). MCUBoot is a small bootloader that lets your application update itself without putting any constraints on how and when you download your updates. Your application simply has to download an update image and copy it to the designated location in some connected memory device, and then reboot.

The bootloader will then see the update image and swap it with the current application image, in a way that is both power-off-safe and reversible. Then it will boot up the new image, but if it fails to boot, it will swap back to the old one to let it sort out the problem.

MCUBoot always uses cryptographic signatures to sign images and make sure that only authenticated and non-corrupt images can be installed. It also optionally can encrypt the images until they are installed, so that your application code isn't available to someone who obtains the update image.

## Where Mbed OS Runs

Mbed OS runs on a variety of development & evaluation boards with a number of different types of ARM microcontrollers. Our lineup includes popular hobbyist boards, like the RP2040, Mbed LPC1768, Arduino Giga and Portenta, and FRDM-K64F. It also includes a number of somewhat more obscure vendor boards produced by vendors for evaluation of their MCUs. For the full list of supported boards, see our list [here](https://mbed-ce.github.io/mbed-ce-test-tools/targets/).

Below is the list of supported microcontroller families, by manufacturer. In general, if Mbed OS supports a given MCU family, it can be ported to any specific MCU within that family and a specific board without too much difficulty.

| Manufacturer | MCU Families |
|--------------|--------------|
| ST Microelectronics | F0*, F1*, F2*, F3*, F4, F7, G0*, G4*, H5, H7, L0*, L1*, L4, L5*, U5, WB*, WL*|
| NXP (incl. Freescale) | i.MXRT 105x/6x, i.MXRT 117x*, K22F*, KL25Z, KL43Z*, KW43Z*, KL46Z*, K64F, K66F*, K82F*, LPC1114*, LPC17xx, LPC54114*, LPC546xx|
| Nuvoton | M48x, M46x, M45x, Nano130, NUC472, M2354, M251, M261 |
| Raspberry Pi | RP2040 |
| Ambiq Micro | Apollo3 |
| Infineon (formerly Cypress) | PSOC 62*, PSOC 64* |
| Maxim | MAX32620*, MAX32630*, MAX32660*, MAX32670* |
| Nordic Semiconductor | nRF52832, nRF52840 |
| Renesas | RZ/A1xx*, RZ/A2xx* |
| Silicon Labs | EFM32GG* |
| Analog Devices | ADuCM4050*, ADuCM3029* |
| Giga Devices | GD32F3*, GD32F4* |
| Samsung | S1SBP6A* |
| Toshiba | TX04 M460*, TXZ+ M4G*, TXZ+ M4K*, TXZ+ M4N* |

* Denotes device families that have not been tested yet by the Mbed CE maintainers. Device families with this mark do not have upload method support and may not have been tested since the vendor's original porting effort, so they are more likely to have issues.

## Why Mbed Community Edition?

This site is for Mbed OS Community Edition, or Mbed CE for short. Mbed CE is a community-led continuation of the Mbed OS project after its abandonment by ARM. Anyone can contribute features, bugfixes, or even new targets to Mbed CE -- visit the repository linked in the header for the source code!

## Differences from Mbed OS 6

At Mbed CE, we are proud to have made a number of improvements from ARM Mbed OS 6. These include (but are not limited to):

- Setting up a CMake-based build system that builds and links the C++ code using standard tooling (instead of homegrown python scripts)
- Splitting the Mbed codebase up into a core library and a number of optional libraries, such as networking and storage. This reduces build time and code bloat if you don't need these extra features.
- Support for a number of new target boards for existing MCUs, including Teensy 4.0/4.1, Arduino Nicla Sense, and Arduino Giga & Portenta
- Support for some completely new MCU families, including STM32H5, STM32U0, and RP2040
- A new "upload method" system, which allows you to build and flash code with one click by building a target from the CLI or an IDE, whether you are using the USB drive mode or a debugger
- API updates to the `I2C` class to make it better documented, easier to use, and have an enum for return codes. No more "0 means success from some functions but failure from others"!
- A new [CI test shield](https://github.com/mbed-ce/mbed-ce-ci-shield-v2) that does compliance testing on drivers like I2C, SPI, and PWM to make sure that they actually work how they are supposed to when merging changes. Work on automating this testing for more devices is still ongoing.
- A memory bank configuration system that keeps track of what memory banks exist on a device and lets you easily configure the region of flash that a build should use (and it understands the difference between what _physically exists_ vs what _should actually be used_!)
