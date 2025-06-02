# Upload Methods
Upload methods are the different tools that Mbed CE's build system can use to get code onto your device. Some upload methods methods also support debugging and stepping through code. Each upload method works together with the appropriate piece of interface hardware, which may already be present on your dev board or must be acquired separately and hooked up. 

You can watch one example of installing an upload method (STM32CUBE) in a video: [Youtube guide](https://youtu.be/ETAq6Es4-xo)

In order to upload, your board will need to have one or more upload methods configured for it in CMake. Some boards already have configuration provided by us (see [here](https://github.com/mbed-ce/mbed-os/tree/master/targets/upload_method_cfg) in the source). For boards that don't, or custom boards, you will need to provide the configuration yourself, by defining certain parameters in your top-level buildscript (or an include file). The different parameters and their values are explained in this document.

**IMPORTANT:** These variables need to be set in your CMakeLists immediately after including `mbed_project_setup`, before any add_subdirectory() calls.

Once you configure some upload methods, you can then run cmake with the `-DUPLOAD_METHOD=<method>` argument to select the method and enable uploading your code to the target.

To upload an executable, just run `ninja flash-xxx`, where xxx is replaced by the name of the executable target. Don't forget that you can also use `ninja help` to see the list of all available targets.

To debug, see the instructions associated with your IDE or the command line instructions.

## Upload method list

The below table lists each upload method, the device types it works on, its _parameters_ (variables that will have a fixed value) and its _options_ (variables that are different for different people and should be set on the command line when you run cmake). Since this config is set by standard CMake code, it's very easy to add your own custom logic to adjust these variables as needed!

**All current upload methods (see below for details):**

|Name|CMake Argument|Supports Uploading|Supports Debugging|Upload Speed|Supported On
|---|---|---|---|---|---|
|No upload method|`UPLOAD_METHOD=NONE`|❌|❌|N/A|N/A
|[Mbed USB](#mbed-usb)|`UPLOAD_METHOD=MBED`|✔️|❌|Fast|All Mbed-enabled boards (ones with a USB drive interface)
|[J-Link](#j-link)|`UPLOAD_METHOD=JLINK`|✔️|✔️|Fast|Mbed boards with J-Link On-Board. Custom boards with a J-Link probe.
|[pyOCD](#pyocd)|`UPLOAD_METHOD=PYOCD`|✔️|✔️|Medium|Almost all Mbed boards. Custom boards with an ST-Link or DAPLink probe.
|[OpenOCD](#openocd)|`UPLOAD_METHOD=OPENOCD`|✔️|✔️|Fast, if configured properly|Many different debug probes, including DAPLink and ST-Link. However, requires configuration.
|[STM32Cube](#stm32cube)|`UPLOAD_METHOD=STM32CUBE`|✔️|✔️|Fast|All STMicroelectronics Mbed boards, custom boards with ST_Link probes.
|[STM32Cube DFU](#stm32cube-dfu)|`UPLOAD_METHOD=STM32CUBE_DFU`|✔️|❌|Fast|Boards with USB DFU bootloader
|[stlink](#stlink)|`UPLOAD_METHOD=STLINK`|✔️|✔️|Fast|All STMicroelectronics Mbed boards, custom boards with ST_Link probes.
|[ArduinoBossac](#arduinobossac)|`UPLOAD_METHOD=ARDUINO_BOSSAC`|✔️|❌|Fast|Arduino Boards (w/ Arduino bootloader)
|[LinkServer](#linkserver)|`UPLOAD_METHOD=LINKSERVER`|✔️|✔️|Fast|NXP and Freescale boards, custom boards with DAPLink probes
|[Picotool](#picotool)|`UPLOAD_METHOD=PICOTOOL`|✔️|❌|Fast|Raspberry Pi boards (via USB DFU bootloader)
|[dfu-util](#dfu-util)|`UPLOAD_METHOD=DFU_UTIL`|✔️|❌|Medium|Boards with USB DFU bootloader

## Common Parameters (for all methods)

> UPLOAD_METHOD_DEFAULT

**Type:** String

This sets the default upload method that CMake will use if one is not explicitly set.

> MBED_UPLOAD_BASE_ADDR

**Type:** Integer

_added in [#369](https://github.com/mbed-ce/mbed-os/commit/11369b48be9c333fb2e4257b6f0cbef32f8f5e3f) on Oct 8 2024_

Base address for uploading code, i.e. the memory address where the first byte of the bin/hex file will get loaded to (with 0x prefix). This generally should point to the start of the desired flash bank and defaults to the configured start of the primary ROM bank (MBED_CONFIGURED_ROM_START).

*Note:* Unlike bin files, hex files contain a base address, which appears to be set based on the linker script. If the `output_ext` property in JSON is set to hex, uploading will use hex files instead of bin files, so the value of this parameter may not be respected by certain upload methods (e.g. J-Link). More investigation into this area may be needed.

## Common Options (for all methods)

> MBED_GDB_PORT

**Type:** Integer

This controls the port that GDB servers will be run on when debugging. A value higher than 1024 is recommended to allow debugging without root on Linux/Mac.

> MBED_UPLOAD_SERIAL_NUMBER

**Type:** String

_added in [#369](https://github.com/mbed-ce/mbed-os/commit/11369b48be9c333fb2e4257b6f0cbef32f8f5e3f) on Oct 8 2024_

This sets the serial number of the Mbed board or the debug probe (e.g. the ST-Link or the J-Link_ that will be used to upload. Setting this option is mandatory if you have multiple of the same type of board/probe plugged into your machine so that Mbed knows which one to use.

For J-Link probes, this option can be set to [the serial number of the probe or its nickname](https://wiki.segger.com/J-Link_GDB_Server#-select), both of which can be viewed using the J-Link Configuration tool. For other upload methods, this should be set to the serial number of the debug probe for your board. The easiest way to view these is by running `pyocd list` inside the Mbed virtual environment. On Linux, you can also find these by running `udevadm info` the same way as you do when setting up the TTY ports, see [here](https://github.com/mbed-ce/mbed-os/wiki/Configuring-an-Mbed-TTY-Device-on-Linux) for details.

## Mbed USB

This upload method interfaces with standard MBed boards which present themselves as USB drives. The Mbed python tools are used to automatically locate and flash boards connected to the system. The `MBED_UPLOAD_SERIAL_NUMBER` parameter is respected if you wish to select a specific board.

> ⚠️ **Personally, I have found the MBED upload method to be extremely unreliable on Linux.** It seems to be some sort of issue with the CMSIS-DAP firmware. If code uploads are not working reliably, switch to one of the other upload methods.

### Parameters

> MBED_UPLOAD_ENABLED

**Type:** Bool

Whether the MBed upload method can be activated.

> MBED_RESET_BAUDRATE

**Type:** Integer

**Default:** 9600

On some boards, Mbed Tools has to connect to the board's serial port in order to reset them. This configuration requires Mbed Tools to know the baud rate the board is operating at (though you can also likely get away with setting a slower baud rate here than what's in use).

## J-Link

This upload method connects to your processor via a J-Link JTAG box (or a J-Link On-Board) and the J-Link command line tools. It supports both flashing and debugging. Nbed OS will automatically locate the J-Link tools in their standard install locations on Windows, Mac, and Linux.

### Parameters

> JLINK_UPLOAD_ENABLED

**Type:** Bool

Whether the J-Link upload method can be activated.

> JLINK_CPU_NAME

**Type:** String

The name that your processor is known by to J-link. These are listed [here](https://www.segger.com/downloads/supported-devices.php).

> JLINK_UPLOAD_INTERFACE

**Type:** String

Method that the J-Link uses to talk to your processor. Can be 'JTAG' or 'SWD'. If unset, defaults to 'JTAG'.

> JLINK_CLOCK_SPEED

**Type:** Integer or String

Clock speed of the JTAG or SWD connection. Accepts either a speed in kHz or "adaptive" to automatically determine speed using the RTCK pin.

## pyOCD

This upload method utilizes Mbed's own [pyOCD](https://github.com/mbedmicro/pyOCD) application to flash and debug your processor. pyOCD mainly supports the DAPLink and STLink debug probes integrated into Mbed dev boards, but can also use standalone DAP-based programmers and has experimental support for the J-Link probe. Unlike all other debuggers, pyOCD has the ability to recognize and display the threads that are currently running in Mbed RTOS. This makes it the most convenient debugging solution for many Mbed applications. Just be prepared for a slow and glitchy experience on many processors, as it is not maintained as actively as many other tools.

Installation of pyOCD is usually as simple as `python3 -m pip install pyocd`, though on some platforms there are additional binary components that need to be installed for certain debug probes.

NOTE: Some older Mbed boards will need to have their firmware updated to work with pyOCD.

### Parameters

> PYOCD_UPLOAD_ENABLED

**Type:** Bool

Whether the pyOCD upload method can be activated.

> PYOCD_TARGET_NAME

**Type:** String

Name of your processor as passed to the `-t` option of pyOCD. This is usually the full or partial model number.

> PYOCD_CLOCK_SPEED

**Type:** Integer or String

Clock speed of the JTAG or SWD connection. Default is in Hz, but can use k and M suffixes for kHz and MHz

> PYOCD_EXTRA_OPTIONS

**Type:** List of strings

Additional command-line options which will be passed to `pyocd flash` and `pyocd gdbserver`. This is the way to set [session options](https://pyocd.io/docs/options.html), e.g. to reset before trying to connect you would set this variable to `-Oconnect_mode=pre-reset`.

## OpenOCD

This upload method utilizes the [OpenOCD](http://openocd.org/) application to flash and debug your processor. OpenOCD is highly configurable and supports a huge array of targets, from processors to flash memories to FPGAs. However, this flexibility comes at a cost: it can be a bit of a pain to configure. Normally, using OpenOCD with your target requires you to find or write special config scripts to configure it for each target. However, Mbed CE provides a working OpenOCD config for all targets we can test, so hopefully you won't have to deal with this. Once configured correctly, OpenOCD can be used as a versatile GDB server and flash programmer.

OpenOCD can be installed through most distro package managers, and Windows binaries can be downloaded from [here](https://github.com/openocd-org/openocd/releases/latest). On Windows, if you extract the downloaded files into a folder called e.g. `openocd-0.11` in Program Files, it should be automatically be detected by CMake.

Note: We recommend using at least openocd 0.11, as this brings in a great deal of improvements and fixes. Additionally, for support of some newer targets such as STM32U5, OpenOCD 0.12 might be required.

Note: for Nuvoton devices, the Nuvoton fork of OpenOCD is required. Download it from [https://github.com/OpenNuvoton/OpenOCD-Nuvoton/releases](here), and point CMake to it via setting the OpenOCD option. For example: `-DOpenOCD="C:/Program Files (x86)/OpenOCD-nuvoton/bin/openocd.exe"`.

### Parameters

> OPENOCD_UPLOAD_ENABLED

**Type:** Bool

Whether the OpenOCD upload method can be activated.

> OPENOCD_CHIP_CONFIG_COMMANDS

**Type:** List of Strings

This config option specifies all OpenOCD commands needed to configure the program for your target processor. At minimum, this should include loading an interface config file and a target config file. Since these options may need to access scripts in the OpenOCD install dir, CMake provides the variable `OpenOCD_SCRIPT_DIR` which will resolve to the scripts directory of OpenOCD on the current machine.

> OPENOCD_VERSION_RANGE

**Type:** String (CMake version range)

Acceptable version range of OpenOCD. This may be a single version (e.g. "0.12"), in which case it is treated as a minimum, or a versionMin...<versionMax constraint, e.g. "0.12...<0.13", to accept any 0.12.x version but not 0.13 or higher.

## STM32Cube

This uploader uses STMicroelectronics' official upload and debugging tools for its ST-LINK programmers. The upload tool can be obtained from the standalone [STM32CubeProg package](https://www.st.com/en/development-tools/stm32cubeprog.html), but for the GDB server you need the [STM32CubeIDE](https://www.st.com/en/development-tools/stm32cubeide.html)(3.3GB) or [STM32CubeCLT](https://www.st.com/en/development-tools/stm32cubeclt.html)(1.7GB), which includes both programs.

In my testing, STM32Cube is at least 5 times faster than PyOCD at uploading code to the chip, so if you have a large program it might be worth taking the time to set up. Also, its debugger seems to be considerably faster at things like setting breakpoints and single-stepping through code.

If you need the *programmer only*, you can install the relatively lightweight STM32CubeProg application. Once installed, find the `STM32_Programmer_CLI` executable in its install dir and pass the `-DSTM32CubeProg_PATH=<path to STM32_Programmer_CLI>` argument to CMake to point to it. 

If you need the programmer and the debugger, you must install the whole STM32CubeIDE or STM32CubeCLT. If you installed to the default install location, CMake should find it automatically. If not, set the `STM32CUBE_IDE_PATH` or `STM32CUBE_CLT_PATH` variable to point to the IDE or CLT install dir, which CMake will use to find the other tools. Note that on Macs this needs to point to the IDE dir inside the app package, e.g. `-DSTM32CUBE_IDE_PATH=/Applications/STM32CubeIDE.app/Contents/Eclipse`.

Unfortunately, one big limitation of STM32Cube is that there are no ARM Linux builds. So if you are trying to use something like a Raspberry Pi, you might have to use OpenOCD or stlink instead.

### Parameters: 

> STM32CUBE_UPLOAD_ENABLED

**Type:** Bool

Whether the STM32Cube upload method can be activated.

> STM32CUBE_CONNECT_COMMAND

**Type:** List

"Connect" (-c) command to pass to the programmer to connect to your target device. `port=SWD` should be all that's needed for most Mbed boards, but some also seem to need `reset=HWrst`. 

> STM32CUBE_GDBSERVER_ARGS

**Type:** List

Arguments to pass to the ST-Link gdbserver. `--swd` should be all that's needed in most situations.

### Options:

> STM32CUBE_PROBE_SN

**Type:** String

_As of Oct 8, 2024 this is deprecated in favor of MBED_UPLOAD_SERIAL_NUMBER_

Serial number of the ST-Link probe to connect to. If blank, will connect to any probe. You can get the list of serial numbers plugged into your machine with `STM32_Programmer_CLI -l`.

## STM32Cube DFU

This is an alternate version of the STM32Cube upload method that uses STM32CubeProgrammer in Device Firmware Upgrade (DFU) mode. This lets you flash code to STM32 MCUs connected over USB and booted into their bootloader ROM. This is one of the simplest ways to flash code as it requires no external debug probe. However, it cannot debug, and it generally requires a manual reset of the target to get in and out of DFU mode.

For information on how to set up this upload method, see the STM32Cube section above.

Note that Mbed CE also supports uploading to DFU devices via dfu-utils (see below for details). Compared to dfu-utils, STM32Cube is faster, but lacks the ability to reset the device and exit out of DFU mode after programming. (no idea why this is, but even official STMicro posts say to use dfu-utils if you need to reset the device).

### Parameters: 

> STM32CUBE_DFU_UPLOAD_ENABLED

**Type:** Bool

Whether the STM32Cube DFU upload method can be activated.

> STM32CUBE_DFU_CONNECT_COMMAND

**Type:** List

"Connect" (-c) command to pass to the programmer to connect to your target device. You'll need to supply the VID and PID of the DFU device. Example: `port=USB vid=0x0483 pid=0xdf11`. 

Note that if the user sets `-DMBED_UPLOAD_SERIAL_NUMBER=x`, a string like `sn=x` will get added to the end of these options to specify the serial number.

## stlink

This upload method is an open-source clone of ST's STM32CUBE flasher and debugger, supporting a wider range of devices and host machines. 

It can be downloaded from [the project releases page](https://github.com/stlink-org/stlink/releases), or installed through your package manager on many linux distributions. On Windows, once the stlink folder has been extracted into Program Files, it should be automatically be detected by CMake.

### Parameters

> STLINK_UPLOAD_ENABLED

**Type:** Bool

Whether the STLINK upload method can be activated.

> STLINK_LOAD_ADDRESS

**Type:** String

Load address argument to pass to stlink. 

> STLINK_ARGS

**Type:** List of Strings

Arguments to pass to stlink programs. The list of valid options is [here](https://github.com/stlink-org/stlink/blob/develop/doc/tutorial.md).

### Options:

> STLINK_PROBE_SN

**Type:** String

_As of Oct 8, 2024 this is deprecated in favor of MBED_UPLOAD_SERIAL_NUMBER_

Serial number of the ST-Link probe to connect to. If blank, will connect to any probe. You can get the list of serial numbers plugged into your machine with `st-info --serial`.

## ArduinoBossac

This upload method is [Arduino's variant of the bossac upload tool](https://github.com/arduino/BOSSA/tree/nrf), with additional patches to enable it to work with certain Arduino devices. To install it, you must either:
 - Install Arduino IDE, then install the board package for one of the bossac boards (e.g. Nano 33 BLE).
 - Or, download and install one of the binary packages using [TinyGo's instructions page](https://tinygo.org/docs/reference/microcontrollers/nano-33-ble/#installing-bossa).

By default, CMake will search the Arduino IDE's package install dir (`$HOME/.arduino*/packages/arduino/tools/bossac/1.9.1-arduino2` on Linux/Mac, `%LocalAppData%/Arduino*/packages/arduino/tools/bossac/1.9.1-arduino2` on Windows) for the bossac executable. If the executable is not located there, you will need to specify it manually via CMake argument, e.g. `-DArduinoBossac=/path/to/bossac`.

**Warning: If you have non-Arduino bossac in your PATH, CMake may find that instead unless you manually specify.**  Unfortunately Arduino did not create any way to tell if a bossac executable is built from their fork or not.

To use a board with ArduinoBossac, first boot it into bootloader mode by double-tapping the reset button. Then, figure out what serial port it shows up as and pass that to the `ARDUINO_BOSSAC_SERIAL_PORT` CMake parameter. Then, you should be able to flash it.

### Parameters: 

> ARDUINO_BOSSAC_UPLOAD_ENABLED

**Type:** Bool

Whether the ArduinoBossac upload method can be activated.

### Options:

> ARDUINO_BOSSAC_SERIAL_PORT

**Type:** String

Serial port name to talk to the bootloader on, e.g. COM7 or /dev/ttyACM0.

## LinkServer

As of 2023, NXP has [released](https://mcuoneclipse.com/2023/05/14/linkserver-for-microcontrollers/) an official command-line tool for debugging their development boards. This replaces the old Redlink system that was tightly integrated with MCUXpresso IDE. (though, technically it doesn't replace Redlink, it just adds a new command line user interface on top and packages it as a standalone tool). 

To install LinkServer, download and run one of the installer packages from [here](https://www.nxp.com/design/software/development-software/mcuxpresso-software-and-tools-/linkserver-for-microcontrollers:LINKERSERVER). The installers seem pretty straightforward for Windows, Mac, and Linux -- you just run the file after downloading it. That said, they only seem to support a pretty limited set of Linux distros -- I was able to get the package to install on Ubuntu 20 x64 but you might have to get creative on other distros.

Note: If you get an error on Ubuntu 22.04 like "libncurses5 must be installed in order to use the Arm GNU toolchain. The library is not available on your system", I believe you can simply ignore this, as it already gets far enough through the installer to have installed the packages you need.

Once you install LinkServer, CMake should find it automatically. If not, pass `-DLinkServer_PATH=</path/to/LinkServer>` to CMake.

### Parameters: 

> LINKSERVER_UPLOAD_ENABLED

**Type:** Bool

Whether the LINKSERVER upload method can be activated.

> LINKSERVER_DEVICE

**Type:** String

Chip name and board to connect to, separated by a colon. Example: `MIMXRT1062xxxxx:MIMXRT1060-EVKB`. This gets passed as the "device" argument to the LinkServer executable.

### Options:

> LINKSERVER_PROBE_SN

**Type:** String

_As of Oct 8, 2024 this is deprecated in favor of MBED_UPLOAD_SERIAL_NUMBER_

Serial number, or substring of the serial number, of the probe to connect to. You can get the list of probes and their serial numbers from `LinkServer probes`.

## Picotool

Picotool is a program for uploading code to Raspberry Pi devices with UF2 ROM bootloaders. It's needed in order for Mbed to program these devices.

To install Picotool, on Windows, you may download a binary for the tool [here](https://sourceforge.net/projects/rpi-pico-utils/files/v1.3.0/picotool.exe/download) and copy it into any location on your PATH. For other platforms, you will likely need to install it from source. Install instructions can be found on the [picotool repository](https://github.com/raspberrypi/picotool).

As long as picotool is installed onto your PATH, it should be found automatically by CMake. If it's not, you can manually set its location by passing `-DPicotool=/path/to/picotool`. to CMake.

To use a board with Picotool, first boot it into bootloader mode by holding down the BOOT (aka BOOTSEL) button when plugging in the USB cable. To make sure it's working, run `picotool info` in a terminal, and you should see your board show up.

**For Windows only:** If your device is not showing up, you might need to manually install the WinUSB driver for it. This can be done using [Zadig](https://zadig.akeo.ie/).

![Zadig screenshot](https://i.ibb.co/QH6hpR1/zadig-rp2boot-winusb.png)

### Parameters: 

> PICOTOOL_UPLOAD_ENABLED

**Type:** Bool

Whether the Picotool upload method can be activated.


### Options:

> PICOTOOL_TARGET_BUS

> PICOTOOL_TARGET_ADDRESS

**Type:** Integer

If you have multiple RPi Pico devices plugged in, these options may be set in order to select which one you want to program. The bus number and address can be found by running `picotool list` when multiple RPi Picos in bootloader mode are plugged in.


## dfu-util

[dfu-util](https://dfu-util.sourceforge.net/) is a program for flashing devices with a USB DFU firmware upgrade interface. This is used for certain Mbed-compatible boards, such as the Arduino Giga and Portenta. dfu-util only provides flashing, not debug support.

To install dfu-util, you may install it through a package manager, for example `apt-get install dfu-util` on Ubuntu, `brew install dfu-util` on Mac, and `pacman -S mingw-w64-ucrt-x86_64-dfu-util` with MSYS2 on Windows. The Arduino IDE also provides this package, so if you have the IDE and have installed an Arduino core that uses dfu-util, Mbed should be able to find it. Last but not least, you could download the [binaries](https://dfu-util.sourceforge.net/releases/) somewhere and set the `-Ddfu_util_PATH=/path/to/dfu-util` CMake option to point to it.


### Parameters: 

> DFU_UTIL_UPLOAD_ENABLED

**Type:** Bool

Whether the dfu-util upload method can be activated.

> DFU_UTIL_TARGET_VID_PID

**Type:** String

VID:PID pair of the target device being programmed. Example: `2341:0366`.

> DFU_UTIL_TARGET_INTERFACE

**Type:** Integer

Interface number of the interface on the target that should be programmed. You can use `dfu-util -l` to list the available interfaces for each target.

# Creating your Own Upload Method

These configurable options don't cover every single option that each upload method provides. To customize the commands used further, you can create your own upload method CMake module for your needs. First, copy one of the cmake scripts under `mbed-os/tools/cmake/upload_methods` to your own project (make sure that the location you add it to is on `CMAKE_MODULE_PATH`). Then, give it a new name, and change all variables using the old name to use the new name (e.g. `UPLOAD_JLINK_FOUND` -> `UPLOAD_MYMETHOD_FOUND`). Next, make the changes you need to the options and commands used, and add any needed configuration settings to your buildscript (including `UPLOAD_MYMETHOD_ENABLED`). Finally, you can activate the new upload method by passing the name to CMake via `-DUPLOAD_METHOD=MYMETHOD`.
