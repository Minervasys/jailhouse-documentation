# S32V234SBC
NOTE: Most of the information for the S32G2 are also applicable for the S32V.

Due to toolchain updates, it is complex to build the S32V from scratch using the
[Yocto](https://github.com/nxp-auto-linux/auto_yocto_bsp) environment from NXP.

Best is to just download a rootfs from e.g., https://www.nxp.com/design/design-center/software/embedded-software/linux-software-and-development-tools/bsp-for-s32-microcontrollers-and-processors:BSP-S32
or just build an Arm64 rootfs and use it with the S32V (note the side of the SDCard).

## Linux
Similarly to the S32G, the S32V Linux is supported here:
https://github.com/nxp-auto-linux/linux.git

For simplicity, since it was already supported before, we use the 5.4.24
version of the kernel that was supported before.

```
git checkout -b <branchname> v5.4.24_bsp26.1
git am 0001-scripts-dtc-Remove-redundant-YYLOC-global-declaratio.patch
git am 0002-s32v234sbc-jailhouse-support-integrate-PSCI-emulatio.patch
```

## U-Boot
- Checkout `v2020.04_bsp26.1` (https://github.com/nxp-auto-linux/u-boot/tree/v2020.04_bsp26.1)
- Apply u-boot patches

NOTE: Contrary to Xilinx boards it's not sufficient to copy the u-boot on the
first partition, but the first sectors of the mmc must be overwritten e.g.:
```
sudo dd if=u-boot-dtb-fdt.s32 of=/dev/sdb bs=512 seek=8 conv=fsync
```
(See the information in the NXP Manual e.g.,
S32V234_LinuxBSP_23.1_User_Manual.pdf)

## S32V232 and Jailhouse
- u-boot: apply patches to change boot-up of Linux into EL2
- use `mem=256M` on the kernel command line for the configuration (inmates configs assumes to use DDR from the second bank i.e., > 256 MB)
- **set** CONFIG_MACH_NXP_S32v2 into include/jailhouse/config.h to enable the correct compilation options

