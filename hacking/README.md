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
   * Thus, we must package both OpenSBI and "esos" into u-boot.itb for fastboot to bring up a comparable environment as regular boot.
 * The code quality of vendor u-boot is **bad**. It triggered asserts during my testing. Perhaps that's why spacemit's now shipping edk2 instead?

---

## Building u-boot/OpenSBI

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
This hacked version of u-boot has OpenSBI packaged into `u-boot.itb`

### Combing with esos
```
# Download the latest esos.itb
curl -fL https://archive.spacemit.com/bianbu4/pool/main/e/esos/bianbu-esos_1.0.5_riscv64.deb | dpkg-deb -X - /tmp/esos
cd <uboot-k3>
./scripts/combine-esos.sh u-boot.itb /tmp/esos/usr/lib/riscv64-linux-gnu/esos/esos.itb > u-boot-esos.itb
```
Now `u-boot-esos.itb` has everything.

## Let's run it
Put the device into recovery mode, and poweron. You should see the following print out on debug console
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

The Host machine should discover a fastboot USB device:
```
kernel: usb 3-1.4: new high-speed USB device number 69 using xhci_hcd
kernel: usb 3-1.4: New USB device found, idVendor=361c, idProduct=1001, bcdDevice= 0.01
kernel: usb 3-1.4: New USB device strings: Mfr=1, Product=2, SerialNumber=0
kernel: usb 3-1.4: Product: USB download gadget
kernel: usb 3-1.4: Manufacturer: DFU
```
At this point, we are ready to fastboot the `FSBL.bin`:
```shell
cd <uboot-k3>
fastboot stage FSBL.bin
fastboot continue
```
There should be print outs on the debug console similar to:
```
setup= 0x80 0x20000,
fastboot_handle_command: max-download-size
usb_tx_bytes : start len[14]
usb_rx_bytes : start len[4096]
fastboot_handle_command: 00074000
Starting download of 475136 bytes
usb_tx_bytes : start len[12]
usb_rx_bytes : start len[475136]
usb_tx_bytes : start len[4]
usb_rx_bytes : start len[4096]
fastboot_handle_command: continue
usb_tx_bytes : start len[4]
j...

U-Boot SPL 2022.10 (Jul 24 2026 - 07:12:47 +0000)
DDR Part Number: MT62F1G32D2DS, Size: 4096MB, Data Rate: 6400MT/s
DDR training consume 2992ms
```
Now u-boot SPL has done DDR training, and has started its own fastboot server, ready to accept the next stage. On host:
```shell
cd <uboot-k3>
fastboot oem speed:super-speed
fastboot stage u-boot-esos.itb
fastboot continue
```

