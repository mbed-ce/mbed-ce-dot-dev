# RTT

RTT, short for Real-Time Transfer, is a [protocol](https://www.segger.com/products/debug-probes/j-link/technology/about-real-time-transfer/) created by SEGGER for transferring text to and from an embedded device using a debugger connection. It's an alternative to using a UART as a console which does not require any hardware or connections other than the existing SWD or JTAG connection that you use to flash code. 

Starting in version 7.1, Mbed CE supports using RTT for the Mbed OS [console](FileHandle.md#redirecting-the-console) via JSON option, and it can also be used manually in code via the `RTTHandle` class.

## Using RTT for the Console

To enable RTT for the regular Mbed console, just add the JSON option:

```json
"target.console-rtt": true
```

You will also need to turn off UART and/or USB console if they are enabled:
```json
"target.console-uart": false,
"target.console-usb": false,
```

With this enabled, Mbed will initialize the console at boot on RTT channel 0 (the standard I/O channel).

## Using RTT Manually

Any RTT channel may be used in your own code by instantiating the `RTTHandle` class. For example:

```cpp

#include "mbed.h"
#include "RTTHandle.h"

char upBuffer[512];
char downBuffer[8];
RTTHandle rttChannel1(1, upBuffer, sizeof(upBuffer), downBuffer, sizeof(downBuffer));
FILE* rttChannel1Stream;

void main() {
    rttChannel1Stream = fdopen(&rttChannel1, "r+");

    fprintf(rttChannel1Stream, "Hello on RTT channel 1");
}

```

Also be sure to link the `mbed-rtt` library in CMake to gain access to this class:

```cmake
target_link_libraries(MyApplication mbed-rtt)
```

## Connecting to RTT via the Debugger

Mbed features basic support for using RTT during a debug session started by Mbed's build system. This support will only activate when `target.console-rtt` is enabled in JSON, and makes the RTT console for channel 0 (the default stdio channel) available on a telnet port on the local machine. The port number is controlled by the `MBED_RTT_PORT` CMake variable -- default 19022.

RTT is supported when using the `JLINK`, `OPENOCD`, and `PYOCD` upload methods. It is *not* supported with any other upload methods at this time.

!!! warning
    RTT is currently not supported when developing in CLion and using the `OPENOCD` or `PYOCD` upload methods, because CLion does not have a way to run a custom, per-target gdbinit file.

### Making a telnet connection

You can follow the below steps to connect to the telnet server once the debugger is running:

**On Linux:**

Telnet is installed by default, so you should just be able to run the following command to connect to the RTT stream:

```
telnet localhost:19022
```

**On Mac:**

Telnet is not installed by default, so first run

```
brew install telnet
```

then follow the Linux instructions.

**On Windows:**

First run

```
pkgmgr /iu:"TelnetClient"
```

to install the Telnet client. Then use

```
telnet localhost 19022
```

(note that there is NO colon between the host and port, unlike the Linux/Mac command)

to connect.

Another option is to use [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html) in Telnet mode -- just be sure to enable "Implicit CR in Every LF" under the Terminal settings menu.

## RTT Options

Mbed exposes RTT configuration through the following options:

JSON option | Default | Description
------------|---------|------------
`rtt.ch0-up-buffer-size`| 1024 | Size in bytes of channel 0 up buffer (the one used for stdout).
`rtt.ch0-down-buffer-size`| 16 | Size in bytes of channel 0 down buffer (the one used for stdin).
`rtt.max-channels` | 3 | Maximum number of channels supported by RTT. This controls the size of some static arrays within the RTT code. Note that this does NOT, by itself, allocate space for the channels.
`rtt.cb-search-range-start`| unset | Override the start (first address) of the search range that will be used when a debugger searches for the RTT control block.
`rtt.cb-search-range-end`| unset | Override the end (last address) of the search range that will be used when a debugger searches for the RTT control block.

!!! note "Control Block Search Range Overrides"
    These options may be needed on specific targets that have gaps between memory banks, as some debug tools such as OpenOCD does not seem to handle this well.

    If you get an error like
    ```
    Info : rtt: Searching for control block 'SEGGER RTT'
    Error: Failed to read memory at 0x00010000
    Error: rtt: No control block found
    ```
    then you should consider setting these options to target the memory bank that contains the RTT control block.