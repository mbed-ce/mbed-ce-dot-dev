# Networking Basics

One of the primary purposes of Mbed OS is to build network-connected embedded systems. To that end, Mbed supports wired Ethernet networking, as well as wireless networking via Wi-Fi, Bluetooth, and 802.15.4 mesh networks. This page will focus on using Ethernet and Wi-Fi to talk to other devices over Internet Protocol (IP) networks.

## Supported Hardware

In order to use Wi-Fi/Ethernet networking, you need to have the right hardware in your MCU and/or target board. A summary of this hardware follows:

| Connection Type | MCUs Supported | External Hardware Needed | Interface to External Hardware |
|-----------------|----------------|--------------------------|--------------------------------|
| Ethernet        | All with EMAC driver support (see [here](https://mbed-ce.github.io/mbed-ce-test-tools/drivers/DEVICE_EMAC.html)) | Standards-compliant Ethernet PHY* and magnetics. | RMII (in most cases) |
| Wi-Fi           | PSOC6x, STM32H7xx | [Infineon (Cypress) WHD Wi-Fi/BT modules](https://www.infineon.com/products/wireless-connectivity/airoc-wi-fi-plus-bluetooth-combos/wi-fi-4) (CYW4343W, CYW43438, CYW43012) | SDIO |
| Wi-Fi           | All STM32 (due to driver license restriction) | MXChip EMW3080B | SPI or UART |
| Wi-Fi           | All | ESP8266 running ESP8266-IDF-AT AT command firmware | UART (optionally with flow control) |
| Wi-Fi           | All | ESP32 running ESP-AT AT command firmware | UART (optionally with flow control) |

*Note that PHY model support varies by MCU. Some MCUs' drivers currently expect a specific model of phy (usually the one present on the dev board) while others have been updated to use Mbed's [generic ethernet PHY system](https://github.com/mbed-ce/mbed-os/blob/main/connectivity/drivers/emac/CompositeEMAC.md#phy-driver).

## Starting Up the Network

The first step towards connecting to the network is declaring your `[NetworkInterface](https://mbed-ce.github.io/mbed-os/class_network_interface.html)` object and setting it up. This is done a bit differently between Ethernet and Wi-Fi, so we will cover both.

### Ethernet

First you need to obtain an `EthernetInterface`. This class is an implementation of `NetworkInterface` for Ethernet communication. Generally, you can get the default ethernet interface for your board by calling the `EthernetInterface::get_default_instance()` function. Then, you need to call the interface's connect() function to connect to the Ethernet network. The below code shows how to do this and how to print the returned error code, if any.

```cpp
#include <EthernetInterface.h>

EthernetInterface * const ethInterface = EthernetInterface::get_default_instance();

void main() {
    const nsapi_status_t connectRet = ethInterface->connect();
    if (connectRet != NSAPI_ERROR_OK) {
        tr_error("Error connecting: %s", nsapi_strerror(connectRet));
        // wait for the network or abort the application as desired
    }
}
```

This basic snippet will connect to the network and acquire an IP address via DHCP. This is what you want for most networks, but it does require a DHCP server to be running on the network (most wi-fi routers will do this). If there's no DHCP server, you will get the error `NSAPI_ERROR_DHCP_FAILURE`.

If you want to instead use a static IP, you can do that by passing the network parameters to the `set_network()` function. This signature requires the IP address, the [CIDR netmask](https://www.teltonika-networks.com/newsroom/understanding-netmask-a-comprehensive-guide-to-network-subnetting), and (if you want to access the Internet) the gateway (router) address.

```cpp
    SocketAddress ip("192.168.1.5");
    SocketAddress netmask("255.255.255.0");
    SocketAddress ip("192.168.1.1");
    ethInterface->set_network(ip, netmask, gateway);

    // Now call ethInterface->connect() like normal...
```

!!! note "SocketAddress"
    Mbed uses the `[SocketAddress](https://mbed-ce.github.io/mbed-os/class_socket_address.html)` class to represent IP addresses and IP/port pairs. `SocketAddress` instances may be created from the string representation of an IP address and its raw bytes.

### Wi-Fi

Connecting to wi-fi is pretty similar, except you need to declare a `[WiFiInterface](https://mbed-ce.github.io/mbed-os/class_wi_fi_interface.html)` object, and you need to provide the network type, name, and password before you can connect to it.

#### Scanning

To find available networks, you can use the `WiFiInterface::scan()` function. The scan() function takes an array of `[WiFiAccessPoint](https://mbed-ce.github.io/mbed-os/class_wi_fi_access_point.html)` objects to output data into. It returns the number of networks detected, or a negative error code on error.

I find the easiest way to work with it to be using an `std::vector`. This example shows how to scan the networks and print them to the console.

```cpp
#include <WiFiInterface.h>

WiFiInterface * const wifiInterface = WiFiInterface::get_default_instance();

void printNetworks() {
    constexpr size_t maxNetworks = 15; // each network uses 48 bytes RAM
    std::vector<WiFiAccessPoint> scannedAPs(maxNetworks);

    auto ret = wifiInterface->scan(scannedAPs.data(), maxNetworks);
    if(ret < 0) {
        printf("Error performing wifi scan: %s\n", nsapi_strerror(ret));
        // TODO handle error as desired
    }

    if(ret == 0) {
        printf("No networks detected.\n");
    }
    else {
        printf("Detected %d access points: \n", ret);

        // Shrink the vector to fit the number of networks actually seen
        scannedAPs.resize(ret);

        for (WiFiAccessPoint& ap : scannedAPs) {
            printf("- SSID: \"%s\", security: %s, RSSI: %" PRIi8 " dBm, Ch: %" PRIu8 "\n", ap.get_ssid(),
                nsapi_security_to_string(ap.get_security()), ap.get_rssi(), ap.get_channel());
        }
    }
}
```

!!! note "PRIi8/PRIu8"
    These are printf [helper macros](https://en.cppreference.com/w/cpp/header/cinttypes.html) provided by the `<cinttypes>` / `<inttypes.h>` header. They correspond to the correct format specifier needed to print an `int8_t` and `uint8_t` respectively. If you have ever been annoyed about warnings caused by trying to print fixed size integer types (like `uint8_t`, `int16_t`, etc) then these are your best friend!

If you would like to sort the networks from best to worst signal strength, you can drop in this snippet before the `for` loop. It uses Standard Template Library functionality from `<algorithm>` to sort the list by Received Signal Strength Indicator (RSSI):

```cpp
        // Sort by RSSI from high to low.
        auto comparator = [](WiFiAccessPoint const & lhs, WiFiAccessPoint const & rhs) { return lhs.get_rssi() > rhs.get_rssi(); };
        std::sort(scannedAPs.begin(), scannedAPs.end(), comparator);
```

#### Setting Credentials and Connecting
Once you have found a network to connect to, you can then call the `set_credentials()` function to connect to it. This function accepts the SSID and password as C strings, and the security as an enumeration value (as returned by `scan()`).

For example, to connect to a WPA2-secured network called "My Network":

```cpp
wifiInterface->set_credentials("My Network", "<my password>", NSAPI_SECURITY_WPA2);
```

Or, to connect to an unsecured network called "Coffee Shop Wifi":

```cpp
wifiInterface->set_credentials("Coffee Shop Wifi", nullptr, NSAPI_SECURITY_NONE);
```

After setting the credentials, you can then join the network like normal using `connect()` and optionally `set_network()`. See the Ethernet section above for how to do this.

## The Simplest Socket (UDP)
Once you have your Ethernet or Wi-Fi interface set up, it's time to actually start sending data over the network. One of the easiest ways to do that is with a UDP socket.

### UDP Basics
UDP, short for User Datagram Protocol, is the simplest and most basic way to send data across IP networks. With UDP, you simply pass in the chunk of data that you wish to send, and the OS will wrap it in simple UDP and IP headers, then send the packet (also referred to as a "datagram") across the network immediately. There's no flow control or acknowledgement -- each packet is on its own once it's sent, and it's possible it could be dropped or reordered along the way. 

Also, as networks impose a maximum size on each individual packet, there's an upper limit to how much data you can send in one UDP datagram. In most cases, this number (referred to as the MTU or Maximum Transfer Unit) is 1472 bytes. If your data is, or could be, larger than this size, you will have to break it up into pieces. Note that there are some ways to achieve a larger MTU, including Ethernet jumbo frames and IP fragmentation, but these can be difficult to implement on bare-metal embedded devices and they are outside the scope of this document.

### Creating a UDP Socket
To make a UDP socket, we first need to construct a `UDPSocket` object, then we need to `open()` it on the desired network stack (usually the default one). This can be done like this:

```cpp
UDPSocket udpSocket;
auto err = udpSocket.open(&OnboardNetworkStack::get_default_instance());

if(err != NSAPI_ERROR_OK) {
    printf("Failed to open UDP socket: %s\n", nsapi_strerror(err));
}
```

!!! info Maximum Socket Count
    By default, Mbed OS only allocates memory for up to three user sockets to be opened at once. To increase this number, you will need to adjust the network stack configuration. See the "Network Limitations and Tuning" section below for more details.

### Binding the Socket (Or Not)
Next, if you wish this socket to have a specific local port allocated to it, you must call `bind()`. This function attempts to associate the socket with the given port number. Binding will fail if another socket is already using that port (unless you call `setsockopt()` with the `NSAPI_REUSEADDR` socket option).

```cpp
err = udpSocket.bind(12345); // Bind the socket to this port number
if(err != NSAPI_ERROR_OK) {
    printf("Failed to bind UDP socket: %s\n", nsapi_strerror(err));
}
```

If you pass 0 as the port number to `bind()`, or do not call bind() at all, your socket will be given a random unused port number in the high end of the port range. This is called an ephemeral, AKA "dynamic" port, and [will be](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) in the range 49152 through 65535. It's useful when you are going to use your socket for transmission only and don't care about the port number.

!!! warning "Ephemeral Ports"
    Currently, there is no way to determine the port number assigned to a socket using an ephemeral port. So, it's only feasible to use them in situations where you truly do not need to know the local port number.

Bind can also be passed a `SocketAddress` structure instead of just a port number. This SocketAddress should contain the IP address of a local interface, and will restrict the socket to receiving only on this interface. This is useful if, for example, you only want to provide a specific service on the Ethernet port connected to your private LAN, not a wi-fi network connected to the public internet.

Here's a quick reference table if you need it:

|Call|Effect|
|---|---|
|`socket.bind(0)`|Binds to an ephemeral (random) local port. Will receive any traffic going to that port.|
|`socket.bind(12345)`|Binds to local port 12345. Will receive all traffic going to that port.|
|`socket.bind(SocketAddress("192.168.1.10", 0))`|Binds to an ephemeral (random) local port. Will receive only traffic from the interface with IP address 192.168.1.10|
|`socket.bind(SocketAddress("192.168.1.10", 12345))`|Binds to local port 12345. Will receive only traffic coming to this port from the interface with IP address 192.168.1.10|

### Sending a Packet
If you'd like to send a packet, this is pretty easy! Just grab your payload as a byte array and your desired destination as a SocketAddress, then call `sendto()`!

```cpp
uint8_t myPayload[] = {1, 2, 3, 4, 5};
SocketAddress destAddr("192.168.1.11", 12345);
err = udpSocket.sendto(destAddr, myPayload, sizeof(myPayload));
if(err < 0) {
    printf("Failed to send: %s\n", nsapi_strerror(err));
}
```

The only slight gotcha here is that `sendto()` returns an `nsapi_size_or_error_t`, meaning that it will return a positive byte count on success, or a negative error code on error. Luckily all the nsapi error constants are already negative numbers, so you don't need to negate the error code, just don't forget to check that it's `< 0` rather than `!= NSAPI_ERROR_OK` (I definitely didn't make this mistake when writing this guide or anything...).

### Receiving Packets
Receiving UDP packets is also quite simple, and can be done using the `recv()` and `recvfrom()` functions. These functions are identical except that `recvfrom()` also gives you the address that sent the packet -- generally very useful information if you wish to send a response! The only thing to keep in mind is that you must pre-allocate a buffer large enough to contain the packet you are receiving, and pass in that buffer and its size to the receive call.

```cpp
// Rx buffer big enough to hold any UDP payload.
// NOTE: Declare this globally, not inside a function, to avoid destroying your stack space.
char rxPacketBuffer[1472];

/// <snip>

    while(true) {
        SocketAddress sourceAddress;
        auto recvResult = udpSocket.recvfrom(&sourceAddress, rxPacketBuffer, sizeof(rxPacketBuffer));
        if(recvResult < 0) {
            printf("Failed to receive UDP packet: %s\n", nsapi_strerror(recvResult));
        }
        else {
            printf("Received %zu bytes from %s", recvResult, sourceAddress.get_ip_address());
            // Bytes are available in rxPacketBuffer for your application to process
        }
    }
```

!!! warning "Rx Buffer Sizing"
    To save RAM, you may wish to make your Rx buffer smaller. However, if the Rx buffer is too small to fit the received packet, the packet will be truncated to the buffer size you passed. There is currently no indication to the application that this occurred (other than a corrupt payload). So, if you decide to use a smaller Rx buffer, be confident that your packets won't exceed this size!

### Restricting the Source Address
What if you want to ensure your UDP packets come from one specific network peer, rather than any device on the network? You can do that using the `connect()` function.

Calling

```cpp
udpSocket.connect(SocketAddress("192.168.1.12", 56789));
```

will cause your socket to only accept packets coming from this source address and port. You can change the desired peer at any time by calling `connect()` again. You can also call it with a default-constructed `SocketAddress` to remove the restriction and accept all packets.

Note that connecting your socket is not, in itself, enough to prevent spoofing of network packets. But it can be a useful start!

### Full Example

For an example application that shows off the functionality discussed in this section, see the [mbed-cli-network-example](https://github.com/mbed-ce-libraries-examples/mbed-cli-network-example) app, specifically the `udp-send` and `udp-listen` commands defined in network_demo_cmds.cpp.

## Blocking vs Non-Blocking
## TCP Client Sockets
## TCP Server Sockets
## Network Limitations and Tuning