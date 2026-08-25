## K3 OpenOCD/gdb Guide

[SpacemiT official Document](https://github.com/spacemit-com/docs-tool/blob/main/en/user_guide/jtag_debug_user_guide.md)

At the time of writing, I'm using the vendor's [modified OpenOCD](https://spacemit.com/community/resources-download/Tools/JTAG%20debugging%20tool)

### K3 Debug Topology

K3 has 4 CPU clusters, 2 X100 + 2 A100. Each cluster has its own debug module (DM), and there's a global DTM that connects to
these 4 DMs. The latest stable OpenOCD relase (0.12) doesn't support configuring per-hart DM base addresses. The support was merged in 
[this commit](https://github.com/openocd-org/openocd/commit/5754aebc49450cc0da5c8a90ebd059160d21f256)
You'll need to at least build upstream OpenOCD from master to get K3 support. Or just use the vendor's version, as there're still
[some patches](https://review.openocd.org/q/owner:zhen.liang%2540spacemit.com) pending review. I haven't checked if they are k3-related.

