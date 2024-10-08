# Flash Espressif Devices via Network using a Raspberry Pi

## TL;DR

This article enables you to flash an Espressif ESP8266/ESP32 device remotely with a Raspberry Pi attached realizing a TCP/RFC2217 to UART bridge.

## Why?

There are many reasons to have such a setup.
My personal one is that my typical working environment for ESP development is dockerized and I do not have access to a physically attached serial or USB-to-serial device.
So I was looking for a simple and elegant solution to flash the ESP devices remotely over network.

## How?

The bridge is realized by using the RFC2217 protocol.
It describes how to add control commands to the data stream in order to change parameters of the serial device like baudrate.
But it also allows to control signals used for flow control like DTR and RTS which are typically used to bring the ESP into the right state to be flashed.

Luckily, the [`pyserial`](https://github.com/pyserial/pyserial) Python package already supports RFC2217.
And since the standard [`esptool`](https://docs.espressif.com/projects/esptool/en/latest/esp32/) to flash the ESP devices uses `pyserial` internally, it also supports it.

All done? Well, almost. A minor part is missing. With the standard tooling you can already communicate with the ESP out of the box. But to bring it into the right flash state and perform resets, we need two additional GPIO pins.

The script [`rfc2217_server.py`](https://github.com/pyserial/pyserial/blob/master/examples/rfc2217_server.py) is based on the examples from `pyserial`. A very few lines of code have been added to hook into the RTS/DTR update functions to map it to GPIO pins of the Raspberry Pi.

## Prerequisites

You need some pieces of hardware, of course:

1. Raspberry Pi, e.g. the Zero W model is largely sufficient
1. ESP8266/ESP32 board
1. Set of cables to connect the boards

### Wiring Raspberry Pi and ESP

Connect the two boards as shown below. The minimal connection is RXD, TXD and GND, if you bring the ESP manually into flash mode, e.g. via buttons that may be present on your ESP board.
However, the recommendation is to also connect RESET and FLASH to let the flash script bring the ESP into the right state and back to life.
The 3V3 line is only needed if you ESP board does not have its own power supply and you want to have it powered via the Raspberry Pi.

```
Raspberry Pi                           ESP8266
--------------------+                  +----------------
                    |                  |
 ( 1) 3V3           |<--- optional --->|  3V3
 ( 6) GND           |<---------------->|  GND
 ( 8) GPIO14 / TXD  |<---------------->|  RXD
 (10) GPIO15 / RXD  |<---------------->|  TXD
 (15) GPIO22        |<---------------->|  RESET
 (19) GPIO10        |<---------------->|  FLASH / GPIO0
                    |                  |
--------------------+                  +----------------
```

## Installation

Before running through the steps below for the Raspberry, a base image is assumed to be installed already.
Since no desktop is needed a minimal Raspberry Pi OS Lite is recommended.

### Configure Raspberry Pi UART

Enable UART and map PL011 to GPIO14 and GPIO15.

1. Add `dtoverlay=disable-bt`at the end of `/boot/firmware/config.txt` to switch off bluetooth and map the PL011 UART pins.

   ```
   sudo nano /boot/firmware/config.txt
   ```

1. Disable kernel logs into UART by removing the arguments like `console=serial0,115200` from `/boot/firmware/cmdline.txt`.

   ```
   sudo nano /boot/firmware/cmdline.txt
   ```

1. Disable the modem initialization service.

   ```
   sudo systemctl disable hciuart
   ```

1. Reboot the Raspberry Pi.

   ```
   sudo reboot
   ```

### Install Raspberry Pi RFC2217 Server

To start the TCP to UART bridge run the steps below.

1. Install required tools by executing

   ```
   sudo apt install python3-pip python3-virtualenv git
   ```
1. Create a working copy of this repository using git.

   ```
   git clone https://github.com/sharkfox/raspberrypi-esp-flash.git
   cd raspberrypi-esp-flash
   ```

1. Create a Python virtual environment and install the dependencies.

   ```
   python3 -m virtualenv .venv
   source .venv/bin/activate
   pip install -r requirements.in
   ```

1. Run the server script.

   ```
   python3 rfc2217_server.py /dev/ttyAMA0
   ```

1. The Raspberry Pi now waits for clients to connect and establishes a TCP to UART bridge to `/dev/ttyAMA0`.

After reboot it is sufficient to switch into the Python environment by executing `source .venv/bin/activate` and directly starting the script from the last step.

**Note:** By default DTR and RTS are mapped to GPIO10 and GPIO22. Run the `--help` dialog of the RFC2217 server to get more information about the parameters to change them. 

## Flash the ESP8266/ESP32

Now that the Raspberry Pi is set up and waiting for input use the regular [`esptool`](https://docs.espressif.com/projects/esptool/en/latest/esp32/) on your local machine or Docker environment to flash the images to the ESP device via network.
The essential trick is to not use a local serial device but a network host while applying the RFC2217 protocol. Replace `rpi-zero.local` with the hostname or IP address of your Raspberry Pi.

```
python3 -m esptool --port rfc2217://rpi-zero.local:2217 write_flash 0x0 project.ino.bin
```

Speed-up the transfer selecting a higher UART baudrate by adding the parameter `-b 921600`.
