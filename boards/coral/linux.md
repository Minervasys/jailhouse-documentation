# Linux on the `Coral`

## Quick info

### Board details

- The board is supported by a Linux distribution developed by Google called Mendel Linux;
- The board has on-chip memory, you will only need an SD card for flashing it with Mendel Linux;

## Resources

- [Documentation page](https://coral.ai/docs/dev-board/get-started/) for the `Coral`;
- [GettingStartd - Mendel Linux build system](https://coral.googlesource.com/docs/+/refs/heads/master/GettingStarted.md);
- **Upstream repository** of [Mendel Linux](https://coral.googlesource.com/?format=HTML);

## Setup

Follow the official [guide](https://coral.ai/docs/dev-board/get-started/) and you will end up
with Mendel Linux running on the on-chip memory of the `Coral`.

## Boot flow

The diagram below shows the standard boot flow for the `Coral`:

![boot-flow](boards/assets/bootflow.png)

**u-boot** starts with a
[custom script](https://coral.googlesource.com/uboot-imx-debian/+/refs/heads/master/debian/boot.txt)
which loads a specific kernel image and a specific device tree in memory, then it boots the image.

### Login into the system

| User   | Password |
| :----: | :------: |
| mendel | mendel   |

## Boot a custom Kernel

Since the standard workflow of the `Coral` is based on an on-chip memory which contains the whole
file system, the new kernel image and device tree should be passed to the board via a tool like `ssh`.
Thus, you have to connect to the board in some way through point-to-point or the local network.

> **NOTE**: the `Image` file can be named `vmlinuz-<custom_version>`, the coral's device tree
is `fsl-imx8mq-phanbell.dtb` (located in `arch/arm64/boot/dts/freescale/`) and can be named as
you wish.  
**Make sure to update the instructions below accordingly!**

Once the new `image` and `dtb` have been copied in the `/boot` folder of the board's file system, reset the
board and give these commands inside the **u-boot** shell (they can also be turned into a script, if you
prefer):

```
setenv fdt_addr 0x43000000
setenv image vmlinuz-<custom_version>
setenv fdt_file fsl-imx8mq-phanbell.dtb
setenv root "PARTUUID=70672ec3-5eee-49ff-b3b1-eb1fbd406bf5"

cmdline="console=ttymxc0,115200 console=tty0 earlycon=ec_imx6q,0x30860000,115200 root=${root} rootfstype=ext4 rw rootwait init=/sbin/init net.ifnames=0 pci=pcie_bus_perf"

ext2load mmc ${bootdev}:1 ${loadaddr} ${image}
setenv bootargs ${cmdline} ${extra_bootargs}

ext2load mmc ${bootdev}:1 ${fdt_addr} ${fdt_file}
fdt addr ${fdt_addr}
fdt resize

booti ${loadaddr} - ${fdt_addr}
```

This is basically an *adaptation* of the **default boot script** for the `Coral`, but with only the
relevant stuff that we need to boot a custom Kernel.

## Enable internet access to the board

- To download packages from the package manager, add the Google's `apt-key` inside `apt`:
```bash
wget -q -O - https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```

- Set the current time to avoid TSL/HTTPS certificate errors:
```bash
sudo date -s 'yyyy-mm-dd hh:mm:ss'
```

## Jailhouse support

See the [dedicated page](boards/coral/jailhouse.md) to understand how to enable **Jailhouse** on the `Coral`.
