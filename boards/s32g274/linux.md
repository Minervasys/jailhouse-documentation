# Linux on the `s32g274`

## Scope of the guide

*This guide is related to how to build and run Linux and Jailhouse on top of the `s32g274`.*

## Resources

- [Microsys User Manual](#how-to-download-the-microsys-user-manual);
- [S32G2 Reference Manual](https://www.nxp.com/webapp/Download?colCode=S32G2RM) -- **REQUIRES SIGN IN**;
- [S32G2 Linux BSP 36.0 User Manual](#how-to-download-s32g2-linux-bsp-36.0-user-manual);

### How to download `S32G2 Linux BSP 36.0 User Manual`

Follow the steps listed in this [comment](https://community.nxp.com/t5/S32G/Where-can-we-download-the-last-Linux-BSP-38-0-User-Manual-for/m-p/1766687/highlight/true#M5477)
from the NXP Forum to download the User Manula of BSP 36.0

### How to download the `Microsys User Manual`

The **Microsys** package (bsp 36) can be downloaded as a tar file from the
[product page](https://www.microsys.de/en/products/system-on-modules/arm-architecture-soms/miriac-mpx-s32g274a)
in the **DOWNLOADS** section.

## Booting the board for the first time

> All the steps below are performed on an Ubuntu 20.04 machine (*version officially supported by NXP and Microsys*).

The `s32g2` has a BSP (board support package) with Yocto (automated build tool) made available
by NXP and extended by Microsys (the board vendor).

The related manuals can be downloaded as declared in the [Resources](#resources) section.

By following the steps reported in the `README` by Microsys, you'll get a ready-to-run SD card
to mount directly on the board.

> **NOTE**: just make sure to install `python2` and to launch this script:

```bash
sources/meta-alb/scripts/host-prepare-ubuntu-mint-debian.sh
```

which will install all the required dependencies, then you can run:

```bash
bitbake microsys-image-auto
```

The final result will be the `sdcard` file that has to be copied over a new SD card.

```bash
cd build_s32g274ar2sbc2/tmp/deploy/images/s32g274ar2sbc2/
sudo dd if=microsys-image-auto-s32g274ar2sbc2-20240524103805.rootfs.sdcard of=/dev/<device> bs=1M && sync
```

## `s32g274` boot flow

The standard boot flow for the `s32g274` is showed in the following diagram:

![boot-flow](/boards/assets/bootflow.png)

The boot flow can be customized in several ways, for example:

- by changing the **switch-2** (see the `Microsys User Manual` and search for `SW2`)
you can decide whether to boot the board from the ROM bootloader or directly from the SD card;
- by changing the **bootable image** from U-Boot shell you can decide which os to boot (*through their
image files*).

### `SW2` switch

By turning **on** the second switch in `SW2` you can boot the board using only the inserted SD card.
This could be useful when you want to change the code of the *bootloaders-chain* (i.e. the
ARM-trusted-firmware and U-Boot) for example for booting U-Boot and Linux with `EL2` exception level.

### Custom boot sequence

From the U-Boot shell check the boot command:

```bash
printenv bootcmd
# bootcmd=pfeng stop; mmc dev ${mmcdev}; if mmc rescan; then run bootfit_sd; fi

printenv bootfit_sd
# bootfit_sd=setenv bootargs ${bootargs_sd} ${sja1110_cfg}; ext4load mmc ${mmcdev}:1 ${loadaddr} boot/fitImage.itb; bootm ${loadaddr}${kconfig}
```

`bootfit_sd` is actually a composition of sub-commands sequentially run.
So, if you need to make some changes in between those phases, first run:

```bash
pfeng stop; mmc dev ${mmcdev}; setenv bootargs ${bootargs_sd} ${sja1110_cfg}; ext4load mmc ${mmcdev}:1 ${loadaddr} boot/fitImage.itb;
```

then simply run each sub-command one after the other:

```bash
bootm start ${loadaddr}${kconfig}; bootm loados; bootm ramdisk; bootm fdt; bootm prep; bootm go
```

## Building Linux

Linux can be built both manually (documented in the
[Jailhouse guide](/boards/s32g274/jailhouse.md#building-linux-manually)) and through Yocto.
All the steps to build Linux with Yocto are documented in the BSP 36.0 User Manual in
section `3.1.8 Additional instructions for developers`.

## Jailhouse on the `s32g274`

See the [dedicated page](/boards/s32g274/jailhouse.md) to understand how to build and enable Jailhouse.
