# DMA SPI for STM32

Mbed CE supports DMA-based SPI on most STM32 device families.  This allows SPI transfers of arbitrary length to occur in the background with no involvement from the main CPU except at the start and end of the transfer.

## What Is DMA?

DMA, or Direct Memory Access, is a feature of many microprocessors that allows data to be moved around without intervention by the host processor.  Broadly speaking, a DMA controller is like a simple coprocessor which is programmed by means of a "descriptor" -- instruction data which tells it what to do.  By creating the right descriptor, you can do almost anything with a DMA controller - from a simple memory-to-memory copy to complete automation of peripherals on a chip.

When used with SPI peripherals, DMA is very powerful.  Normally, asynchronous SPI on STM32 devices is done with interrupts, but this has major disadvantages.  The SPI peripheral can only store a few bytes at a time, so it has to keep interrupting the CPU to get it to queue up more data.  This will eat up a lot of your CPU time, and above a certain SPI frequency, the CPU simply can't keep up anymore.  But since DMA can operate without the CPU's intervention, it gets around these limitations. Once a DMA SPI transaction is started (which normally takes 1s to 10s of microseconds), it will run completely in the background without CPU intervention.

## Using DMA SPI

To enable DMA SPI, create an SPI object, then call:

```cpp
spi.set_dma_usage(DMA_USAGE_ALWAYS);
```

Then, do asynchronous operations on it using `SPI::transfer()` (nonblocking) and `SPI::transfer_and_wait()` (blocking).  These operations will now use DMA in the background, and be much more efficient!

## DMA Channel Allocations

On STMicro devices, DMA is divided into DMA controllers and DMA channels (though some devices refer to DMA channels as streams).  Each DMA controller is like a mini coprocessor, with its own interface to access memory and peripherals.  Each DMA channel is like a "thread" that the coprocessor executes -- it can only execute one channel at a time, and it switches between the channels as they are ready to execute.  Generally, SPI data doesn't need to be transferred very quickly, since the SPI bus runs much slower than the processor clock.  So, it usually isn't an issue that multiple channels have to share the same DMA controller.

Where the channel allocation can get a bit trickier is when you're using DMA in your own code and also want to use DMA SPI in Mbed.  You will need to make sure that you don't use the same DMA channels that Mbed does.  To see what channels Mbed is using on your device, refer to the header `mbed-os/targets/TARGET_STM/TARGET_STM32xx/stm_dma_info.h` (where `TARGET_STM32xx` is replaced by the correct folder for your device family).  This header contains the list of DMA channels that Mbed uses on your device.  Also, see below for more information on how DMA channels are allocated.

### Processor-Specific Notes

- **STM32F4xx and STM32F7xx:** DMA may not be used on both SPI5 and SPI6 at the same time, since these peripherals share their DMA channels.  Either one of SPI5 and SPI6 may be used with DMA so long as the other one is not.
- **STM32H7xx:** DMA may not be used on SPI6, since this SPI is not connected to the main DMA controller.

## DMA Line Mapping

STM32 devices generally fall into three categories with regard to how they map their peripherals to their DMA controller.  The way that this mapping works is an important factor into whether you can use DMA to create the design that you want, especially if you intend to use DMA features in your own applications.

Some devices simply use an "OR" gate to combine several peripherals' DMA request lines together into the request line for each DMA channel.  These devices have no flexibility in their DMA configurations -- it's up to you to use the correct peripherals.  For example, if you wanted to use both SPI1 and I2C3 with DMA on STM32F3, since these two peripherals share a single DMA channel, you'd be out of luck.  These "completely fixed" devices include:

- STM32F0
- STM32F1
- STM32F3
- STM32L1

!!! note
    None of these devices have SPI peripherals that conflict with each other, so since Mbed only uses DMA for SPI, if you are not using DMA for other things in your own application, you shouldn't need to worry about the channel assignments.

Other devices use a "request selection" model.  On these devices, each DMA channel has several possible request lines it can use, each of which goes to one peripheral.  These devices generally have at least two possible channel+request combinations for each peripheral, so you have more possible configurations for DMA.  However, you can still run into trouble if both possible options for a peripheral are used by other DMA channels.

These "request selection" devices include:

- STM32F4
- STM32F7
- STM32L4
- STM32L0

Lastly, other devices use a DMA Mux-based architecture.  The DMA mux is a separate input stage to the DMA controller which allows any DMA request source to be routed to any DMA channel.  On these devices, the selection of DMA channels can be done completely arbitrarily and basically is just done by Mbed in numeric order.  DMA Mux devices include:

- STM32L4+
- STM32H7
- STM32G0
- STM32G4
- STM32U5
- STM32WB
