# Jailhouse on the `Coral`

## Executing Jailhouse on the Coral

Jailhouse has been already ported for the `imx8mq` (Coral's SoC) by NXP (Manufacturer),
you can find the root cell configuration in `configs/arm64/imx8mq.c`.

The configuration only needs the setup for **MemGuard** and for **Cache coloring**.

### `imx8mq` memory regions

The device tree reserves only 1 GB (`0x40000000`) for the main memory in Linux, so the actual memory
range is:

```
0x40000000 - 0x80000000
```

While the available memory region on the system is:

```
0x40000000 - 0xFFFFFFFF
```

Thus, Jailhouse can reserve some memory in this unused space with no problems:

```
jailhouse@0xffaf0000  {
	no-map;
	reg = <0x0 0xffaf0000 0x0 0x510000>;
};
```

> **NOTE**: this has been added inside `arch/arm64/boot/dts/freescale/fsl-imx8mq-som.dtsi`.

## Building Linux

> :warning: Make sure to have installed the package /lib/firmware on the building machine.

> :construction: **this part is still a work in progress, the instructions are not completed yet**.

```bash
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-

make coral_default_defconfig
# Enable CONFIG_HOTPLUG_CPU, CONFIG_KALLSYMS, CONFIG_KALLSYMS_ALL
make menuconfig
make -j$(nproc)
```

> The file to copy inside the Coral's `boot/` folder is `arch/arm64/boot/Image`.

## Building Jailhouse

Since Jailhouse needs to install stuff on the target file system, but the Coral does not
work with a SD card, the easiest way to install stuff on the Coral is through `sshfs`.

> **NOTE**: make sure to mount the file system as root in order to have the right privilege
to write inside it.

Then you can compile Jailhouse as usual by setting the mounted remote file system as the `DESTDIR`:
```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- KDIR=<path/to/kernel> DESTDIR=<path/to/coral/file-system> install
```

> :warning: Be sure to give the **absolute path** to the mounted file system!

> **NOTE**: remember to copy the compiled `imx8mq` cells in the remote file system!
*You can also copy the entire Jailhouse folder in the target*.

### Possible problems

If the `make install` instruction is **not** installing Jailhouse components correctly, you have to
install them **manually**.

> *This could happen because of some problems between **make subprocesses** and the **remote file system**.*

For example:
```bash
cp driver/jailhouse.ko <path/to/coral/file-system>/home/mendel
cd tools && install jailhouse demos/ivshmem-demo membomb utilstress <path/to/coral/file-system>/usr/local/sbin
install -m 644 jailhouse-completion.bash <path/to/coral/file-system>/usr/share/bash-completion/completions/jailhouse
```

## Interacting with Jailhouse: how-to

These steps are suggested for the best experience with **Linux + Jailhouse**:

1. open the serial port with `minicom` to launch our custom version of Linux through `u-boot`;

2. use SSH to connect to the `mendel` user on the Coral, in this way we avoid the annoying and
unresponsive `minicom` shell, furthermore we will see in `minicom` the output of Jailhouse with no dirt from the terminal!

## Testing Jailhouse

This is an example script (MemGuard and coloring combined) to see how to test Jailhouse on the Coral:

> **NOTE**: be sure to launch it as **root**.

```bash
#!/bin/bash

cd /home/mendel
insmod jailhouse.ko

jailhouse enable jailhouse/configs/arm64/imx8mq-col.c
jailhouse cell create jailhouse/configs/arm64/imx8mq-membomb0-col.c
jailhouse cell load jailhouse/inmates/demos/arm64/mem-bomb.bin
jailhouse cell start col-membomb-0
membomb -v -c 1 -s 1048576 -t 100000 -m 1000 -e
```
