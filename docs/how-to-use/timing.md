# Timing and Clocks

Mbed OS features several different ways to keep track of the time and to run code after a certain amount of time has expired. This page will walk you through the various time sources and discuss how to use them.

## The Time Sources

Mbed OS can have as many as four different time sources for applications to use. While all of them are usable for basic time tracking tasks, they do have some important differences to be aware of.

### Microsecond (μs) Ticker

**Classes:** `Timer`, `Ticker`, `TickerDataClock`

**Availability:** All targets

**Resolution:** microseconds

**Width:** 64 bits

**Counts:** At all times except during deep sleep.

The μs ticker is the primary way of keeping time in Mbed OS. It is the highest-resolution source of time available, and the only one available unconditionally on all targets. This makes it generally the time source used when internal Mbed OS functionality requires timekeeping. 

Internally, Mbed implements the μs ticker via a target-specific timer peripheral, such as a `TIMER` instance on RP2xxx, a `TIM` instance on STM32, or a `TIM` instance on LPC17xx. Mbed OS internally scales the clock frequency down to 1us, and also performs rollover compensation on the counts value. This means that you can count on a 64-bit, rollover-proof time value even if the hardware counter is only 16 bits!

The big limitation of the μs ticker is that it does not count when the MCU is in deep sleep (as generally most clocks are turned off or significantly slowed in this state). This means that if your MCU may go to sleep for significant periods of time, the us ticker will no longer be a true boot-time timestamp. If this behavior causes problems for you, and you don't want to use the LP ticker instead, you can disable deep sleep by calling

```cpp
sleep_manager_lock_deep_sleep()
```

at the start of main().

!!! warning "Peripheral Ownership"
    The μs ticker (and LP ticker, if supported) are one of the very few instances where Mbed OS will take control of a peripheral at boot. Normally Mbed tries not to use peripherals unless you explicitly instantiate the corresponding class. If you intend to manually interact with the timer peripherals on your chip, it is highly recommended to inspect your target's implementation of `us_ticker.c` and `lp_ticker.c` to determine which timer peripherals Mbed is using.

### Low Power (LP) Ticker

**Classes:** `LowPowerTimer`, `LowPowerTicker`, `TickerDataClock`

