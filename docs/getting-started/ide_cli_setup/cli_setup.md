# CLI Setup

This page will show you how to take an existing Mbed CE project, build it, and debug it using command line tools.

## Building
1. Open a terminal in the project directory.
2. Create and enter a build directory: `mkdir build && cd build`
3. Run CMake: `cmake .. -GNinja -DCMAKE_BUILD_TYPE=Develop -DMBED_TARGET=<your mbed target>`
    - Valid options for the MBED_TARGET option are any supported Mbed OS board target, such as `LPC1768` or `NUCLEO_F429ZI`
    - The Develop build type is recommended for normal development work, but there is also the Debug build type which disables optimizations, and the Release build type which disables debug information.
4. Build the project by running `ninja`
5. If the project has an executable which can be flashed, and you have an upload method configured (see below), run `ninja flash-<executable>` to upload it to a connected device.

## Debugging

Suppose you want to debug an executable by the name of MyProgram.  

1. First, if you haven't already, you will need to make sure the project is configured to use an upload method that supports debugging.  Read about upload methods on the [Upload Methods page](../../upload-methods.md), and select a method to use using the `-DUPLOAD_METHOD=<method>` flag to CMake. 
2. Now, plug in your target and start a GDB server for it by running `ninja gdbserver` in the build directory.
3. Finally, open another terminal in the build directory and run `ninja debug-MyProgram`.  You should be dropped into a GDB session connected to your target!