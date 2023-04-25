# Piridium Hardware
Piridium is a CubeSat bus design inspired by TJREVERB and the concept of Pycubed. It uses a Raspberry Pi Zero as a flight computer and Iridium 9602N modem as a radio. The bus is divided into two boards, with the EPS board including six 18650 battery cells and protection and balancing circuitry, and the avionics board including everything else (flight computer, radio, telemetry ADC, inhibits and RBF, solar input charge regulators, 5V output regulator, inertial measurement unit, real time clock and CR2032 battery backup, and payload connector).

## Issues
Charge balance for current EPS prototype seems to not work, despite following datasheet recommendations exactly. Possibly a bad chip, but worth noting.

## Tech Specs
Flight Computer: Raspberry Pi Zero/Zero W

Radio: Iridium 9602N Short Burst Data

Battery: Six 18650 Li-Ion cells (builder's discretion, recommend protected 3-4Ah) in a 3P2S configuration

Battery protection: ~7.7A overcurrent protection (charge & discharge), 4.29V/cell overcharge protection with hysteresis down to 4.05V/cell, 3V/cell overdischarge protection with hysteresis up to 3.2V/cell

Solar input: 4 input connectors feeding two regulators with 25V Max (limited by the voltage divider for telemetry) with charge cutoff at 8.27V, MPPT settable using a trim pot

Output regulation: 2 isolated 5V rails up to 3A output, one for the radio, the other for the Raspberry Pi, each with latching current limiters and a watchdog timer

Payload connection: 1x SPI bus with one dedicated chip enable pin, 1x I2C bus, 1x unused configurable GPIO pin, switchable battery power (no overcurrent protection); see schematic for pinout

Telemetry: Current out and voltage in for both solar regulators, battery input voltage and current, payload battery current

Antenna connector: Standard SMA vertical

Inhibits: Two discharge inhibits (do not block charge), two RBF (one low side ground disconnect, one high side power disconnect, blocks both charge and discharge)

## Hardware notes

### Flight Computer
Raspberry Pi Zero running Raspbian OS lite. Flight software can be found here: https://github.com/Alan-JH/piridium-fsw

### Iridium Modem
Defaults to off when the bus is powered, only turns on when S_MODEM_ON_OFF is pulled high by the Raspberry Pi on GPIO pin 20. There is an LED to indicate when the modem is powered, and GPIO input for ring indicator and network availability (S_RING_INDICATOR on GPIO pin 16 and S_NET_AVAIL on GPIO pin 12, respectively). Communication with the device is done through 3.3V UART, connected to the single UART bus on the Raspberry Pi. There is also a separate connector on the board to be used with a USB to UART converter in case configuration is needed without access to the Raspberry Pi. Due to its strict power requirements the modem can only be powered with the avionics board's 5V regulators, and not through the extra port. All UART connections driven by the Pi run through a 74HC126 buffer to the modem, to prevent latchup conditions if the modem is powered off. A separate patch antenna is required, and can connect with the SMA connector on the board.

### Power Input
There are two LT3652 MPPT tracking charge regulators on the board, each with two input connectors in parallel. The intent is for two solar panels on opposite sides of the CubeSat to be connected to the input of the same regulator, since one will not be producing power while the other is. Input voltage can range up to 25V, and the maximum power point can be set using the trim potentiometer on each regulator. I recommend doing this by connecting a power supply to the input connector, setting its voltage to the desired maximum power point voltage (Vmpp) and adjusting the trim pot until the voltage at the Vmpp test point is 2.7V. The regulators each output up to 2A, and cut off charge at 8.27V float and 0.2A termination. 

### 5V Regulator
There is one TPS54394RSAR regulator on board which outputs two isolated 3A buses, which are configured to 5V. One (labelled 5VP in the schematic) feeds the radio and must not be shared with anything else. The other (labelled 5V in the schematic) feeds the RPi and may be shared, although there is no connector native to the board to do this (and it isn't recommended to share it anyways, as the RPi is an important component). Each rail has its own 5V latching current limiter that disconnects its respective voltage bus in case of an short circuit or overcurrent event on the bus. The current limiter resets after approximately 5 seconds (timed by a simple RC circuit) to check if the overcurrent state has been removed. 

### Watchdog Timer
There is a MAX6369 watchdog timer connected to the enable pin of the 5V regulator, set to reset both 5V buses if a watchdog reset signal is not received for 60 seconds. This watchdog timer is enabled by placing (or soldering) a jumper on the WDT_EN jumper, otherwise it will be disconnected from the enable pin and cannot reset power. The watchdog timer can be disabled for testing but should be enabled for flight as it reboots the system in case of a software crash. The flight software must pulse the watchdog reset pin at least once every 60 seconds if the jumper is in place (including within 60 seconds after starting to boot).

### Telemetry
One MCP3208 ADC provides voltage and current readouts, along with a MAX6035 3V reference. Telemetry readouts include Solar Regulator A Output Current (CH0) and Input Voltage (CH1), Solar Regulator B Output Current (CH2) and Input Voltage (CH3), Battery Voltage (CH4), Battery Output Current (CH5, does not read charging current, only discharging current. Includes payload current), Payload Current draw (CH6). The ADC is on CE0 of the Raspberry Pi. Each telemetry readout has a multiplier which can be found in the flight software example or by doing the math yourself :)

### Real Time Clock
A DS3232M RTC provides an uninterrupted time reading with a CR2032 battery backup. This is located on the I2C1 bus and the Raspberry Pi can be configured to use this RTC for its operating system time. 

### Inertial Measurement Unit
An Adafruit BNO055 may be placed on the board. Honestly, this doesn't really have much use though, as calibration is impossible without moving the board around yourself and gets reset with power cycles.

### Inhibits & RBF
Two inhibits and two RBF pins are available. The two inhibits are placed after the charge output but before the battery output bus on the board, so when enabled they block discharging but not charging. One RBF pin is on the high side between the EPS input terminal and everything else on the board, and the other is on the low side. Both will block discharging and charging when enabled. The inhibit connector is an 8 pin header on the bottom of the board labelled on the silkscreen. Connecting the H and L pins enable the inhibit/RBF, blocking current flow. These should be connected to separation switches on the frame that are closed when depressed. Be careful not to connect the H and L pins of different inhibit/RBFs (e.g. connecting pins 2 and 3).

### Battery protections and balancing
A BQ29209 provides resistive charge balancing and an R5460N212AF provides pack level overcurrent, overvoltage, and undervoltage protections. Due to the 20mOhm resistor connected to the R5460N212AF, battery pack output voltage will droop significantly at higher current draws (e.g. more than a few amps). Overcurrent protection uses the Rds(on) of the two MOSFETs along with the shunt resistor to total 26mOhm, so if the MOSFETs need to be substituted, recalculate the effective series resistance again to see if the resulting OCP setpoint is acceptable. If not, adjust the shunt resistor accordingly. 

## Component selection notes
For passives and unspecified components, select automotive grade (AEC-Q100, AEC-Q101, AEC-Q200, AEC-Q300 etc.) if possible. For capacitors especially, derate voltage by about 50% (i.e. for a capacitor on a line experiencing 5V, choose a cap rated for at least 10V). Also recommend sticking to X7R dielectric and resin body or soft body if possible. For 0.1uF caps, 50V for everything is fine. For 10uF, I had to order a bunch of 25V automotive grade caps for everything and a few 50V non automotive caps specifically for the solar input. For current shunts, derate the power rating by a similar amount for thermal concerns.

For the CR2032 battery, I used a Murata CR2032W for its +125C max operating temperature. The battery holder is a LINX-HLD-001-THM. For the battery terminals, use Phoenix Contact 1985807

Shield: [![CC BY-NC-SA 4.0][cc-by-nc-sa-shield]][cc-by-nc-sa]

This work is licensed under a
[Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License][cc-by-nc-sa].

[![CC BY-NC-SA 4.0][cc-by-nc-sa-image]][cc-by-nc-sa]

[cc-by-nc-sa]: http://creativecommons.org/licenses/by-nc-sa/4.0/
[cc-by-nc-sa-image]: https://licensebuttons.net/l/by-nc-sa/4.0/88x31.png
[cc-by-nc-sa-shield]: https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg
