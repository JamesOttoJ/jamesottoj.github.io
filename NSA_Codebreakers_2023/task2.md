# TASK 2
- [Finding the Datasheet](#finding-the-datasheet)
- [Using the Pins](#using-the-pins)

### Challenge Description
> Thanks to your efforts the USCG discovered the unknown object by trilaterating the geo and timestamp entries of their record with the correlating entries you provided from the NSA databases. Upon discovery, the device appears to be a device with some kind of collection array used for transmitting and receiving. Further visual inspection shows the brains of this device to be reminiscent of a popular hobbyist computer. Common data and visual ports non-responsive; the only exception is a boot prompt output when connecting over HDMI. Most interestingly there is a 40pin GPIO header with an additional 20pin header. Many of these physical pins show low-voltage activity which indicate data may be enabled. There may be a way to still interact with the device firmware...
>
> Find the correct processor datasheet, and then use it and the resources provided to enter which physical pins enable data to and from this device
> 
> Hints:
> 
> - Note: For the pinout.svg, turn off your application's dark mode if you're unable to see the physical pin labels (eg: 'P1', 'P60')
> - The pinout.svg has two voltage types. The gold/tan is 3.3v, the red is 5v.
> - The only additional resource you will need is the datasheet, or at least the relevant information from it

### Files Given
pinout.svg
cpu.jpg
boot_prompt.log

### Finding the Datasheet
To get a datasheet, the most important thing to know is what device you need the datasheet for. To get this, we can look at the [cpu.jpg](./static/cpu.jpg) file to find what model it is. Looking at it, we see that it's from broadcom, and the model number is BCM2837. If you look up the full "BCM2837RIFBG" it will just show the BCM2837 data. From this we can find the [datasheet](https://usermanual.wiki/Datasheet/BCM2837ARMPeripheralsBroadcom.1054296467) and see that this CPU is in the raspberry pi 3 model b.

### Using the Pins
Looking at the [Raspberry pi GPIO documentation](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html) the voltage section says that both 5v and 3.3v are options but that output pins can be set to 3.3 or 0 volts. This means we just need to pick any 3.3v and any ground pin to complete the circuit.

For data, we need a Tx(data transmit) and Rx(data recieve) signal. Looking at the GPIO pinout from the datasheet, we can see that there are 5 different alternative modes for the GPIO pins. To find which mode to use, we can turn to the boot_prompt which specifies ALT 5. From there, there are a couple options to choose from for a pair of Tx/Rx: 14/15, 32/33, 40/41. Looking at what GPIO pins we have available, we just need to select one of the pairs and count which physical pin it is using the nubers on the outside

For my [pinout](./static/pinout.svg), the answer was:
3.3v: P10
GND: P9
Tx: P37
Rx: P38