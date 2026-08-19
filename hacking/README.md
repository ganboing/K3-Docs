## K3 bootflow

Based on the [schematics of K3 Pico-ITX](https://cdn-resource.spacemit.com/file/product/K3/k3_pico_hw/SCH-146-V10_K3_DEB1_P1_LP5315B_PDF.pdf),
K3 supports the following boot modes:

| GPIO_69 | GPIO_68 | GPIO_66 | GPIO_65 | Function |
| ------- | ------- | ------- | ------- | -------- |
|    0    |    X    |    0    |    0    | Boot from EMMC |
|    0    |    X    |    0    |    1    | Boot from SPI NOR |
|    0    |    X    |    1    |    0    | Boot from SPI NAND |
|    0    |    X    |    1    |    1    | Boot from UFS |
|    1    |    0    |    X    |    X    | USB Download (fastboot) |
|    1    |    1    |    X    |    X    | UART Download (xmodem) |

Upon power-on, only hart 0 starts running the masked ROM, and others are halted. The ROM code is responsible for detecting boot modes, 
and loads (and verify) a binary blob from the boot source. The binary blob must be properly signed/encapsulated for the masked ROM to 
validate and load it into the SRAM (DRAM is not initialized at this time). [This tool](https://github.com/spacemit-com/uboot-2022.10/blob/k3-br-v1.0.y/tools/build_binary_file.py) 
is responsible for creating the proper headers and footers. [This configuration](https://github.com/spacemit-com/uboot-2022.10/blob/k3-br-v1.0.y/board/spacemit/k3/configs/fsbl.json)
is used for the vendor u-boot SPL blob. Once the blob is loaded and validated, ROM transfer the control to the blob and off we go.
(There's no sign of calling back to ROM code for runtime services). In the current vendor firmware, this blob is exactly the u-boot SPL,
which is a heavily modified (and surely messy) version that contains DDR init code. After DRAM initialized, SPL continues to load other
blobs from EMMC/SPI/UFS partitions.

For firmware/bootloader hacker, we are interested in a workflow that can easily boot a given firmware blob without re-flashing.
To achieve this, we leave the boot mode to `USB Download`. In this mode, the masked ROM will start a fastboot server on the debug type-C
port and waiting for accepting for downloading the same binary format mentioned earlier. The maximum download size is ***`0x74000`***.
Once the SPL is downloaded and starts running, it'll reset the debug Type-C port and start a fastboot server again on the same Type-C port.
The fastboot implementation is from u-boot. This allows us to continue uploading the rest blobs, including OpenSBI, u-boot proper, and other
things like the `esos` binaries.

```
sys: 0x10001200
bm:2
usb_init : enter 3296,3072
usb_core_init : enter 
DWC3_GRXTHRCFG:0x4400000
ROM: usb download handler
 rst
done H
setup= 0x1000680 0x400000,
 rst
done H
setup= 0x60500 0x0,
setup= 0x1000680 0x120000,
setup= 0x2000680 0x90000,
setup= 0x2000680 0x200000,
setup= 0x3000680 0xff0000,
setup= 0x3020680 0xff0409,
setup= 0x3010680 0xff0409,
setup= 0x10900 0x0,
usb_rx_bytes : start len[4096]
setup= 0x3020680 0xff0409,
setup= 0x3040680 0xff0409,
```
