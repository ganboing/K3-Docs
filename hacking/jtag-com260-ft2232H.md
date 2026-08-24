# JTAG on COM260

<img src="./k3-com260-jtag.png" alt="JTAG on CoM260" width="672" height="504"> 

## Parts:

* [K3 CoM260](https://docs.banana-pi.org/en/BPI-SM10/BananaPi_BPI-SM10)
* [FT2232H Mini Module](https://ftdichip.com/products/ft2232h-mini-module/)
* [SparkFun MicroSD sniffer](https://www.sparkfun.com/sparkfun-microsd-sniffer.html)

The pins on MicroSD breakout board are actually pressed-fit. I've got near-zero experience with soldering, and without tools, so
I bought some [press-fit pins](https://www.mouser.com/en/ProductDetail/649-93689-103-02LF), compressed it a bit with pliers, and
just pressed it in.

## JTAG Pins

K3 exposes JTAG pins through both the dedicated JTAG pins and MMC1 pins, but AFAIK, the CoM260 core board SM10 only has MMC1 pins
connected, and only through the MMC1 sdcard slot. For Pico-ITX, it does have the JTAG pins exposed, but it's hidden under the fan,
and you need to [solder the wires](https://forum.spacemit.com/t/topic/1345/2?u=1783079046) (or perhaps use Pogo pins?).
To me, the CoM260 is workable, and I don't want to risk destroying my board.

By default MMC1 pins are multiplexed to MMC1 functions (function 0) at poweron. To use JTAG, they must be switched to function 5:

<img src="./pinmux_MMC1.png" width="986" height="138">

| Pin | Function 5 |
| --- | ---------- |
|MMC1_DAT3|PRI_TDI|
|MMC1_DAT2|PRI_TMS|
|MMC1_DAT1|PRI_TDO|
|MMC1_CLK|PRI_TCK|

***Note that no TRST is exposed through MMC1 pins***

To perform the switching, write to the corresponding pinctrl register of the pins. The pinctrl block is at `0xd401e000`.
Each pin has a corresponding 32-bit [pinctrl register](https://elixir.bootlin.com/linux/v7.2/source/Documentation/devicetree/bindings/pinctrl/spacemit,k1-pinctrl.yaml), in which there
is a 3-bit field controlling the function (0-7) that the pin connects to. 
Vendor u-boot has a convenient device-tree property [`spacemit,enable-debug-jtag`](https://github.com/spacemit-com/uboot-2022.10/blob/769ed686043d06b65c2863af479e97985e3b0d64/arch/riscv/dts/k3_spl.dts#L28) that
switches MMC1 functions in SPL. Un-comment that line, and don't forget to disable mmc1(sdhci1) in u-boot proper and Linux device-tree
so the pinmux won't get switched back.

## JTAG Connection

I'm using a FT2232H Mini Module for JTAG. It's cheap, and OpenOCD has decent support. Connect the pins as shown:
```
TXD: BDBUS0 -> UART0_RXD_LS
RXD: BDBUS1 <- UART0_TXD_LS
TCK: ADBUS0 -> MMC1_CLK
TDI: ADBUS1 -> MMC1_DAT3/CD
TDO: ADBUS2 <- MMC1_DAT1
TMS: ADBUS3 -> MMC1_DAT2
VIO: VCCIO  <- VDD_3V3_SYS (PIN 17 of 40-PIN)

!!!Use VCCIO from the COM260 Board's VDD_3V3_SYS board to avoid power backfeeding!!!
```

This setup conveniently gives us both the debug UART0 and JTAG through the same USB converter. Note that the VCCIO of
FT2232H needs to be feed from COM260's 3V3 to avoid FT2232H backfeeding power back when COM260 is powered-off. Otherwise,
you'll get some random garbage characters in the UART console when COM260 is turned off. The backfeeding can also confuse
the PMIC or something on the board, that K3 has some possibility to not turning on when 12V is re-connected.

This however, creates another challenge. When the FT2232H is first connected without COM260 power-on, there's no VCCIO, and for
some reason the FT2232H mistakenly reports itself as a Quad RS232-HS:
```
usb 3-1.2: new high-speed USB device number 9 using xhci_hcd
usb 3-1.2: config 1 interface 1 altsetting 0 bulk endpoint 0x83 has invalid maxpacket 64
usb 3-1.2: config 1 interface 1 altsetting 0 bulk endpoint 0x4 has invalid maxpacket 64
usb 3-1.2: New USB device found, idVendor=0403, idProduct=6011, bcdDevice= 8.00
usb 3-1.2: New USB device strings: Mfr=1, Product=2, SerialNumber=0
usb 3-1.2: Product: Quad RS232-HS
usb 3-1.2: Manufacturer: FTDI
```

To workaround this, power-on COM260, and do a `usbreset` of this "Quad RS232-HS" device. It'll then behave properly and enumerated as FT2232H:
```
usb 3-1.1: new high-speed USB device number 10 using xhci_hcd
usb 3-1.1: New USB device found, idVendor=0403, idProduct=6010, bcdDevice= 7.00
usb 3-1.1: New USB device strings: Mfr=1, Product=2, SerialNumber=3
usb 3-1.1: Product: FT2232H MiniModule
usb 3-1.1: Manufacturer: FTDI
usb 3-1.1: SerialNumber: FT80197L
```

Once the FT2232H is properly probed, COM260 can be turned off/on at will, and not causing issues with FT2232H.

## OpenOCD adapter config
```
# JTAG adapter setup
adapter driver ftdi

ftdi vid_pid 0x0403 0x6010
ftdi channel 0
ftdi layout_init 0x0008 0x000b
reset_config none
```

Nothing special. Note the `reset_config none` is deliberate, as we have neither sRST or tRST.
