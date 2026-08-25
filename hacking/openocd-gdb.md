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

Once you get the vendor's OpenOCD (v1.1.2) in my example, there's a `k3.sh` file that looks like:

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

 * `ADAPTER_DRIVER=jlink`: Change it to your adapter config, such as the [FTDI config](https://github.com/ganboing/K3-Docs/blob/master/hacking/jtag-com260-ft2232H.md#openocd-adapter-config) for my CoM260 setup, and copy the file into  ../share/openocd/scripts/interface
 * `SPEED 8000`: The adapter speed that gets set to via `adapter speed <SPEED>`. No need to change unless the session is too slow for you
 * `CLUSTERS`, `CLUSTERx_COREIDS` are the cores OpenOCD connects to. ***Don't specify the ones that's power-gated, otherwise it'll hang the OpenOCD session***. Use the following scripts:
   * `k3-1x1.sh`: Debug only core 0. ***Use this during early boot***
   * `k3-x100.sh`: Debug only X100 cores, cluster 0/1
   * `k3-a100.sh`: Debug only A100 cores, cluster 2/3
   * `k3.sh`: Debug everything
