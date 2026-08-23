# JTAG on COM260

<img src="./k3-com260-jtag.png" alt="JTAG on CoM260" width="672" height="504"> 

## Parts:

* [K3 CoM260](https://docs.banana-pi.org/en/BPI-SM10/BananaPi_BPI-SM10)
* [FT2232H Mini Module](https://ftdichip.com/products/ft2232h-mini-module/)
* [SparkFun MicroSD sniffer](https://www.sparkfun.com/sparkfun-microsd-sniffer.html)

## JTAG Pins

To my best knowledge, CoM260 only has the JTAG pins exposed on the module (core) board, code name SM10. There's no JTAG pins
on the base board. From K3 chip pinouts, JTAG can be either connected to JTAG pins in the PMIC pin group, or the MMC1 pins.
The MMC1 pins is perhaps the only viable way if we don't want to hack the board and solder wires. The reason we can use MMC1 pins
for JTAG is because they are multiplexed into multiple functions. (Every pin on K3 is multiplexed up to 8 predefined functions)
Function 5 of MMC1 pins are:
| Pin | Function 5 |
| --- | ---------- |
|MMC1_DAT3|PRI_TDI|
|MMC1_DAT2|PRI_TMS|
|MMC1_DAT1|PRI_TDO|
|MMC1_CLK|PRI_TCK|

***Note that no TRST is exposed through MMC1 pins***

The multiplexing of pins are controlled by pinctrl block at `0xd401e000`. Each pin has a corresponding 32-bit [pinctrl register](https://elixir.bootlin.com/linux/v7.2/source/Documentation/devicetree/bindings/pinctrl/spacemit,k1-pinctrl.yaml), in which there
is a 3-bit field controlling the function (0-7) that the pin connects to. At reset, the default function is 0 for MMC1 pins, so
they need to be switched to Function 5. Vendor u-boot 
