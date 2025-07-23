# I²C

I²C, short for Inter-Integrated-Circuit, is a two-wire bus commonly used to move data between chips on a circuit board. Mbed supports I2C communication via the `I2C` and `I2CSlave` classes. This doc page will provide an overview of how to use I2C in Mbed OS.

## A brief overview of I²C

### Hardware Connections

![I2C topology diagram](https://www.analog.com/en/_/media/analog/en/landing-pages/technical-articles/i2c-primer-what-is-i2c-part-1-/36684.png?la=en&w=900)

First let's begin with a brief overview of how the I²C bus works in hardware. I²C uses two data lines, SDA (**S**erial **DA**ta) and SCL (**S**erial **CL**ock). Both data lines are *open-drain*, meaning that each I²C device can choose to either connect the line to ground or leave it floating. If no device is connecting the line to ground, a pullup resistor (usually between 1kΩ and 10kΩ) pulls the line up to a logic high. This setup means that multiple chips can safely use the bus lines without the risk of one device trying to output a high while another device outputs a low (which could cause a short).

In an I²C bus, there must be one or more master devices, which can initiate reads and writes, and one or more slave devices, which can accept reads and writes.

!!! note "Terminology Note"
    Referring to I²C devices as masters and slaves is problematic terminology, which we would like to move away from over time. The accepted newer terms are 'controllers' and 'targets'. However, most of Mbed OS has not been updated to use the newer terms yet, as this would be a badly breaking change. PRs would gladly be accepted to update this!

### Addressing

Each slave device is identified by a seven-bit address. This address must be passed to Mbed APIs in order to direct reads and writes to the correct location.

!!! info
    Each slave device on a bus must have a unique address, which can sometimes be a problem if you want multiple instances of the same chip on an I²C bus. Various methods exist to work around this issue, including chips with pin-configurable addresses and I²C multiplexers.

### Speed

The base I²C standard defines three different speeds:

- Standard mode, which runs at 100kHz
- Fast mode, which runs at 400kHz
- Fast mode plus, which runs at 1MHz

All Mbed devices which support I²C should support at least standard and fast mode. Fast mode plus support depends on the target implementation, and also requires somewhat different hardware (e.g. lower pullup resistors).

It is possible to run I²C at speeds other than these three frequencies (for instance, for debugging signal integrity issues by seeing if behavior is different at a lower frequency). However, some Mbed targets use precalculated timings and may not support arbitrary I²C frequencies.

!!! warning
    Due to how I²C works, if multiple devices are sharing a bus, you cannot go faster than the maximum bus speed of any of the devices. Otherwise, slower devices may misinterpret messages that are too fast for them and cause interference on the bus.
    
    For example, if you have two 400kHz devices and one 100kHz device on a bus, you must run the entire bus at 100kHz!

### Transactions

All activity on the I²C bus takes the form of a transaction. Each transaction begins with a start condition, a direction indicator (read/write), and a target (slave) address. If the target is present on the bus and sees the start operation, it will respond with an **ACK**nowledge (ACK) indicating it's there and ready to listen. If no target responds, this is a **N**ot **ACK**nowledge (NACK), and the master stops the transaction and returns an error to the application.

Once the target acknowledges a write operation, the master can then send as many bytes as it wishes. The target has the option to NACK each byte, meaning that a target device can potentially indicate an error or out-of-data condition by NACKing at some point during the write.

For a read operation, on the other hand, the master supplies the clock and the target device must output bytes over the bus. The master controls the number of bytes read, and the target cannot NACK after the read starts.

Once the read or write operation completes, whether successful or with an error, the master must end the transaction. This is done either by sending a Stop condition (which relinquishes the bus and potentially allows another master to initiate a transaction) or a Repeated Start condition, which initiates another transaction without relinquishing the bus.

!!! note
    Most I²C target devices don't make a distinction between a stop/start and a repeated start, but a few do! (looking at you U-Blox)

### Register Abstraction

Many (though not all) I²C target devices use a "register abstraction" on top of I2C. This allows the target device to have a _register space_ -- that is, a collection of values mapped to specific integer addresses.

In devices with a register abstraction, you usually perform a write by sending the register address you wish to write to, then the value(s) that you wish to write, in a single transaction. Reads, meanwhile, remember the last address accessed and return data from that address. So, to read an arbitrary address, you would do a write transaction containing only the register address, then (preferably) a repeated start, then a read transaction that reads the desired number of bytes.

!!! warning "Don't confuse your addresses!"
    The register address discussed above is not the same as the I²C target address. The register address is the first byte (or bytes) of the I²C data payload, by convention. The I²C target address is the first byte that appears on the wire and is specified in the I²C standard.
    
For a more concrete example of using a chip with a register abstraction (the LM75B), see the code examples below.

!!! note
    The register abstraction is a common convention implemented on top of I²C, but is in no way specified in the I²C standard. This means that different chips can, and do, implement it differently, so always read the datasheet to get the specifics!

### More Info
The above sections should tell you everything you need to know about I²C in order to use the Mbed `I2C` class and communicate with I²C target devices. 

For a deeper dive into I²C, including physical layer signaling, [this page from Analog Devices](https://www.analog.com/en/resources/technical-articles/i2c-primer-what-is-i2c-part-1.html) looks like a good reference! (it's also where I got the above image)

## I²C in Mbed

In Mbed, the `I2C` class allows using the I²C bus as a master device. 

### Initializing the Object

First, we need to create an I2C object. Its constructor takes the SDA and SCL pins, which you should (hopefully) be able to find in your board's documentation. Many (though not all) Mbed targets also define `I2C_SDA` and `I2C_SCL` macros to the "default" I²C bus pins, so that might be enough to get you started.

```cpp
#include <mbed.h>

I2C i2c(I2C_SDA, I2C_SCL);
```

!!! warning "I2C Object Instances"
    You may only create one instance of the I2C class per physical I²C bus on your chip. This means that if you have multiple sensor drivers using the same bus, you must create one I2C object at the top level and pass it in to each driver. Violating this directive will cause undefined behavior in your code.

Next, you'll want to set the frequency. The default is 100kHz, so you will probably want to increase that to 400kHz for faster speed:
```cpp
int main() {
    i2c.frequency(400000);
}
```

### Address Format
Most I²C devices make use of 7-bit addresses (see [here](https://www.i2c-bus.org/addressing/) for details). Mbed OS, however, works with addresses in 8-bit format, where the least significant bit specifies if the transaction is a read (1) or a write (0). Due to this, you will generally need to use bitshifts and bitwise ORs when passing addresses to I2C functions. See the documentation on each function for details.

I²C also has a [10-bit addressing mode](https://www.i2c-bus.org/addressing/10-bit-addressing/), where   the address is sent in two physical bytes on the bus. Some, but not all, Mbed targets support this mode -- refer to your MCU datasheet and your target's HAL code for details. For 10-bit addresses, use the same format to pass them to I2C functions -- shift them left by one and set the LSBit to indicate the read/write direction. On MCUs that do not natively support 10-bit addressing, you can emulate support by using the single-byte API to send two address bytes; see the linked page above for details. 

### Three API Types
We'll get to some real usage next, but first, we should explain that there are three different forms of the I²C API usable via this class: 

- Transaction-based I²C
- Single-byte I²C 
- Asynchronous I²C

All three of these APIs let you execute I²C operations, but they work differently. Transaction-based and single-byte are both synchronous, but the former manages the I²C transactions automatically while the latter lets you customize each step. Asynchronous I²C, meanwhile, works similarly to the transaction-based API but lets you run I²C operations in the background. The sections below will show you how to use each one.

### Transaction-Based API

The simplest I²C API is the transaction-based API, which is accessed through the [`I2C::read()`](https://mbed-ce.github.io/mbed-os/classmbed_1_1_i2_c.html#a16a28f7b5087a9493701c2b35cb77f3f) and [`I2C::write()`](https://mbed-ce.github.io/mbed-os/classmbed_1_1_i2_c.html#a81a5b394ae924d714e8ec6ab4bbdf4f2) functions.

These functions execute an entire I²C transaction (the start, address, data bytes, and stop) in a single function call. They return an `I2C::Result`, which is an enum indicating ACK, NACK, or other error. (this is a change in Mbed CE; these functions used to return integer fault codes that were harder to work with).

The bytes to be read/written are passed in through an array, which requires that you can predict the size of the data ahead of time. If this information is not known, you may want to use the single-byte API instead (see next section). 

Here's an example of using the transaction-based API to read the temperature from an [LM75BD](https://www.nxp.com/part/LM75BD) temp sensor. This sensor uses a register abstraction, and each write to it starts with a 1-byte register address. It contains two registers we care about: a 1-byte configuration register at address 1, and a 2-byte temperature register at address 0.

```cpp
#include "mbed.h"

I2C i2c(I2C_SDA , I2C_SCL);

const int addr7bit = 0x48; // 7-bit I2C address
const int writeAddr8Bit = 0x48 << 1; // 8-bit I2C address, 0x90
const int readAddr8Bit = writeAddr8Bit | 1; // 8-bit I2C address, 0x91

int main() { 
    i2c.frequency(400000);

    char cmd[2];

    cmd[0] = 0x01; // Select configuration register
    cmd[1] = 0x00; // Default value, enable chip

    // read and write takes the 8-bit version of the address. 
    // set up configuration register (at 0x01)     
    I2C::Result result = i2c.write(writeAddr8Bit, cmd, 2);

    if(result != I2C::ACK) {
        // Chip not accessible, handle error....  
    }

    while (1) {    
        // read temperature register
        cmd[0] = 0x00;
        i2c.write(writeAddr8Bit, cmd, 1, true); // Set repeated to true so that we don't give up the bus after this transaction
        i2c.read(readAddr8Bit, cmd, 2);

        float tmp = (static_cast<int16_t>(cmd[0] << 8) | cmd[1]) / 256.0f;
        printf("Temp = %.2f\n", tmp);

        rtos::ThisThread::sleep_for(500ms);
    }
}  
```

### Single-Byte API

The single-byte API consists of the [`I2C::start()`](https://mbed-ce.github.io/mbed-os/classmbed_1_1_i2_c.html#aedc96a40e844bc2efc8bbef1b0782702) [`I2C::write_byte()`](https://mbed-ce.github.io/mbed-os/classmbed_1_1_i2_c.html#a288f41e052362d155a131a80f1f4c3c0) [`I2C::read_byte()`](https://mbed-ce.github.io/mbed-os/classmbed_1_1_i2_c.html#a6f8eac061d88e58f11e99759f9609700), [`I2C::stop()`](https://mbed-ce.github.io/mbed-os/classmbed_1_1_i2_c.html#aacf6bf8e121fea8b0a92d7197b73fe27) functions.

!!! note
    Mbed CE renamed the functions previously called `write()` and `read()` to `write_byte()` and `read_byte()` to disambiguate them from the transaction-based APIs.

With the single-byte API, you have manual control over each condition and data byte put onto the I²C bus. This is useful for dealing with devices which can return variable amounts of data in one I²C operation, or for when you don't want to create a buffer to store the data. However, this API is more verbose than the transaction-based API and will have a bit more overhead since there's more code executing per byte.

The following is an example that accomplishes the same thing as the above code, but using the single-byte API.

```cpp
#include "mbed.h"

I2C i2c(I2C_SDA , I2C_SCL);

const int addr7bit = 0x48;// 7-bit I2C address   
const int writeAddr8Bit = 0x48 << 1; // 8-bit I2C address, 0x90
const int readAddr8Bit = writeAddr8Bit | 1; // 8-bit I2C address, 0x91

int main() { 
    i2c.frequency(400000);

    // read and write takes the 8-bit version of the address. 
    // set up configuration register (at 0x01)     
    i2c.lock();
    i2c.start();
    I2C::Result result = i2c.write_byte(writeAddr8Bit); // Address byte, LSBit low to indicate write 

    // Check whether we got an ACK on the address byte
    if(result == I2C::ACK) {
        i2c.write_byte(0x01);
        i2c.write_byte(0x00);
    }
    else {
        // Chip not accessible, handle error....  
    }

    i2c.stop();
    i2c.unlock();

    while (1) {
        // Set register to read 
        i2c.lock();
        i2c.start();
        i2c.write_byte(writeAddr8Bit);  
        i2c.write_byte(0x00);
        // To create a repeated start condition, we do not call stop() here  

        i2c.start();
        i2c.write_byte(readAddr8Bit); // Address byte, LSBit high to indicate read

        // Read the two byte temperature word   
        int16_t temperatureBinary = 0;
        temperatureBinary |= static_cast<int16_t>(i2c.read_byte(true)) << 8;
        temperatureBinary |= static_cast<int16_t>(i2c.read_byte(false)); // send NACK to indicate last byte   
        i2c.stop();
        i2c.unlock();     

        float tmp = temperatureBinary / 256.0f;
        printf("Temp = %.2f\n", tmp);

        rtos::ThisThread::sleep_for(500ms);
    }
}  
```

!!! warning "Thread Safety Note"
    If a single I2C object is being shared among multiple threads, you should surround usage of the      single-byte API with [`I2C::lock()`](https://mbed-ce.github.io/mbed-os/classmbed_1_1_i2_c.html#ad67df83ace240f53c1276e24a37ff84c) and [`I2C::unlock()`](https://mbed-ce.github.io/mbed-os/classmbed_1_1_i2_c.html#a5d882b4e464e4f4bd59eab6c75297bda). This ensures that a transaction by one thread is not interrupted by another. It may also improve performance because the backing mutex will not need to be locked for each byte.

### Asynchronous API

The asynchronous API allows you to run I²C operations in the background. This API is only available if your device has the `I2C_ASYNCH` feature. To use this API, use [`I2C::transfer()`](https://mbed-ce.github.io/mbed-os/classmbed_1_1_i2_c.html#a273f3dc6765aea3f2bdca0e70ae9697c) to start an operation and [`I2C::abort_transfer()`](https://mbed-ce.github.io/mbed-os/classmbed_1_1_i2_c.html#a90253410d768f60403698f5f383d1931) to stop it. Alternately, use the [`I2C::transfer_and_wait()`](https://mbed-ce.github.io/mbed-os/classmbed_1_1_i2_c.html#a273c521d9f320d5c1b84975af701890f) function to block the current thread until the transfer finishes.

Some devices implement these features using DMA and others use interrupts, so be mindful that there may still be significant CPU usage if you have multiple and/or high-rate transfers going on.

## More Examples

For more I²C examples you can try out, including a bus scanner and an example that transfers data between two Mbed MCUs over I²C, see the [I2C-examples](https://github.com/mbed-ce-libraries-examples/I2C-examples) repository!