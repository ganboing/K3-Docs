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
and loads (and verify) a binary blob from the boot source, then hands off to execute that blob. The binary blob must be properly
signed/encapsulated for the masked ROM to validate and load it into the SRAM (DRAM is not initialized at this time). It is often
referred to as `FSBL.bin` in the vendor sources. Once `FSBL.bin` starts running, the ROM is completely off the hook.
Details on how ROM finds `FSBL.bin`:
 * SPI flash: A `bootinfo` structure at the beginning describing `FSBL.bin` offset.
 * EMMC/UFS: GPT partition table and (or) `bootinfo` structure describing `FSBL.bin` offset.
 * USB Download (fastboot): Can just stage `FSBL.bin` as-is.
 * UART Download (xmodem): Use xmodem protocol to send `FSBL.bin` (not yet verified). [More info](https://community.milkv.io/t/jupiter-2-bootrom-and-booting-over-uart/3978)

---

## FSBL.bin
`FSBL.bin` starts running in SRAM (`0xc0800000`). Single core, M-mode, DDR uninitialized. In the current vendor implementation,
`FSBL.bin` is really just u-boot SPL, with lots of messy vendor patches: [source code](https://github.com/spacemit-com/uboot-2022.10/blob/k3-br-v1.0.y/board/spacemit/k3/spl.c)

Its jobs are:
 * Clock/pmic/pinctrl/... basic platform init
 * Gather board types, product name, DDR die info/types(ddr4/5), from EEPROM.
 * DDR Training.
 * If Boot modes: loads following into DRAM, and boot OpenSBI:
   * OpenSBI
   * bootloader (uboot or edk2)
   * optionally "esos" for 2 RT24 RV32 cores (power management)
 * If Download modes: start a fastboot server on the same type-C debug port
   * waits forever until a `fastboot continue` is received.
   * Unpack the received fit image and boot accordingly.

## OpenSBI
OpenSBI starts running in DRAM (`0x100000000`). Single core, M-mode.
 * Initialize cache coherency manager
 * Kick start all other harts (hartid != 0)
 * Handoff to u-boot proper or edk2, depending on what's being loaded by u-boot SPL.

## U-boot proper / edk2
Normal/regular bootflow from here on.

---

## Lesson learned so far
 * From u-boot-spl.bin or any binary to `FSBL.bin`, you need [This tool](https://github.com/spacemit-com/uboot-2022.10/blob/k3-br-v1.0.y/tools/build_binary_file.py).
Sample config file for u-boot-spl: [fsbl.json](https://github.com/spacemit-com/uboot-2022.10/blob/k3-br-v1.0.y/board/spacemit/k3/configs/fsbl.json).
 * `FSBL.bin` limit for `fastboot` (or perhaps all other boot modes) is ***`0x74000`***, most likely because the masked ROM
has to use the `0xc000` at the top for it's own runtime data and stack.
 * The `u-boot.itb` can be used directly for fastboot'ing the next stage after `FSBL.bin`, but it's really hacky:
   * No OpenSBI, forcing the u-boot proper to be running in M-mode. Impossible to even boot Linux.
   * No "esos", disabling power-management functions and may impact features in Linux.
 * The code quality of vendor u-boot is **bad**. It triggered asserts during my testing. Perhaps that's why spacemit's now shipping edk2 instead?

---

## Building u-boot/OpenSBI/...

### OpenSBI
```shell
git clone -b k3-br-v1.0.y https://github.com/spacemit-com/opensbi.git opensbi-k3
cd opensbi-k3
make PLATFORM=generic PLATFORM_DEFCONFIG=k3_defconfig
```

### u-boot
```shell
git clone https://github.com/ganboing/uboot-k3 uboot-k3
cd uboot-k3
ln -snr <opensbi-k3>/platform/generic/firmware/fw_dynamic.bin board/spacemit/k3/
make k3_defconfig all
```

## Combine with esos
```
# Download the latest esos.itb
curl -fL https://archive.spacemit.com/bianbu4/pool/main/e/esos/bianbu-esos_1.0.5_riscv64.deb | dpkg-deb -X - /tmp/esos
cd <uboot-k3>

```

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
