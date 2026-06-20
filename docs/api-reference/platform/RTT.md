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

Mbed features basic support for using RTT during a debug session started by Mbed's build system. This support will only activate when `target.console-rtt` is enabled in JSON, and makes the RTT console for channel 0 (the default stdio channel) available on a telnet port on the local machine (the port number is controlled by the `MBED_RTT_PORT` option -- default 19022).

RTT is supported when using the `JLINK`, `OPENOCD`, and `PYOCD` upload methods. It is *not* supported with any other upload methods at this time.

!!! warning
    RTT is currently not supported when developing in CLion and using the `OPENOCD` or `PYOCD` upload methods, because CLion does not have a way to run a custom, per-target gdbinit file.

## RTT Options

Mbed exposes RTT configuration through the following options:

JSON option | Default | Description
------------|---------|------------
`rtt.ch0-up-buffer-size`| 1024 | Size in bytes of channel 0 up buffer (the one used for stdout).
`rtt.ch0-down-buffer-size`| 16 | Size in bytes of channel 0 down buffer (the one used for stdin).
`rtt.max-channels` | 3 | Maximum number of channels supported by RTT. This controls the size of some static arrays within the RTT code. Note that this does NOT, by itself, allocate space for the channels.
