# USB Setup on Linux

By default, when you plug in an Mbed device to a Linux machine, its serial port (TTY) will get an arbitrary path (e.g. /dev/ttyACM0), and this path can even change depending on the order you plug things in. This can be really annoying, especially if you are working with multiple Mbed devices at the same time.

This page will show you how to configure an Mbed device to have a consistent TTY path on Linux, and also how to set the permissions so that it can be accessed without root.

## Setting up the TTY
1. First, you need to figure out what tty device corresponds to the Mbed board.  It will generally be one of the /dev/ttyACMx devices, which you can list by running `ls -l /dev/ttyACM*`.  Maybe the easiest way to identify the right one is to unplug and replug the board and see which ttyACM device vanishes and reappears.
2. Then, once you have found it, get the serial number of the Mbed device from its tty:
    ```
    udevadm info -a -n /dev/ttyACMx | grep '{serial}' | head -n1
    ```
3. Create a file `/etc/udev/rules.d/99-usb-serial.rules` with contents like:
    ```
    SUBSYSTEM=="tty", ATTRS{serial}=="<serial number>", MODE:="0666", SYMLINK+="tty<mbed target>", GROUP="plugdev"
    ```
    where `<mbed board>` and `<serial number>` are replaced by the mbed target name and the serial number found above.

    For example, for my NUCLEO_F429ZI board, it looks like this:
    ```
    SUBSYSTEM=="tty", ATTRS{serial}=="066CFF343537424257254941", MODE:="0666", SYMLINK+="ttyNUCLEO_F429ZI", OWNER="jsmith"
    ```

    **Note: If you have multiple of the same Mbed board, make sure to give them different symlink names, e.g. `ttyNUCLEO_F429ZI_1` and `ttyNUCLEO_F429ZI_2`.**

4. If you have not done this before, add your username to the `plugdev` group:
   ```
   $ sudo usermod -a -G plugdev $(whoami)
   ```
   You will need to log out and log in for this change to take effect.

5. Reload udev:
    ```
    $ sudo udevadm control --reload-rules
    $ sudo udevadm trigger
    ```
6. You should now see your mbed device at the desired TTY path.
7. To connect to it, install minicom, then run `minicom -D /dev/tty<mbed target> -b 115200` (or substitute a different baud rate if your code is not configured for 115200 baud)

## Setting Up Udev Rules for Programming
If your device has a debugger onboard, you may also need to install udev rules to allow that debugger to be accessed by tools like STM32Cube, OpenOCD, and PyOCD.

The easiest way to do that is to download this config file [here](https://github.com/openocd-org/openocd/blob/master/contrib/60-openocd.rules) and copy it to /etc/udev/rules.d.  Then, reload udev as above.