On the debug console:
```
_rproc_ops_wrapper: Starting rcpu0...
_rproc_ops_wrapper: Starting rcpu1...
Invalid OS Type: esos
No reserved memory region found in source FDT
unknown boot device: 13
load failed: uboot=-1 esos=-1
Invalid OS Type: esos
Invalid OS Type: esos
Invalid OS Type: esos
Invalid OS Type: esos
Invalid OS Type: esos
Can't get 'entry' property from FIT 0x10217c2a0, node: offset 928, name uboot (FDT_ERR_NOTFOUND)
WARNING: riscv,rpmi-hsm driver is experimental and may change
WARNING: riscv,rpmi-shmem-mbox driver is experimental and may change
WARNING: riscv,rpmi-system-reset driver is experimental and may change
WARNING: riscv,rpmi-system-suspend driver is experimental and may change

OpenSBI k3-br-v1.0.5-1-gf3594922
Build time: 2026-08-19 00:35:47 -0700
Build compiler: gcc version 15.1.0 ()
   ____                    _____ ____ _____
  / __ \                  / ____|  _ \_   _|
 | |  | |_ __   ___ _ __ | (___ | |_) || |
 | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
 | |__| | |_) |  __/ | | |____) | |_) || |_
  \____/| .__/ \___|_| |_|_____/|____/_____|
        | |
        |_|

WARNING: riscv,rpmi-mpxy-clock driver is experimental and may change
WARNING: riscv,rpmi-mpxy-voltage driver is experimental and may change
WARNING: riscv,rpmi-mpxy-domain driver is experimental and may change
WARNING: riscv,rpmi-mpxy-rtc driver is experimental and may change
WARNING: riscv,rpmi-mpxy-pwrkey driver is experimental and may change
Platform Name               : spacemit k3 com260 ifx board
Platform Features           : medeleg
Platform HART Count         : 16
Platform IPI Device         : aia-imsic
Platform Timer Device       : aclint-mtimer @ 24000000Hz
Platform Console Device     : uart8250
Platform HSM Device         : rpmi-hsm
Platform PMU Device         : ---
Platform Reboot Device      : rpmi-system-reset
Platform Shutdown Device    : rpmi-system-reset
Platform Suspend Device     : rpmi-system-suspend
Platform CPPC Device        : ---
Firmware Base               : 0x100000000
Firmware Size               : 638 KB
Firmware RW Offset          : 0x40000
Firmware RW Size            : 382 KB
Firmware Heap Offset        : 0x85000
Firmware Heap Size          : 106 KB (total), 6 KB (reserved), 18 KB (used), 81 KB (free)
Firmware Scratch Size       : 8192 B (total), 5816 B (used), 2376 B (free)
Runtime SBI Version         : 2.0
Standard SBI Extensions     : time,rfnc,ipi,base,hsm,srst,susp,pmu,dbcn,legacy
Experimental SBI Extensions : fwft,dbtr,sse,mpxy

Domain0 Name                : root
Domain0 Boot HART           : 0
Domain0 HARTs               : 0*,1*,2*,3*,4*,5*,6*,7*,8*,9*,10*,11*,12*,13*,14*,15*
Domain0 Region00            : 0x00000000cac90c00-0x00000000cac90cff M: (I,R,W) S/U: ()
Domain0 Region01            : 0x00000000d4017000-0x00000000d4017fff M: (I,R,W) S/U: (R,W)
Domain0 Region02            : 0x00000000d4282000-0x00000000d4282fff M: (R,W,X) S/U: ()
Domain0 Region03            : 0x00000000f1800000-0x00000000f1803fff M: (I,R,W) S/U: ()
Domain0 Region04            : 0x0000000100e00000-0x0000000100e03fff M: (I,R,W) S/U: ()
Domain0 Region05            : 0x00000000f1000000-0x00000000f100ffff M: (I,R,W) S/U: ()
Domain0 Region06            : 0x00000000f1810000-0x00000000f181ffff M: (I,R,W) S/U: ()
Domain0 Region07            : 0x0000000100000000-0x000000010003ffff M: (R,X) S/U: ()
Domain0 Region08            : 0x0000000100000000-0x00000001000fffff M: (R,W) S/U: ()
Domain0 Region09            : 0x0000000100d00000-0x0000000100dfffff M: () S/U: ()
Domain0 Region10            : 0x0000000100200000-0x00000001003fffff M: () S/U: ()
Domain0 Region11            : 0x0000000100400000-0x00000001005fffff M: () S/U: ()
Domain0 Region12            : 0x0000000100800000-0x0000000100bfffff M: () S/U: ()
Domain0 Region13            : 0x0000000000000000-0xffffffffffffffff M: () S/U: (R,W,X)
Domain0 Next Address        : 0x0000000102000000
Domain0 Next Arg1           : 0x000000010217c2a0
Domain0 Next Mode           : S-mode
Domain0 SysReset            : yes
Domain0 SysSuspend          : yes

Boot HART ID                : 0
Boot HART Domain            : root
Boot HART Priv Version      : v1.12
Boot HART Base ISA          : rv64imafdcbvhx
Boot HART ISA Extensions    : smaia,smstateen,sscofpmf,sstc,zicntr,zihpm,smcntrpmf,zicboz,zicbom,svpbmt,sdtrig
Boot HART PMP Count         : 16
Boot HART PMP Granularity   : 12 bits
Boot HART PMP Address Bits  : 38
Boot HART MHPM Info         : 16 (0x0007fff8)
Boot HART Debug Triggers    : 4 triggers
Boot HART MIDELEG           : 0x0000000000003666
Boot HART MEDELEG           : 0x0000000000f0b509
[  12.523] initcall: 00000001020d0f54
[  12.523] 

U-Boot 2022.10-dirty (Aug 19 2026 - 00:42:43 -0700)
...
```