**Availability:** All targets [with Low-Power Ticker (`DEVICE_LPTICKER`) feature](https://mbed-ce.github.io/mbed-ce-test-tools/drivers/DEVICE_LPTICKER.html)

**Resolution:** Depends on target, usually ~100 microseconds (reports in microseconds)

**Width:** 64 bits

**Counts:** At all times

The LP ticker is similar to the μs ticker except that it is implemented using a "low power" timer peripheral, such as the LPTIM on STM32s or the Always-On Timer on RP234x. These low-power timer peripherals are usually clocked at a slower rate (tick period between 10 and 100 us) and run off of a slower oscillator, such as an internal RC oscillator or an external 32.768kHz crystal.

Unlike the μs ticker, the LP ticker remains on during deep sleep.

!!! note "Acronym Confusion"
    You will also sometimes hear the LP ticker referred to within the Mbed source code as the "Low Precision Ticker", I assume because it fits the acronym and the LP ticker is indeed less precise than the μs ticker. In fact, this may have been the original meaning of the term, but "Low Power" seems to be in much more common use now.

### Real Time Clock (RTC)

**Classes:** [`RealTimeClock`](https://mbed-ce.github.io/mbed-os/classmbed_1_1_real_time_clock.html)

**Availability:** All targets [with RTC (`DEVICE_RTC`) feature](https://mbed-ce.github.io/mbed-ce-test-tools/drivers/DEVICE_RTC.html)

**Resolution:** seconds

**Width:** 64 bits in software, but actual supported time range generally depends on hardware

**Counts:** After `RealTimeClock::init()` is called

The RTC API is Mbed OS's abstraction of the real-time clock feature present on many MCUs. The RTC differs from other timer peripherals in that it keeps track of the absolute calendar time, not time since boot. So, while it's less precise, it is suitable for applications that need the absolute date and time, such as working with filesystems or implementing a clock.

On most targets (not RP2040), the RTC will keep time across chip resets, so you shouldn't need to reset its time unless your MCU gets totally powered off. It also continues to count in deep sleep mode.

!!! note "Calendar Conversion"
    On many chips, for some god-awful reason, the RTC acts as a "hardware calendar" and reads out a day/year/month, hour:minute:second date, rather than a simple timestamp. This seems to be a case of chip designers trying to be helpful, but it actually makes things a lot more complicated, especially as many of these "hardware calendars" have missing or broken leap year support.

    Mbed implements the [needed logic](https://github.com/mbed-ce/mbed-os/blob/fa83cabe2d2e2d159948bf8c943b1f0fa6121fee/platform/source/mbed_mktime.c#L95) to convert the time from these RTCs back into a timestamp when reading them so that we can provide a consistent RTC API.

### System Tick

**Classes:** `rtos::Kernel::Clock`

**Availability:** Always

**Resolution:** milliseconds

**Width:** 64 bits

**Counts:** From boot

This time source provides access to the real-time OS's tick counter. This counter is incremented every millisecond when the RTOS performs a scheduling operation.

The underlying time source depends: if the target does not define `MBED_TICKLESS`, it uses the ARM core's SysTimer peripheral. This is universal to all ARM MCUs, but has the limitation that it forces a wakeup each millisecond, creating a lot of sleep enter/exit overhead and preventing deep sleep from being used by the RTOS at all. If `MBED_TICKLESS` is defined, the RTOS is configured to use the μs ticker or LP ticker as its time and wakeup source.

If there is no RTOS, this time source is still available, but does not use the RTOS and directly reads the μs ticker or LP ticker.

## The Chrono Library

## Measuring Time with Timer

## Getting the Time with a Clock

## Scheduling Callbacks with Ticker

## Delay Functions

## C Time API

Mbed also implements the back-end of the [C library time API](https://cppreference.com/c/chrono). The following functions get defined in terms of Mbed time sources:

- [`clock()`](https://en.cppreference.com/c/chrono/clock) - Defined to return the time since boot, as measured by the μs ticker. Note that Newlib defines `CLOCKS_PER_SEC` to 100, so this is a fairly low-resolution way of measuring time.
- [`gettimeofday()`](https://linux.die.net/man/2/gettimeofday) and [`time()`](https://cppreference.com/c/chrono/time) - Returns seconds since the unix epoch from the RTC (initializing it if needed).
- [`settimeofday()`](https://linux.die.net/man/2/gettimeofday) and [`set_time()`](https://mbed-ce.github.io/mbed-os/group__platform__rtc__time.html#ga5d1e10825bf4a6ecdd567e9f2f384ed1) - Sets the given unix epoch time into the RTC (initializing it if needed).

If your MCU does not have RTC support in Mbed, the functions that use the RTC will instead use the LP ticker (with an offset controlled by the setter functions). This should be functionally identical except that time will not persist after a chip reset. If the LP ticker also is not supported, these functions do nothing and return 0.

Here's a quick example of how to use these functions:

```cpp
#include "mbed.h"

int main()
{
    set_time(1256729737);  // Set RTC time to Wed, 28 Oct 2009 11:35:37

    while (true) {
        time_t seconds = time(NULL);

        printf("Time as seconds since January 1, 1970 = %u\n", (unsigned int)seconds);

        printf("Time as a basic string = %s", ctime(&seconds)); // warning: this uses a global string buffer and is NOT thread safe

        char buffer[32];
        strftime(buffer, 32, "%I:%M %p\n", localtime(&seconds));
        printf("Time as a custom formatted string = %s", buffer);

        ThisThread::sleep_for(1000);
    }
}
```

Also note that if you wish to convert the UNIX timestamp returned by `time()` into a calendar date and time, this can be done using the [`gmtime_r()`](https://cppreference.com/c/chrono/gmtime) function. This will give you a `struct tm` which contains the calendar date and time.

## Future Work