## K3 OpenOCD/gdb Guide

[SpacemiT official Document](https://github.com/spacemit-com/docs-tool/blob/main/en/user_guide/jtag_debug_user_guide.md)

At the time of writing, I'm using the vendor's [modified OpenOCD](https://spacemit.com/community/resources-download/Tools/JTAG%20debugging%20tool)

### K3 Debug Topology

K3 has 4 CPU clusters, 2 X100 + 2 A100. Each cluster has its own debug module (DM), and there's a global DTM that connects to
these 4 DMs. The latest stable OpenOCD relase (0.12) doesn't support configuring per-hart DM base addresses. The support was merged in 
[this commit](https://github.com/openocd-org/openocd/commit/5754aebc49450cc0da5c8a90ebd059160d21f256)
You'll need to at least build upstream OpenOCD from master to get K3 support. Or just use the vendor's version, as there're still
[some patches](https://review.openocd.org/q/owner:zhen.liang%2540spacemit.com) pending review. I haven't checked if they are k3-related.

### K3 JTAG Tips

From vendor's OpenOCD (v1.1.2), take `k3.sh` for example:

```
spacemit-openocd-linux-V1.1.2/bin$ cat k3.sh 
#!/bin/bash
# SPDX-License-Identifier: GPL-2.0-or-later

SCRIPT_DIR="../share/openocd/scripts"

# ADAPTER_DRIVER can set as jlink cmsis-dap ...
ADAPTER_DRIVER=jlink

./openocd -c "bindto 0.0.0.0" -c "gdb port 1024" -c "telnet port 4444" \
          -c "set SPEED 8000" \
          -c "set SECJTAG 0" \
          -c "set TARGET acpu" \
          -c "set CLUSTERS {0 1 2 3}" \
          -c "set CLUSTER0_COREIDS {0 1 2 3}" \
          -c "set CLUSTER1_COREIDS {0 1 2 3}" \
          -c "set CLUSTER2_COREIDS {0 1 2 3}" \
          -c "set CLUSTER3_COREIDS {0 1 2 3}" \
          -c "set RVTRACE 0" \
          -c "set CSTRACE 0" \
          -c "set ENCODERS {0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15}" \
          -c "set ATBBRIDGES {0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15}" \
          -c "set FUNNELS {0 1 2 3}" \
          -f $SCRIPT_DIR/interface/$ADAPTER_DRIVER.cfg \
          -f $SCRIPT_DIR/spacemit_helper.tcl \
          -f scripts/spacemit-k3.cfg
```

There are several things going on here:

 * `bindto`, `gdb port`, `telnet port`: I actually deleted these options, and let OpenOCD use the default ports, 3333 for gdb, 4444 for telnet
 * `TARGET`: Either `acpu` (the 16 cores main CPU) or `rcpu` (the 2 cores RCPU). The SoC has both CPU/RCPU JTAG connected to the same TAP, and another TAP (irlen 9) implements register 0x98 controlling the switch [ref](https://www.spacemit.com/community/document/info?lang=en&nodepath=hardware/key_stone/k3/k3_docs/k3_usermanual/16_peripherals/jtag.md). `spacemit-k3.cfg` is responsible for setting the correct value to register 0x98 of this TAP, and then connects to the CPU/RCPU TAP.
 * `ADAPTER_DRIVER=jlink`: Change it to your adapter config, such as the [FTDI config](https://github.com/ganboing/K3-Docs/blob/master/hacking/jtag-com260-ft2232H.md#openocd-adapter-config) for my CoM260 setup, and copy the file into  ../share/openocd/scripts/interface.
 * `SPEED 8000`: The adapter speed that gets set to via `adapter speed <SPEED>`. No need to change unless the session is too slow for you
 * `CLUSTERS`, `CLUSTERx_COREIDS` are the cores OpenOCD connects to. ***Don't specify the ones that's power-gated, otherwise it'll hang the OpenOCD session***. Use the corresponding scripts, which conveniently defined these values for you:
   * `k3-1x1.sh`: Debug only core 0. ***Use this during early boot***
   * `k3-x100.sh`: Debug only X100 cores, cluster 0/1
   * `k3-a100.sh`: Debug only A100 cores, cluster 2/3
   * `k3.sh`: Debug everything
 * `RVTRACE`, `CSTRACE`, `ENCODERS`, `ATBBRIDGES`, `FUNNELS`: Trace components. Will be covered later.

Tailor the `k3-xyz.sh` scripts to your needs, and a successful launch of OpenOCD looks like this:
```
spacemit-openocd-linux-V1.1.2/bin$ ./k3.sh 
Open On-Chip Debugger 0.12.0+dev-g57f52fa (2026-06-10-07:54)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
8000
0
acpu
0 1 2 3
0 1 2 3
0 1 2 3
0 1 2 3
0 1 2 3
0
0
0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15
0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15
0 1 2 3
none separate
sbawrite32
Info : [k3.x100.0] Hardware thread awareness created
Info : [k3.x100.4] Hardware thread awareness created
Info : [k3.a100.8] Hardware thread awareness created
Info : [k3.a100.12] Hardware thread awareness created
Info : clock speed 2000 kHz
Info : JTAG tap: post.unknown tap/device found: 0x08502c0d (mfg: 0x606 (Shenzhen Chixingzhe Technology Co Ltd), part: 0x8502, ver: 0x0)
Info : JTAG tap: pre.unknown enabled
Info : JTAG tap: k3.cpu enabled
Info : [k3.x100.0] datacount=2 progbufsize=2
Info : [k3.x100.0] Vector support with vlenb=32
Info : [k3.x100.0] S?aia detected with IMSIC
Info : [k3.x100.0] Core 0 made part of halt group 1.
Info : [k3.x100.0] Examined RISC-V core
Info : [k3.x100.0]  XLEN=64, misa=0x8000000000b411af
Info : [k3.x100.0] Examination succeed
Info : [k3.x100.1] datacount=2 progbufsize=2
Info : [k3.x100.1] Vector support with vlenb=32
Info : [k3.x100.1] S?aia detected with IMSIC
Info : [k3.x100.1] Core 1 made part of halt group 1.
Info : [k3.x100.1] Examined RISC-V core
Info : [k3.x100.1]  XLEN=64, misa=0x8000000000b411af
Info : [k3.x100.1] Examination succeed
Info : [k3.x100.2] datacount=2 progbufsize=2
Info : [k3.x100.2] Vector support with vlenb=32
Info : [k3.x100.2] S?aia detected with IMSIC
Info : [k3.x100.2] Core 2 made part of halt group 1.
Info : [k3.x100.2] Examined RISC-V core
Info : [k3.x100.2]  XLEN=64, misa=0x8000000000b411af
Info : [k3.x100.2] Examination succeed
Info : [k3.x100.3] datacount=2 progbufsize=2
Info : [k3.x100.3] Vector support with vlenb=32
Info : [k3.x100.3] S?aia detected with IMSIC
Info : [k3.x100.3] Core 3 made part of halt group 1.
Info : [k3.x100.3] Examined RISC-V core
Info : [k3.x100.3]  XLEN=64, misa=0x8000000000b411af
Info : [k3.x100.3] Examination succeed
Info : [k3.x100.4] datacount=2 progbufsize=2
Info : [k3.x100.4] Vector support with vlenb=32
Info : [k3.x100.4] S?aia detected with IMSIC
Info : [k3.x100.4] Core 0 made part of halt group 1.
Info : [k3.x100.4] Examined RISC-V core
Info : [k3.x100.4]  XLEN=64, misa=0x8000000000b411af
Info : [k3.x100.4] Examination succeed
Info : [k3.x100.5] datacount=2 progbufsize=2
Info : [k3.x100.5] Vector support with vlenb=32
Info : [k3.x100.5] S?aia detected with IMSIC
Info : [k3.x100.5] Core 1 made part of halt group 1.
Info : [k3.x100.5] Examined RISC-V core
Info : [k3.x100.5]  XLEN=64, misa=0x8000000000b411af
Info : [k3.x100.5] Examination succeed
Info : [k3.x100.6] datacount=2 progbufsize=2
Info : [k3.x100.6] Vector support with vlenb=32
Info : [k3.x100.6] S?aia detected with IMSIC
Info : [k3.x100.6] Core 2 made part of halt group 1.
Info : [k3.x100.6] Examined RISC-V core
Info : [k3.x100.6]  XLEN=64, misa=0x8000000000b411af
Info : [k3.x100.6] Examination succeed
Info : [k3.x100.7] datacount=2 progbufsize=2
Info : [k3.x100.7] Vector support with vlenb=32
Info : [k3.x100.7] S?aia detected with IMSIC
Info : [k3.x100.7] Core 3 made part of halt group 1.
Info : [k3.x100.7] Examined RISC-V core
Info : [k3.x100.7]  XLEN=64, misa=0x8000000000b411af
Info : [k3.x100.7] Examination succeed
Info : [k3.a100.8] datacount=2 progbufsize=2
Info : [k3.a100.8] Vector support with vlenb=128
Info : [k3.a100.8] S?aia detected with IMSIC
Info : [k3.a100.8] Core 0 made part of halt group 1.
Info : [k3.a100.8] Examined RISC-V core
Info : [k3.a100.8]  XLEN=64, misa=0x8000000000b4112f
Info : [k3.a100.8] Examination succeed
Info : [k3.a100.9] datacount=2 progbufsize=2
Info : [k3.a100.9] Vector support with vlenb=128
Info : [k3.a100.9] S?aia detected with IMSIC
Info : [k3.a100.9] Core 1 made part of halt group 1.
Info : [k3.a100.9] Examined RISC-V core
Info : [k3.a100.9]  XLEN=64, misa=0x8000000000b4112f
Info : [k3.a100.9] Examination succeed
Info : [k3.a100.10] datacount=2 progbufsize=2
Info : [k3.a100.10] Vector support with vlenb=128
Info : [k3.a100.10] S?aia detected with IMSIC
Info : [k3.a100.10] Core 2 made part of halt group 1.
Info : [k3.a100.10] Examined RISC-V core
Info : [k3.a100.10]  XLEN=64, misa=0x8000000000b4112f
Info : [k3.a100.10] Examination succeed
Info : [k3.a100.11] datacount=2 progbufsize=2
Info : [k3.a100.11] Vector support with vlenb=128
Info : [k3.a100.11] S?aia detected with IMSIC
Info : [k3.a100.11] Core 3 made part of halt group 1.
Info : [k3.a100.11] Examined RISC-V core
Info : [k3.a100.11]  XLEN=64, misa=0x8000000000b4112f
Info : [k3.a100.11] Examination succeed
Info : [k3.a100.12] datacount=2 progbufsize=2
Info : [k3.a100.12] Vector support with vlenb=128
Info : [k3.a100.12] S?aia detected with IMSIC
Info : [k3.a100.12] Core 0 made part of halt group 1.
Info : [k3.a100.12] Examined RISC-V core
Info : [k3.a100.12]  XLEN=64, misa=0x8000000000b4112f
Info : [k3.a100.12] Examination succeed
Info : [k3.a100.13] datacount=2 progbufsize=2
Info : [k3.a100.13] Vector support with vlenb=128
Info : [k3.a100.13] S?aia detected with IMSIC
Info : [k3.a100.13] Core 1 made part of halt group 1.
Info : [k3.a100.13] Examined RISC-V core
Info : [k3.a100.13]  XLEN=64, misa=0x8000000000b4112f
Info : [k3.a100.13] Examination succeed
Info : [k3.a100.14] datacount=2 progbufsize=2
Info : [k3.a100.14] Vector support with vlenb=128
Info : [k3.a100.14] S?aia detected with IMSIC
Info : [k3.a100.14] Core 2 made part of halt group 1.
Info : [k3.a100.14] Examined RISC-V core
Info : [k3.a100.14]  XLEN=64, misa=0x8000000000b4112f
Info : [k3.a100.14] Examination succeed
Info : [k3.a100.15] datacount=2 progbufsize=2
Info : [k3.a100.15] Vector support with vlenb=128
Info : [k3.a100.15] S?aia detected with IMSIC
Info : [k3.a100.15] Core 3 made part of halt group 1.
Info : [k3.a100.15] Examined RISC-V core
Info : [k3.a100.15]  XLEN=64, misa=0x8000000000b4112f
adapter speed: 8000 kHz
    TargetName         Type       Endian TapName            State       
--  ------------------ ---------- ------ ------------------ ------------
 0  k3.x100.0          riscv      little k3.cpu             running
 1  k3.x100.1          riscv      little k3.cpu             running
 2  k3.x100.2          riscv      little k3.cpu             running
 3  k3.x100.3          riscv      little k3.cpu             running
 4  k3.x100.4          riscv      little k3.cpu             running
 5  k3.x100.5          riscv      little k3.cpu             running
 6  k3.x100.6          riscv      little k3.cpu             running
 7  k3.x100.7          riscv      little k3.cpu             running
 8  k3.a100.8          riscv      little k3.cpu             running
 9  k3.a100.9          riscv      little k3.cpu             running
10  k3.a100.10         riscv      little k3.cpu             running
11  k3.a100.11         riscv      little k3.cpu             running
12  k3.a100.12         riscv      little k3.cpu             running
13  k3.a100.13         riscv      little k3.cpu             running
14  k3.a100.14         riscv      little k3.cpu             running
15* k3.a100.15         riscv      little k3.cpu             running
Info : [k3.a100.15] Examination succeed
Info : [k3.x100.0] starting gdb server on 3333
Info : Listening on port 3333 for gdb connections
Info : [k3.x100.0] External trigger 0 made part of halt group 0.
Info : [k3.x100.4] External trigger 0 made part of halt group 0.
Info : [k3.a100.8] External trigger 0 made part of halt group 0.
Info : [k3.a100.12] External trigger 0 made part of halt group 0.
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
```

Now you can connect gdb to it by doing `target extended-remote :3333`
