# Setting Up Mbed CE with PlatformIO

Like its predecessor ARM Mbed, Mbed CE can be used with PlatformIO. PlatformIO is an embedded development framework that makes setup of Mbed and the tools it needs (compiler, build system, etc) simple -- almost everything can be installed just by creating your `platform.ini` file and telling PlatformIO to go. It also provides a repository of libraries that you can use (though you will need to be careful to find libraries that are compatible with your target board and with Mbed -- many have compatibility restictions).

## Should I Use PlatformIO?
PlatformIO provides an extremely simple way to set up Mbed CE, and is a great option for simple use of Mbed. It avoids the need to clone or create a stater project or spend time installing the toolchain. If you are new to embedded development, or you are instructing others, it is likely the fastest and simplest way to get running.

However, the following restrictions apply with PlatformIO:

- Only one executable can be built per project (just like the old ARM Mbed build system)
- It's not possible to add custom build system logic (generating files, etc) or tests
- Some MCUs currently do not have debugging support in PlatformIO, as PlatformIO doesn't have the right debug tools. This currently includes `LPC1768` and newer STM32 lines such as `STM32H5xx` and `STM32U0xx`.
- It is only possible to keep the 3-4 most recent releases of Mbed CE available on PlatformIO due to space constraints (we only get 500MB of package storage with a free account). So, you cannot really pin a specific Mbed CE version over any length of time.

!!! warning "Beta Status"
    Mbed CE integration with PlatformIO is still in **beta status**. While in this period, the following additional limitations apply:

    - Using Mbed CE requires explicitly using forked versions of the platform repos by adding custom URLs to platformio.ini. This is explained further below.
    - Only the most popular MCU targets and boards are supported with PlatformIO. Currently this includes:
        - Nearly all STM32 boards
        - The Mbed LPC1768 board
        - NXP (formerly Freescale) Kinetis line of MCUs, e.g. `K64F`
        - RP2040
    - After modifying `mbed_app.json5`, a manual clean of the project is required as build system dependencies are not properly set up.

    Fixing most of these issues would require official support for Mbed CE in PlatformIO. We are in communication with the PlatformIO developers, and they would be happy to add this support if we can demonstrate significant usage of Mbed CE on their platform.

If you decide to use PlatformIO, please continue with this guide. If not, continue on to the [Toolchain Install](toolchain-install.md) page to set up the full Mbed CE dev environment.

## Getting Started with PlatformIO
PlatformIO offers both a [command line interface](https://docs.platformio.org/en/latest/core/index.html) and a [VS Code plugin](https://platformio.org/install/ide?install=vscode).

### VS Code Setup (Recommended)
- Install the PlatformIO extension
- Open the extension from the new tab on the left, and wait until it finishes installing (see the bottom console)
- Once it completes, restart VS Code as instructed

### CLI Setup
- Run the [PIO Core Installer](https://docs.platformio.org/en/latest/core/installation/methods/installer-script.html)
- Add the folder mentioned in the console output to your PATH to enable running the `pio` command

## Finding your Platform and Board
PlatformIO organizes code into _platforms_ and _frameworks_. A platform describes a line of target devices, and is roughly equivalent to a target family in Mbed, though it can sometimes be a little bit broader (for example, in Mbed each STM32 line is its own target family, but in platformio, they are all part of the `ststm32` platform). A framework, meanwhile, is a library that allows programming on the platform, such as Arduino, Zephyr, or Mbed CE!

If you have a target board you want to program, you will need to start by finding out what it is called in PlatformIO. [This page](https://docs.platformio.org/en/latest/boards/index.html) has the full list. Once you find a board, click the link to go to its page, and this will show its PlatformIO board and platform. For example, on the page for [Nucleo F429ZI](https://docs.platformio.org/en/latest/boards/ststm32/nucleo_f429zi.html), we can see that it contains the following example config file:

```
[env:nucleo_f429zi]
platform = ststm32
board = nucleo_f429zi
```

From this, we can see that the platform name is `ststm32` and the board name is `nucleo_f429zi`.

## Creating an Mbed CE PlatformIO Project

Now, we can create our project and get started with Mbed CE. First, we need to make a blank PlatformIO project. If you are using the VS Code plugin, there is a wizard for this accessible via the PlatformIO sidebar. If using the command line interface, the `pio project init` command can be used.

When creating the project, it will ask for a platform, board, and framework. Input the platform and board names that you found previously. As for the framework, mbed-ce is not an official option yet, so you can select arduino, mbed, or any other framework that the board supports. We'll fix that next.

Once your project has been created, open up `platformio.ini`. It should look something like this (example for Nucleo F429ZI):

```
[env:nucleo_f429zi]
platform = ststm32
board = nucleo_f429zi
framework = mbed
```

!!! note "Environment Names"
    The text after `env:` is the name of this environment configuration and can be anything. It's usually helpful to keep it the same as the board name, but it doesn't have to be.

We will need to change the `framework` to `mbed-ce`. We will also need to change the platform to the correct Mbed CE forked repository:

|Platform|Replacement|
|--------|-----------|
|`ststm32`|`https://github.com/mbed-ce/pio-platform-ststm32.git#dev/add-mbed-ce`|
|`freescalekinetis`|`https://github.com/mbed-ce/pio-platform-freescalekinetis.git#dev/add-mbed-ce-support`|
|`nxplpc`|`https://github.com/mbed-ce/pio-platform-nxplpc.git#dev/add-mbed-ce-support`|
|`raspberrypi`|`https://github.com/platformio/platform-raspberrypi.git`|

Your platformio.ini file should now look something like:

```
[env:nucleo_f429zi]
platform = https://github.com/mbed-ce/pio-platform-ststm32.git#dev/add-mbed-ce
board = nucleo_f429zi
framework = mbed-ce
```

You can now save this file. In the IDE, that should begin setting up everything automatically. If using the command line, you will need to use `pio run` after saving it. Be patient, as the initial download of all the tools and repositories can take 5-10 minutes the first time.

## Using your Project

Before building your project, you will need to create at least one cpp file under the `src/` folder that contains `int main()`. For example, you could copy the one from the hello world project [here](https://github.com/mbed-ce/mbed-ce-hello-world/blob/master/main.cpp).

To build your project, you can use the build button on the PlatformIO sidebar from VS Code or the `pio run` command on the command line.

After connecting your target, you should then be able to flash code to it via the "Upload" target in the IDE or the `pio run -t upload` command. Note that some targets may need upload method configuration to be able to flash code and/or be debugged. The upload method can be changed via the `upload_protocol` option in platformio.ini -- see your board's documentation page for the list of allowed upload methods.

!!! warning "mbed_app.json5"
    A default mbed_app.json5 file will be created in your project directory the first time you build the project, if there is not one already. This file will contain reasonable defaults for use of Mbed CE with platformio (mainly, this means setting the serial baudrate to 9600 to match what the serial terminal expects).

    While the Mbed CE - PlatformIO integration is in beta status, you will need to _execute a manual clean of the project_ whenever you modify mbed_app.json5. Otherwise, settings changes may not get properly picked up. You can do this by building the "Clean" target in the IDE, or by running `pio run -t clean` on the command line.

## Depending on Libraries

Mbed CE contains a number of optional libraries that must be linked for functionality like networking and storage. Normally, you would use CMake to link your application to these libraries, but platformio projects don't use CMake buildfiles. Instead, you can do this via adding a block like the following to `mbed_app.json5`:

```json5
"link_libraries": [
    "mbed-storage-sd", // for SDBlockDevice
    "mbed-netsocket" // For networking support
] 
```