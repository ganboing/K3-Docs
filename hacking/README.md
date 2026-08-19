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
is used for the vendor u-boot SPL blob. The blob will be loaded into `0xc0800000` and the size limit is ***`0x74000`***, most likely
because the masked ROM has to use the `0xc000` at the top for it's own runtime data and stack. Once the blob is loaded and validated,
ROM code transfers the control to the blob and off we go. (There's no sign of calling back to ROM code for runtime services).
In the current vendor firmware, this blob is exactly the u-boot SPL, which is a heavily modified (and surely messy) version that 
contains DDR init code. After DRAM initialized, SPL continues to load other blobs from EMMC/SPI/UFS partitions.

Vendor U-boot is at https://github.com/spacemit-com/uboot-2022.10/tree/k3-br-v1.0.y

As firmware/bootloader hackers, we are interested in a workflow that can easily boot a given firmware blob without re-flashing.
To achieve this, we leave the boot mode to `USB Download`. (You may also use the UART download mode if your particular board doesn't
have debug Type-C, but I imagine it'll be quite slow. Ref: https://community.milkv.io/t/jupiter-2-bootrom-and-booting-over-uart/3978)
In this mode, the masked ROM will start a fastboot server on the debug type-C port and waiting for accepting for downloading the
same binary format mentioned earlier. This is ideal for debugging and CI automation.

In the current vendor U-Boot, once the SPL is downloaded and starts running, it'll reset the debug Type-C port and start a fastboot
server again on the same Type-C port. It's intended for flashing and recover a bricked device, but we can make use of this feature to
conveniently upload the rest blobs, including OpenSBI, u-boot proper, and optionally the `esos` binaries for the RT cores for power management.
There're several things we need to hack on top of the vendor u-boot:
 * We need to package all binary blobs into the u-boot.itb and fastboot on that, as we are not loading them from EMMC/SPI/UFS.

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
