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

This time source provides access to the real-time OS's tick counter. This counter is incremented every millisecond when the RTOS performs, or would have performed, a scheduling operation.

The underlying time source depends: if the target does not define `MBED_TICKLESS`, it uses the ARM core's SysTimer peripheral. This is universal to all ARM MCUs, but has the limitation that it forces a wakeup each millisecond, creating a lot of sleep enter/exit overhead and preventing deep sleep from being used by the RTOS at all. If `MBED_TICKLESS` is defined, the RTOS is configured to use the μs ticker or LP ticker as its time and wakeup source.

If there is no RTOS, this time source is still available, but does not use the RTOS and directly reads the μs ticker or LP ticker.

## The Chrono Library

Mbed OS 6 and later make heavy use of the [C++ std::chrono library](https://cppreference.com/cpp/chrono) for storing and operating on times. This library is extremely powerful and capable, **but** suffers from a major flaw: it's not the easiest to understand, and certain operations (e.g. converting to float) are not very intuitive.

In this section, we'll do our best to remedy this flaw by explaining the basics of how `std::chrono` works and illustrating how to accomplish common tasks.

### Time Durations

Time durations are represented in `std::chrono` by the [`std::chrono::duration`](https://cppreference.com/cpp/chrono/duration) type. A `duration` represents a certain amount of time, e.g. one second or ten milliseconds. You could also think of it as the _change_ in time between two timestamps. It does NOT represent an absolute timestamp, such as "1 second after the Unix epoch".

The `std::chrono::duration` class takes two template arguments: the representation type, `Rep`, and the conversion ratio, `Period`. `Rep` is the underlying type that is used to store the duration internally. It is always a signed integer, and is usually `int64_t` for durations of seconds or finer. The other template arg, `Period`, basically gives the scaling factor between `Rep` and whole seconds. For example, if we wanted to define a duration type expressing milliseconds, we could do this as:

```cpp
typedef std::chrono::duration<int64_t, std::ratio<1, 1000>> milliseconds;
```

This tells the chrono library that this is a duration which contains an int64_t, and that an integer value of 1 in this duration equals 1/1000th of a second.

Of course, we don't actually need to define that ourselves, because the standard library [provides](https://cppreference.com/cpp/chrono/duration#Helper_types) `std::chrono::seconds`, `std::chrono::milliseconds`, `std::chrono::microseconds`, and others already! These make working with "standard" time types a snap.

`duration`s implement operator overloading and support all your standard mathematical operations.

```cpp
// Durations can be added and subtracted
1ms + 3ms; // evaluates to 4ms
3ms - 1ms; // evaluates to 2ms

// Durations can be multiplied, but only by integers
1ms * 4; // evaluates to 4ms

// Durations can be divided by integers or another duration
3ms / 2; // evaluates to 1ms (integer division!)
105ms / 50ms; // evaluates to 2

// Durations can be modulo-ed by integers and other durations
3ms % 2; // evaluates to 1ms
105ms % 50ms // evaluates to 5ms
```

!!! note "Storage Type"
    Internally, durations are stored as a single integer of type `Rep`. There is no additional memory overhead from using the C++ class, and as long as compiler optimizations are on, there should be virtually no code size or execution time overhead either versus simply using an integer.
    This is one of those "zero-overhead abstractions" that us C++ people are so proud of!

### Literal Suffixes

Is `std::chrono::milliseconds` too much typing for you? Well, `std::chrono` has a way to make defining times even easier. If you stick this line at the top of your code file:

```cpp
using namespace std::chrono_literals;
```

then you can use standard SI units abbreviations as literal suffixes to declare time constants super easily:

```cpp
using namespace std::chrono_literals;

constexpr auto SHORT_TIMEOUT = 1us; // equivalent to std::chrono::microseconds(1)
constexpr auto LONG_TIMEOUT = 10s; // equivalent to std::chrono::seconds(10)
```

!!! tip
    You cannot directly use these literal suffixes with variables, but you can use multiplication to do basically the same thing.

    ```cpp
    void doSomething(int timeout_ms) {

        // Long way
        const std::chrono::milliseconds timeout(timeout_ms);

        // Short way
        const auto timeout = 1ms * timeout_ms;
    ```

### Clocks and Time Points

`duration`s are only half the story of the chrono library. The other half is [`std::chrono::time_point`](https://cppreference.com/cpp/chrono/time_point). This class represents an absolute point in time, _as measured by_ some specific clock. `time_point`s are templated on the clock itself, meaning that time_points from two different clocks are treated as completely different types by the compiler. This makes it impossible to use them together in expressions, meaning there is a very strong compile-time check against accidentally comparing time_points from different clocks.

!!! info "Ye Olden Days of C"
    When using C time APIs, accidentally mixing up "arbitrary" timestamps (e.g. time since boot) and "absolute" timestamps (e.g. time since UNIX epoch) is a big issue that the language provides very little protection against. This is one of the primary reasons to use std::chrono over such raw integer types.

Internally, each time point is simply stored as a duration (so, a single integer) representing the time since (or before) the epoch. What the epoch represents depends on the clock -- it could be the time since the device booted, or the time since Jan 1 1970, or something else entirely. You can get this duration by calling the `time_since_epoch()` method. You can also construct a `time_point` with a `duration` value to create one containing a specific time.

Just like `duration`, `time_point` supports standard arithmetic operations, though only addition and subtraction:

```cpp
RealTimeClock::time_point timeA(5s);

// Adding a duration to a time_point returns an offset time_point
auto timeB = timeA + 5s; // evaluates to RealTimeClock::time_point(10s)

// Subtracting two time_points returns a duration
timeB - timeA; // evaluates to 5s
```

Notice that these operations convert between durations and time_points in the expected manner. Subtracting two time_points gets a _delta_ time (a duration), and the compiler won't let you add two absolute time_points as that would not make sense.

!!! warning "Chrono Type Initialization"
    `std::chrono::duration` is a plain-old-data type with no constructor, meaning that if you don't explicitly set the time value when you create one, it will be left uninitialized by the compiler.

    ```cpp
    std::chrono::milliseconds time1(1); // initialized to 1ms
    std::chrono::milliseconds time2{}; // initialized to zero
    std::chrono::milliseconds time3; // Contents undefined, likely random stack data
    ```

    However, `std::chrono::time_point` DOES have a default constructor which sets the time to zero:

    ```cpp
    RealTimeClock::time_point timePoint1(1s); // initialized to 1 second since epoch
    RealTimeClock::time_point timePoint2{}; // initialized to 0 seconds since epoch
    RealTimeClock::time_point timePoint3; // initialized to 0 seconds since epoch
    ```

    I have never really understood why this relatively arbitrary difference exists between these two types...

Clocks, meanwhile, are represented as C++ classes providing a specific interface. Specifically, they must have a static function called `now()` which returns the current time, and `duration` and `time_point` typedefs that give the types of the clock's "native" durations and time_points. 

!!! note "Who says C++ doesn't have duck typing?"
    Due to `std::chrono`'s use of templates, clocks do not need to actually extend any specific class. Any class which contains [the necessary members](https://cppreference.com/cpp/named_req/Clock) is, _by definition_, a Clock.

In ordinary desktop C++, the standard library defines [several clocks](https://cppreference.com/cpp/chrono#Clocks) for you to use. This includes `std::steady_clock`, which provides a steadily increasing time value with a completely arbitrary epoch, and `std::system_clock`, which provides the time since the UNIX epoch but _may_ experience sudden jumps if the system time changes.

In Mbed OS, we instead provide a few different clocks representing the available sources of time as described above:

```cpp
// Gets the time from the real-time clock, if supported. Resolution is seconds.
auto rtcTime = RealTimeClock::now();

// Gets the boot time timestamp from the μs ticker
TickerDataClock usClock(get_us_ticker_data());
auto usTickerTime = usClock.now();

// Gets the boot time timestamp from the LP ticker
TickerDataClock lpClock(get_lp_ticker_data());
auto lpTickerTime = lpClock.now();
```

!!! note "Pseudo-Clocks"
    As you may notice from the above code, `TickerDataClock` needs to be instantiated (passing the data of the ticker you want to read) before it can be used. This type of Clock is referred to as a pseudo-clock, and it does not quite match up with the regular std::chrono API. In particular, this API design means that time_points from the μs and LP ticker cannot be distinguished by the compiler, so some safety against bad behavior is lost. I am not entirely sure why Mbed uses this approach -- perhaps to enable more code reuse between code operating on the μs and LP tickers?

    Also, it's worth noting that `TickerDataClock` is only a lightweight wrapper class around the actual ticker logic, so you can create as many instances as you like and don't need to worry about sharing them around. All instances will return the same time from now().

### Converting and Rounding Times

Time conversions in std::chrono work in a very intelligent way. Suppose you're trying to convert a duration of type A into a duration of type B. Well, if the duration is exactly representable in type B without any loss of precision, this can actually be done implicitly by the library, with no casting required.

```cpp
void wait_ms(std::chrono::milliseconds ms);

// Seconds can be converted exactly to milliseconds, so this will convert automatically
wait_ms(1s);

// This will NOT compile because there is a loss of precision when converting us to ms
wait_ms(7500us);
```

If you DO need to perform a conversion that loses precision, there are several tools for that. The most basic operation is [`std::chrono::duration_cast`](https://cppreference.com/cpp/chrono/duration/duration_cast), which was added in the initial std::chrono library. This is essentially a static_cast for durations, and will give the value in the chosen duration type, truncating any remainder and rounding towards 0.

```cpp
std::chrono::duration_cast<std::chrono::milliseconds>(7400us); // evaluates to 7ms
std::chrono::duration_cast<std::chrono::milliseconds>(7900us); // also evaluates to 7ms
std::chrono::duration_cast<std::chrono::milliseconds>(-7900us); // evaluates to -7ms
```

C++17 added some additional cast functions for doing these types of conversions with explicit rounding: `floor()`, `ceil()`, and `round()`:

```cpp
// note: floor() is identical to duration_cast except for negative numbers
std::chrono::floor<std::chrono::milliseconds>(7400us); // evaluates to 7ms
std::chrono::floor<std::chrono::milliseconds>(7900us); // also evaluates to 7ms
std::chrono::floor<std::chrono::milliseconds>(-7900us); // evaluates to -ms

std::chrono::ceil<std::chrono::milliseconds>(7400us); // evaluates to 8ms
std::chrono::ceil<std::chrono::milliseconds>(7900us); // also evaluates to 8ms

std::chrono::round<std::chrono::milliseconds>(7400us); // evaluates to 7ms
std::chrono::round<std::chrono::milliseconds>(7900us); // evaluates to 8ms
```

!!! info "Conversion Functions"
    If you are using C++17 or later, I would recommend using `floor()`/`ceil()`/`round()` in place of `duration_cast()` when converting between integer durations, as these functions make it much clearer to someone reading your code how rounding is being performed. The only exceptions would be if you _really_ care about performance, or you are positive you _don't_ care about how rounding is performed in this instance.

If you want to find out what the remainder is after casting, the modulo operation is useful for that. Suppose (fictional example) that you wanted to split up a wait time into whole milliseconds and fractional ones:
```cpp
void waitSpecificTime(std::chrono::microseconds waitTime) {
    const auto wholeMs = std::chrono::floor<std::chrono::milliseconds>(waitTime);
    const auto remainder = waitTime % wholeMs;
    
    // note that the Mbed wait functions do not quite work like this, this is just an example
    wait_ms(wholeMs);
    wait_us(remainder);
}
```

Durations can also be converted into floating-point seconds, but the syntax for this one is a little weird and is something you just have to remember:

```cpp
std::chrono::duration_cast<std::chrono::duration<float>>(7400us).count(); // returns approx. 0.0074
```

!!! warning "Floating Point Durations"
    As you can see above, durations can also be instantiated using floating-point types, allowing them to store a non-whole number of ticks. **However**, use of floating point types to store durations and time points is generally not recommended, especially if your code could store large time values in these types.

    This is because floating-point values are internally represented in "scientific notation", with a value and an exponent, each of which having a fixed amount of precision available. So, the larger the float becomes, the more the exponent increases and the less precision is available. This can lead to issues where, as your system operates for longer and your time values increase, you can suddenly no longer do accurate calculations on the time!

    As long as you know that the delta between timestamps is not excessively large, it's OK to do something like
    ```cpp
    float deltaSeconds(std::chrono::microseconds a, std::chrono::microseconds b) {
        const auto deltaUs = b - a;
        return std::chrono::duration_cast<std::chrono::duration<float>>(deltaUs).count();
    }
    ```
    This ensures that only the relative timestamp gets represented as a float.

    However, do NOT do it like:
    ```cpp
    float deltaSeconds(std::chrono::microseconds a, std::chrono::microseconds b) {
        const float deltaSec = std::chrono::duration_cast<std::chrono::duration<float>>(b).count() - 
            std::chrono::duration_cast<std::chrono::duration<float>>(a).count();
        return deltaSec;
    }
    ```
    This way converts the *absolute* timestamps into floats, meaning that the precision of the result will **vary based on the absolute values of `a` and `b`** (very bad, major source of bugs).


### Printing Times

Traditionally, printing out times has been one of the rougher spots with the std::chrono library.

If you wish to print time values with `printf()`-style functions, there are two steps involved: first, you have to convert your time_point or duration into the desired units that you want to print. Then, you have to call the `counts()` function to get the raw ticks value. Also, you must take care to use the correct format specifier corresponding to the `Rep` type of the duration. This is _usually_ int64_t, but can vary if, for instance, something defines a custom duration type. Luckily, modern compilers will warn if you get it wrong.

```cpp
// This provides macros like PRIi64 for printing fixed width integer types
#include <cinttypes>

<snip>

TickerDataClock::duration someDuration = <...>;

// This prints the duration as whatever the native counts type of TickerDataClock::duration is (in this case microseconds).
printf("someDuration is %" PRIi64 "\n", someDuration.count());

// It's better to be explicit about the units, especially if the duration type is not one of the standard ones like std::chrono::microseconds.
// This makes sure your print isn't broken if the duration type changes.
printf("someDuration is %" PRIi64 "ms\n", std::chrono::round<std::chrono::milliseconds>(someDuration).count());

// Printing time_points is similar, except that you have to convert to a `duration` first
RealTimeClock::time_point someTimePoint = <...>;
printf("someTimePoint is %" PRIi64 "s since epoch\n", std::chrono::round<std::chrono::seconds>(someTimePoint.time_since_epoch()).count());
```

If you use the iostream library, e.g. `std::cout`, printing times is easier. C++20 added an [`operator<<` overload](https://en.cppreference.com/cpp/chrono/duration/operator_ltlt) for durations that dumps them as text, e.g. "1000ms" or "50us". Before C++20, this did not exist and working with iostreams was similar to above: you had to convert to an integer using `.count()` and deal with units manually.

!!! warning "`iostream` Considered Harmful"
    This author does not recommend the iostreams library for embedded use. Using it will generally lead to significantly increased code size and much longer compile times. Additionally, its reliance on attaching permanent state to global streams (e.g. to control number formatting) does not work well for larger (and especially multithreaded) applications. And trying to format floating-point numbers in a specific way with iostreams is liable to rapidly make you tear your own hair out.

If you use the [fmt](https://fmt.dev) library, this appears to natively support chrono durations with [their own format syntax](https://fmt.dev/12.0/syntax/#chrono-format-specifications), though it does appear to default to treating them as dates. I have not used fmt myself, but definitely wish to become more familiar with it soon!

### std::chrono for Hardware Counters

In embedded programming specifically, `std::chrono` is very suited for another use: working with time values that come out of a hardware counter that runs at some arbitrary frequency. 

In desktop applications, and with the Mbed time sources, the OS generally does the scaling of the time from whatever unit the hardware uses into standard time units like micro/nanoseconds. However, if you are using a hardware timer or counter at the register level, it usually counts at the rate of some hardware clock that is not a nice round frequency. Converting these counts into standard units is not trivial.

For example, let's say you have a TIMER peripheral that counts at 144MHz, and you want to read out the value of this timer in nanoseconds. The naive way to do this would be to use integer multiplication and division:

```cpp
uint32_t count_ns = TIMER->COUNT * 1000000000 / 144000000;
```

However, doing it this way is a bad idea, because it's quite vulnerable to integer overflow. If the value of `COUNT` is larger than 4 counts (!), then the multiplication will overflow a uint32_t and you will get a garbage result.

!!! info
    We also cannot use floats for this conversion in most cases because, as mentioned earlier, the resulting precision would depend on the absolute value of the time.

There are various ways to get around this issue. You could cast the numbers to int64_t, or you could simplify the "fraction" by removing all the zeros:

```cpp
uint32_t count_ns = TIMER->COUNT * 1000 / 144;
```

This produces the same result, but the multiplication will still overflow if `COUNT` is larger than about 4.2 million. It also required manually simplifying the conversion based on the counter frequency, which is not ideal.

What if I told you that we can do this automatically and in the most optimal manner by using std::chrono instead? Let's define the following:

```cpp
constexpr uint32_t timer_freq = 144000000; // Hz
typedef std::chrono::duration<int64_t, std::ratio<1, timer_freq>> timer_counts;
```

This creates a custom duration type where the native representation is equal to counts from the hardware timer. Now we can create an easy way to read our timer:

```cpp
timer_counts readTimerCounts() {
    return timer_counts(TIMER->COUNT));
}

std::chrono::nanoseconds readTimerNs() {
    return std::chrono::duration_cast<std::chrono::nanoseconds>(readTimerCounts());
}
```

Now we can use native chrono operations to read the time in SI units without needing to do any math ourselves!

Let's take a closer look at that duration_cast in the second function. This cast is doing the bulk of the work here by converting between two time units with different scalings. Internally, this cast does two steps at compile time:

1. Find the common Rep type as the widest of the source and destination Rep types. In this case that would be `int64_t`.
2. Find the conversion factor by dividing the Period of the source duration by the Period of the destination destination, then simplifying the resulting fraction. In this case this would divide 1/144M (`timer_counts::period`) by 1/1B (`std::chrono::nanoseconds::period`), resulting in a fraction of 125/18. In other words, for every 18 timer counts, the time advances by 125ns.

Then, at runtime, it uses the conversion factor computed above to convert between counts and nanoseconds. In this case, it would end up doing: (counts * 125) / 18.

This is neat because we still do everything using integer division, meaning we aren't subject to floating point issues, but we use the most optimal division to avoid overflow as much as possible. Note that instead of `duration_cast`, you could also use `round()` or `floor()` or `ceil()` as discussed earlier.

Unfortunately, there is one downside here for embedded devices: these operations operate on int64_ts, which means they will be a bit slower than using int32_ts since ARM embedded CPUs only have 32-bit registers, so the core needs to shuffle multiple values around between memory and registers for each calculation. There is no way to avoid this issue entirely (unless you redefine all the standard chrono types using int32_ts and accept a much higher risk of overflow), but there is a way to work around it.

Suppose you need to implement a busy wait using the hardware timer. You might think that the easiest way to do it would be this:

```cpp
void hw_timer_sleep(std::chrono::microseconds sleepTime) {
    const auto endTime = readTimerNs() + sleepTime;
    while(readTimerNs() < endTime) {}
}
```

However, if you do it this way instead:

```cpp
void hw_timer_sleep(timer_counts sleepTime) {
    const auto endTime = readTimerCounts() + sleepTime;
    while(readTimerCounts() < until) {}
}
```

then you avoid a great deal of expensive runtime conversions, and you will sleep more accurately. Even though it takes its  `timer_counts`, this function can still be called with microseconds, since microseconds are losslessly convertible to `timer_counts`:

```cpp
hw_timer_sleep(10us); // OK, converts automatically
```

Even better, since the argument is a constant, the above call does the microseconds to counts conversion _at compile time_, meaning that no time conversions are done at runtime at all!

As you can see, using `std::chrono` to convert hardware counters makes a lot of the tough logic significantly simpler and avoids a lot of scary edge cases, though it does perform slightly worse than direct 32-bit integers. The only real limitation of this method is that the counter frequency _must_ be known at compile time -- if it can vary at runtime, then you will have to implement your own logic to do the conversions. If you want to really dive into std::chrono, go ahead and make a `Clock` class for your hardware timer! It's not difficult, but I will leave it as an exercise for the reader.

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