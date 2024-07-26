# Linux on the `Rock5B`

## Scope of the guide

*This guide is related to how to build and run Linux and Jailhouse on top of the `Rock5B`.*

## Resources

- [Board documentation](https://docs.radxa.com/en/rock5/rock5b);
- [RK3588 TRM](https://github.com/FanX-Tek/rk3588-TRM-and-Datasheet);
- [BSP documentation](https://radxa-repo.github.io/bsp/);

## Booting the board for the first time

The `Rock5B` has an official OS image called `RadxaOS` which can be installed by following
this [guide](https://docs.radxa.com/en/rock5/rock5b/getting-started/install-os/boot_from_sd_card)
from the documentation.

After this, you will end up with a bootable SD card to insert into the board and you'll see `RadxaOS`
(Debian fork) booting.

> **NOTE**: you can also connect the board to the network to update and / or install additional packages!

### `Rock5B` boot flow

As for any standard ARM board, the boot flow is the following:

![boot-flow](boards/assets/bootflow.png)

### Login into the system

| User   | Password |
| :----: | :------: |
| radxa  | radxa    |
| rock   | rock     |

## Building Linux

Linux can be built on your laptop or on the board itself, this guide will cover only the steps
required for building Linux on your laptop.

For **additional** documentation about how to build `u-boot` and the Linux Kernel, please refer to this
[page](https://docs.radxa.com/en/rock5/rock5b/low-level-dev).

Radxa offers a build tool called `bsp` ([repository](https://github.com/radxa-repo/bsp.git)), they suggest
running it on Debian or Ubuntu (also through a Docker container). The bsp will build the stuff through a
**podman** container, so you don't have to install too many
[dependencies](https://docs.radxa.com/en/rock5/rock5b/low-level-dev/kernel#install-dependencies).
The `bsp` documentation is linked in the [Resources](#resources) section.

### Prepare the sources

Supposing you have followed the instructions to set up the `bsp` repository, you can prepare the Linux
sources with the following command:

```bash
./bsp --no-build linux rockchip
```

### Build a new kernel package

The build command will build Linux and will produce a set of Debian packages as the result. If you have
updated the kernel code and / or configuration, you have to add the `--dirty` option to avoid the tool
resetting the git source tree.

Furthermore, the `bsp` requires a revision number, this will determine the default boot order of the
kernels installed on the board, so you may want to use a number greater than the one used by the installed
kernel on the board.

The final command will look like this:

```bash
./bsp --dirty -r <revision-number> linux rockchip
```

The only relevant Debian packages that you have to copy on the board are:

- linux-headers-5.10.110-<revision-number>-rockchip_5.10.110-<revision-number>_arm64.deb
- linux-image-5.10.110-<revision-number>-rockchip_5.10.110-<revision-number>_arm64.deb

### Install the new kernel

On the board, just give the following commands:

```bash
sudo dpkg -i linux-headers-5.10.110-<revision-number>-rockchip_5.10.110-<revision-number>_arm64.deb
sudo dpkg -i linux-image-5.10.110-<revision-number>-rockchip_5.10.110-<revision-number>_arm64.deb
```

Then, once you reboot the board, you will end up running with your freshly compiled kernel.

### BONUS: changing the boot order of the installed kernels

If you are lazy and you don't want to change each time the revision number, you can stick with the
default one (1) and update `u-boot`'s boot order through the configuration file located here:
`/usr/share/u-boot-menu/conf.d/radxa.conf`.

You can edit this file with your editor of choice and append this line:

```bash
U_BOOT_DEFAULT="l1"
```

Where `l1` should be your newly installed kernel, you can check the right position in the
`/boot/extlinux/extlinux.conf` file.

Once you have inserted the right list value, just update `u-boot` settings with:

```bash
sudo u-boot-update
```

And your kernel will be the default one to be booted from now on.

### BONUS: boot a custom kernel from `u-boot` shell

The `u-boot` binary provided by Radxa does not offer an interactive shell, but if you want to compile
also a custom version of u-boot to enable it, then you can choose the kernel to launch with these commands
(supposing you have moved the `vmlinuz` binary and your custom `dtb` in the `/boot/` folder):

```
load mmc 1:3 0x2200000 /boot/vmlinuz-5.10.110-<revision-number>-rockchip
setenv bootargs "root=/dev/mmcblk1p3 rootwait"
load mmc 1:3 0xa100000 /boot/custom_rock5b.dtb
booti 0x2200000 - 0xa100000
```

## Jailhouse on the `Rock5B`

See the [dedicated page](boards/rock5b/jailhouse.md) to understand how to build and enable Jailhouse.
