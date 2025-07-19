# I²C

I²C, short for Inter-Integrated-Circuit, is a two-wire bus commonly used to move data between chips on a circuit board. Mbed supports I2C communication via the `I2C` and `I2CSlave` classes. This doc page will provide an overview of how to use I2C in Mbed OS.

## A brief overview of I²C

![I2C topology diagram](https://www.analog.com/en/_/media/analog/en/landing-pages/technical-articles/i2c-primer-what-is-i2c-part-1-/36684.png?la=en&w=900)

First let's begin with a brief overview of how the I²C bus works in hardware. I²C uses two data lines, SDA (**S**erial **DA**ta) and SCL (**S**erial **CL**ock). Both data lines are *open-drain*, meaning that each I²C device can either connect the line to ground or leave it floating. If no device is connecting the line to ground, a pullup resistor (usually between 1kΩ and 10kΩ) pulls the line up to a logic high. This system means that multiple chips can safely use the bus lines without the risk of one device trying to output a high while another device outputs a low.

In an I²C bus, there must be one or more master devices, which can initiate reads and writes, and one or more slave devices, which can accept reads and writes.

!!! note
    Referring to I²C devices as masters and slaves is problematic terminology, which we would like to move away from over time. The accepted newer terms are 'controllers' and 'targets'. However, Mbed OS has not been updated to use the newer terms yet, as this would be a badly breaking change.

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

It is possible to run I²C at speeds other than these three frequencies (useful, for instance, for debugging signal integrity issues by seeing if behavior is different at a lower frequency). However, some Mbed targets use precalculated timings and may not support arbitrary I²C frequencies.

!!! warning
    Due to how I²C works, if multiple devices are sharing a bus, you cannot go faster than the maximum bus speed of any of the devices.  Otherwise, slower devices may misinterpret messages    that are too fast for them and cause interference on the bus.
    
    For example, if you have two 400kHz devices and one 100kHz device on a bus, you must run the entire bus at 100kHz!      

### Transactions

All activity on the I²C bus takes the form of a transaction. Each transaction begins with a start condition, a direction indicator (read/write), and a slave address. If the slave is present on the bus and sees the start operation, it will respond with an acknowledge (ACK) indicating it's there and ready to listen. If no slave responds, this is a **N**ot **ACK**nowledge (NACK), and the master stops the transaction and returns an error to the application.

Once the slave acknowledges a write operation, the master can then send as many bytes as it wishes. The slave has the option to NACK each byte, meaning that a slave device can potentially indicate an error or out-of-data condition by NACKing at some point during the write.

For a read operation, on the other hand, the master supplies the clock and the slave device must output bytes over the bus. The master controls the number of bytes read, and the slave cannot NACK after the read starts.

Once the read or write operation completes, whether successful or with an error, the master must end the transaction. This is done either by sending a Stop condition (which relinquishes the bus and potentially allows another master to initiate a transaction) or a Repeated Start condition, which initiates another transaction without reliquishing the bus.

!!! note
    Most I2C target devices don't make a distinction between a stop/start and a repeated start, but a few do! (looking at you U-Blox)

### More Info
The above sections should tell you everything you need to know about I²C in order to use the Mbed `I2C` class and communicate with I2C target devices. 

For a deeper dive into I²C, including physical layer signaling, [this page from Analog Devices](https://www.analog.com/en/resources/technical-articles/i2c-primer-what-is-i2c-part-1.html) looks like a good reference! (it's also where I got the above image)

## I2C in Mbed

In Mbed, the `I2C` class allows using the I²C bus as a master device.

There are three different forms of the I²C API usable via this class:       
<ul>     
<li>Transaction-based I²C</li>       
<li>Single-byte I²C</li> 
<li>Asynchronous I²C</li>
</ul>    
         
All three of these APIs let you execute I²C operations, but they work differently.      
         
<h1>Transaction-Based API</h1>        
         
The simplest API, which should be appropriate for most use cases, is the transaction-based API, which is       
accessed through the \link I2C::read(int address, char *data, int length, bool repeated) read() \endlink and the         
\link write(int address, const char *data, int length, bool repeated) write() \endlink functions.  These functions       
execute an entire I²C transaction (the start condition, address, data bytes, and stop condition) in a single  
function call.     
         
The bytes to be read/written are passed in through an array, which requires that you can predict the 
size of the data ahead of time.  If this information is not known, you may want to use the single-byte API instead       
(see below).       
         
Example of using the transaction-based API to read the temperature from an LM75BD:
```cpp
#include "mbed.h"  
I2C i2c(I2C_SDA , I2C_SCL); 
const int addr7bit = 0x48;      // 7-bit I2C address   
const int addr8bit = 0x48 << 1; // 8-bit I2C address, 0x90         
         
int main() {       
    char cmd[2];   
    while (1) {    
        cmd[0] = 0x01;    
        cmd[1] = 0x00;    
         
        // read and write takes the 8-bit version of the address.  
        // set up configuration register (at 0x01)     
        I2C::Result result = i2c.write(addr8bit, cmd, 2);
         
        if(result != I2C::ACK)        
        {
  // Chip not accessible, handle error....   
        }
         
        ThisThread::sleep_for(500);   
         
        // read temperature register  
        cmd[0] = 0x00;    
        i2c.write(addr8bit, cmd, 1, true); // Set repeated to true so that we don't give up the bus after this transactio
        i2c.read(addr8bit | 1, cmd, 2);         
         
        float tmp = (float((cmd[0]<<8)|cmd[1]) / 256.0); 
        printf("Temp = %.2f\n", tmp); 
  }      
}        
```
         
         
<h1>Single-Byte API</h1>  
         
The single-byte API consists of the \link I2C::start() start() \endlink, \link I2C::write_byte() write_byte()  
\endlink, \link I2C::read_byte() read_byte() \endlink, and \link I2C::stop() stop() \endlink functions.        
With the single-byte API, you have manual control over each condition and data byte put onto the I2C bus.      
This is useful for dealing with devices which can return variable amounts of data in one I2C operation,        
or when you don't want to create buffers to store the data.  However, this API is more verbose than the        
transaction-based API and will have a bit more overhead since there's more code executing per byte.
         
The following is an example that accomplishes the same thing as the above code, but using the single-byte API. 
@code    
#include "mbed.h"  
I2C i2c(I2C_SDA , I2C_SCL); 
const int addr7bit = 0x48;      // 7-bit I2C address   
const int addr8bit = 0x48 << 1; // 8-bit I2C address, 0x90         
         
int main() {       
    while (1) {    
        // read and write takes the 8-bit version of the address.  
        // set up configuration register (at 0x01)     
        i2c.lock();
        i2c.start();      
        I2C::Result result = i2c.write_byte(addr8bit); // Write address, LSBit low to indicate write 
        i2c.write_byte(0x01);         
        i2c.write_byte(0x00);         
        i2c.stop();
        i2c.unlock();     
         
        if(result != I2C::ACK)        
        {
  // Chip not accessible, handle error....   
        }
         
        ThisThread::sleep_for(500);   
         
        // Set register to read       
        i2c.lock();
        i2c.start();      
        i2c.write_byte(addr8bit); // Write address     
        i2c.write_byte(0x00);         
        // To create a repeated start condition, we do not call stop() here  
         
        i2c.start();      
        i2c.write_byte(addr8bit | 1); // Write address, LSBit high to indicate read      
         
        // Read the two byte temperature word   
        uint16_t temperatureBinary = 0;         
        temperatureBinary |= static_cast<uint16_t>(i2c.read_byte(true)) << 8;
        temperatureBinary |= static_cast<uint16_t>(i2c.read_byte(false)); // send NACK to indicate last byte   
        i2c.stop();
        i2c.unlock();     
         
        float tmp = (float(temperatureBinary) / 256.0);
        printf("Temp = %.2f\n", tmp); 
  }      
}        
@endcode 
         
\attention If a single I2C object is being shared among multiple threads, you should surround usage of the     
single-byte API with \link I2C::lock() lock() \endlink and \link I2C::unlock() unlock() \endlink.  This        
ensures that a transaction by one thread is not interrupted by another.  It may also improve performance       
because the backing mutex will not need to be locked for each byte.
         
<h1>Asynchronous API</h1> 
         
The asynchronous API allows you to run I²C operations in the background.  This API is only        
available if your device has the I2C_ASYNCH feature.  To use this API, use \link I2C::transfer() transfer() \endlink     
to start an operation and \link I2C::abort_transfer() abort_transfer() \endlink to stop it.  Alternately, use the        
\link I2C::transfer_and_wait() transfer_and_wait() \endlink function to block the current thread until         
the transfer finishes.    
         
Some devices implement these features using DMA, others use interrupts, so be mindful that there may still be  
significant CPU usage if you have multiple and/or high-rate transfers going on.
         
<h1>A Note about Addressing</h1>      
Most I²C devices make use of 7-bit addresses (see <a href="https://www.i2c-bus.org/addressing/">here</a> for details).  
Mbed OS, however, works with addresses in 8-bit format, where the least significant bit specifies if the transaction     
is a read (1) or a write (0).  Due to this, you will generally need to use bitshifts and bitwise ORs when passing        
addresses to I2C functions.  See the documentation on each function for details.         
         
I²C also has a <a href="https://www.i2c-bus.org/addressing/10-bit-addressing/">10-bit addressing mode</a>, where        
the address is sent in two physical bytes on the bus.  Some, but not all, Mbed targets support this mode -- refer        
to your MCU datasheet and your target's HAL code for details.  For 10-bit addresses, use the same format to    
pass them to I2C functions -- shift them left by one and set the LSBit to indicate the read/write direction.   
On MCUs that do not natively support 10-bit addressing, you can emulate support by using the single-byte API   
to send two address bytes; see the linked page above for details.  
         
<h1>Other Info</h1>
         
The I2C class is thread-safe, and uses a mutex to prevent multiple threads from using it at the same time.     
         
\warning Mbed OS requires that you only create one instance of the I2C class per physical I²C bus on your chip.         
This means that if you have multiple sensors connected together on a bus, you must create one I2C object at the
top level and pass it in to the drivers for each sensor.  Violating this directive will cause undefined        
behavior in your code.   