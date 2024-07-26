# Linux on the `ZCU106`

## Scope of the guide

*This guide is related to how to build and run Linux and Jailhouse on top of the `ZCU106`.*

## Resources

- [Petalinux documentation](https://docs.amd.com/r/2023.2-English/ug1144-petalinux-tools-reference-guide/Overview);
- [ZCU106 memory map](https://docs.amd.com/r/en-US/ug1085-zynq-ultrascale-trm/PL-AXI-Interface);
- [ZCU106 boot modes](https://docs.amd.com/r/en-US/ug1085-zynq-ultrascale-trm/Boot-Modes);
- [Jailhouse tutorial for the ZCU102](https://github.com/siemens/jailhouse/blob/master/Documentation/setup-on-zynqmp-zcu102.md);

## Building Petalinux

### Project setup

Once you have chosen the version to use, in order to execute PetaLinux commands,
you have to source the settings for that particular version.
Below there is an example with the `2023_2` version:

```bash
source /tools/xilinx/2023_2/petalinux/settings.sh
```

> **TIP**: you can add this line in your `.bashrc` to avoid executing *manually*
the command above every time.

### Create the Petalinux project

In order to build PetaLinux, it's advised to start with the **bsp** of the `ZCU106`.
You can download them [here](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/embedded-design-tools/2023-2.html),
be sure to download the **bsp** aligned with the **version** of PetaLinux you are using (`2023_2` in this case).

```bash
petalinux-create -t project --template zynqMP -s bsp/xilinx-zcu106-v2023.2-10140544.bsp -n <project_name>
```

### Configure PetaLinux

```bash
petalinux-config
```

Create a file system compatible with SD card: `Image Packaging Configuration`
→ `Root filesystem type` → **select** `EXT4 (SD/eMMC/SATA/USB)`.

> :warning: to avoid problems with the FPGA module, be sure to **turn off** the `FPGA Manager` → `FPGA Manager`
option, otherwise the `BOOT.BIN` will not contain the FPGA's bitstream.

### Configure the Linux Kernel

In PetaLinux, you can also configure the single components of the project, for example, you can
configure Linux with the following command:

```bash
petalinux-config -c kernel
```

### Configure the `rootfs`

Usually, you have to interact with stuff on the system which needs `root` privileges,
by default, `root` access is **disabled** and you only have the `petalinux` user.
Luckily, you can enable `root` autologin by configuring the `rootfs` component inside
PetaLinux ([source](https://docs.amd.com/r/2023.2-English/ug1144-petalinux-tools-reference-guide/Login-Changes)):

```bash
petalinux-config -c rootfs
```

Then you can enable the  `Image Features` → `[ ] serial-autologin-root` option.

### BONUS: apply patches to Linux

If you need to apply some patches to the standard Linux kernel provided by PetaLinux,
these are the steps required:

```bash
petalinux-devtool modify linux-xlnx
# then the folder below will be created and initialized as a git repository
cd components/yocto/workspace/sources/linux-xlnx/
# Apply your patches here...
```

### Build the whole project

```bash
petalinux-build
```

You can also build a single component:

```bash
petalinux-build -c <component>
```

## Booting Linux on the `ZCU106`

> **NOTE**: these are the steps required to boot the ZCU106 with an SD Card,
other boot modes are not covered in this guide.

In order to boot the `ZCU106` correctly, you have to do the following stuff:

1. create the **boot files** required by the board;
2. prepare an SD card with two partitions:
    - `boot` with the required boot files (i.e. `BOOT.BIN`, kernel image and u-boot script);
    - `root` with the root filesystem;
3. set the `SW6` switch on the board in order to boot an SD card;
4. turn on the board and have fun;

### 1. Creating the boot files

```bash
petalinux-package --boot --u-boot --fpga
```

This command will generate the `BOOT.BIN` (in the `project-root/images/linux` folder)
file which will include all the required stuff to boot correctly the `ZCU106`.

### 2. Preparing the SD card image

With the `--wic` option in PetaLinux, you can create an image with the required
partitions and the files needed:

```bash
petalinux-package --wic --rootfs-file images/linux/rootfs.tar.gz --bootfiles "BOOT.BIN image.ub boot.scr"
```

> **NOTE**: the bootfiles listed in the command above are the only ones required by the
*first-stage bootloader* according to PetaLinux's documentation.

This command will create the `petalinux-sdimage.wic` file in the `images/linux` folder,
this file can then be copied on the SD card with a tool like `dd`:

```bash
sudo dd if=petalinux-sdimage.wic of=/dev/sdX conv=fsync && sync
```

> **TIP**: if you are building petalinux on a remote workstation, you can't attach an SD card
to it, to avoid downloading 6 GB of stuff (the `wic` file's size), you can compress it
with `tar` (since the image is padded with 0s, you will save a lot of space) and
download the compressed archive. On the host you will decompress it and use `dd` as
usual.

### 3. Setting the right `SW6` configuration

As described in the **Boot modes** reference posted [here](#resources), the right setup is the one
called `SD1 LS (3.0)` which needs the switch configured as follows:

```
sw. 1 0 0 0
nr. 1 2 3 4
```

### 4. Turning on the board

Now it's time to actually boot the `ZCU106`. First, connect a **micro-usb** cable to the UART/JTAG port
on the board, then connect your laptop to the serial port with the following `minicom` command:

```bash
# the UART/JTAG cable will create 4 ttyUSBs: ttyUSB0 .. ttyUSB3
# ttyUSB1 is the one connected to the UART1 of the board.
sudo minicom -D /dev/ttyUSB1
```

> **NOTE**: remember to turn off `Hardware control flow` in `minicom`: `CTRL-A + O` → `Serial port setup`.

Then you can turn on the board and wait Linux to be started.

### BONUS: enable output from the UART2

If you want to see the stuff printed on UART2, this is the command to execute in Linux:

```bash
getty 115200 ttyPS1 &
```

### BONUS: boot a custom `device tree blob` and a custom `Image`

To speed up development, you can also customize by hand the `dtb` and build a custom `Image`,
then you can boot Linux with these updated components with this command from the `u-boot` shell:

```bash
fatload mmc 0 0x2A00000 <custom-system.dtb> && fatload mmc 0 0x3000000 <custom-Image> && booti 0x3000000 - 0x2A00000
```

## Jailhouse on the `ZCU106`

See the [dedicated page](boards/zcu106/jailhouse.md) to understand how to build and enable **Jailhouse**.
