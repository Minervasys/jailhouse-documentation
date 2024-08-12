# Jailhouse on the `s32g274`

This guide will explain the required steps to have a working instance of Jailhouse on top of the
`s32g274` SoC.

## Jailhouse -- root cell configuration for the `s32g274`

The `s32g274` has two root cells available, a *standard* one and a *colored* one, they
can be checked in the following files:

- `configs/arm64/s32g2.c`
- `configs/arm64/s32g2-col.c`

### `s32g274` memory regions

The addresses of our interest are the ones which refer to the DRAMs available on the board.
In particular, we have:

```
1) 0x00_8000_0000 - 0x00_FFFF_FFFF -> first 2 GB block of DDR_0
2) 0x08_8000_0000 - 0x08_FFFF_FFFF -> second 2 GB block of DDR_0
```

The entire memory map can be seen in the `S32G2_Memory_Map` Excel file attached to the
`S32G2 Reference Manual` (listed in the [Resources](/boards/s32g274/linux.md#resources) section).

The memory reserved for the `root cell` and the `inmates` is the following:

```dts
jailhouse@0x880000000 {
	no-map;
	reg = <0x8 0x80000000 0x00 0x40000000>;
};
```

## Integrating Jailhouse build process with the `s32g2`'s one

By following the steps reported in the `README` by Microsys, you'll get a ready-to-run SD card
to mount directly on the board (*it's already available*). The whole system will start at `EL3`
privilege level (with the ARM Trusted Firmware) which will launch `u-boot` at `EL1` privilege
level which is **not suitable for hypervisors** (for more details about **ARM's Exception Levels**,
see the [documentation](https://developer.arm.com/documentation/102412/0103/Privilege-and-Exception-levels/Exception-levels)).

Since we need to run the Jailhouse Hypervisor, the system must execute in `EL2`, to reach this goal,
we'll build everything **manually** (but you can update Yocto's recipes too).

> The NXP Manual shows the detailed instructions for building manually the BSP in **Chapter 3.2**.

> **NOTE**: to avoid an additional layer of complexity, the updated components (.i.e. Linux, u-boot and ATF) have
been compiled manually without relying on Yocto's build system.

### Building Linux manually

Since the SD card is already prepared with a valid `rootfs` in a valid partition,
you can **easily** build Linux manually without relying on **Yocto**. With these steps:

1. Clone the repository and switch to the right branch:

```bash
git clone https://github.com/nxp-auto-linux/linux.git
git switch release/bsp36.0-5.15.85-rt
```

2. Apply Microsys patches available in the [patches](/boards/s32g274/patches/) directory:

```bash
cd /path/to/nxp-linux
git am /path/to/jailhouse-documentation/boards/s32g274/patches/linux/*
```

3. Apply Jailhouse enabling patches for `5.15.y` available [here](/jailhouse-enabling-patches/5.15.y/):

```bash
cd /path/to/nxp-linux
git am /path/to/jailhouse-documentation/jailhouse-enabling-patches/5.15.y/*
```

4. Configure and build Linux:

> **NOTE**: an already prepared configuration for the kernel is available [here](/boards/s32g274/configs).

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
	s32g274ar2sbc2_defconfig
# CONFIG_OF_OVERLAY and CONFIG_KALLSYMS_ALL must be enabled
# CONFIG_LOCALVERSION_AUTO must be disabled
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig
# now we can build the kernel
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```

5. Create the flattened image tree for u-boot with `mkimage` (from `u-boot-tools` package):

```bash
mkimage -f kernel-s32g274ar2sbc2.its fitImage.itb
```

6. Copy the new image file in the `boot` folder on the eMMC card (you need root privileges for this):

```bash
cp fitImage.itb <path/to/emmc/boot/>
```

### Building `u-boot` manually

1. Clone the repository:

```bash
git clone https://github.com/nxp-auto-linux/u-boot
git switch release/bsp36.0-2020.04
```

2. Apply Microsys patches available in the [patches](/boards/s32g274/patches/) directory:

```bash
cd /path/to/nxp-u-boot
git am /path/to/jailhouse-documentation/boards/s32g274/patches/u-boot/*
```

3. Configure and build `u-boot`:

```bash
make CROSS_COMPILE=aarch64-linux-gnu- s32g274ar2sbc2_atf_defconfig
make CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```

### Building ARM Trusted Firmware (ATF) manually

In order to start `u-boot` and `Linux` in `EL2`, we need to build the **ATF** with Hypervisor
support.

1. Clone the repository:

```bash
git clone https://github.com/nxp-auto-linux/arm-trusted-firmware.git
git switch release/bsp36.0-2.5
```

2. Apply Microsys patches available in the [patches](/boards/s32g274/patches/) directory:

```bash
cd /path/to/nxp-atf
git am /path/to/jailhouse-documentation/boards/s32g274/patches/atf/*
```

3. Build **ATF**:

```bash
make CROSS_COMPILE=aarch64-linux-gnu- ARCH=aarch64 \
    PLAT=s32g274ar2sbc2 \
    BL33=<path/to/u-boot-nodtb.bin> \
    S32_HAS_HV=1
```

> **NOTE**: `u-boot-nodtb.bin` is located in the `u-boot` build path.

4. Copy the `fip.s32` image in the initial available space in the eMMC card
(you need root privileges for this):

> The `ATF` blob will be copied in the initial space available in the eMMC's first
partition.

```bash
sudo dd if=<path/to/fip.s32> of=/dev/<sdcard_dev> skip=512 seek=512 \
    iflag=skip_bytes oflag=seek_bytes conv=fsync,notrunc
```

> When you have to build manually also the components of the boot system, such as the
**ARM Trusted Firmware** and **u-boot**; the Linux kernel will show some **backtrace logs** during
the boot related the probing of some Microsys' modules (probably because we are missing
something from Microsys' build recipes for Yocto), the boot process will complete and the system
will be usable, but it's not 100% guarantee that everything will work (especially Ethernet).

### Building Jailhouse

1. Clone the repository:

```bash
git clone https://github.com/Minervasys/jailhouse.git
git switch minerva/next
```

2. Compile and install Jailhouse on the eMMC card (you need root privileges for this):

```bash
sudo make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
	KDIR=<path/to/linux/build/dir> \
	DESTDIR=<path/to/emmc/dir> install
```

## Boot the system on the board

Turn **on** switch 2 (*User Manual, section 3.3.1*) to boot the board directly from the eMMC card
and you'll be able to boot Linux in EL2 and to enable Jailhouse on the s32g2.

## Testing cache coloring

> **NOTE**: make sure to check and update, if necessary, the `include/jailhouse/mem-bomb.h` header file
in order to load the `s32g2` **memory configuration**.

> :warning: If you want to use coloring without specifying the way size (autodetected), you have to enable
`CONFIG_DEBUG` in Jailhouse's configuration file (`include/jailhouse/config.h`).

This is a script for testing that `cache-coloring` is working as expected:

```bash
#!/bin/bash

SIZE=${SIZE:-1048576}

JAILHOUSE_DIR=/home/root/jailhouse

modprobe jailhouse
jailhouse enable $JAILHOUSE_DIR/s32g2-col.cell

for i in `seq 0 1`; do
	jailhouse cell create $JAILHOUSE_DIR/s32g2-bomb$i-col.cell
	jailhouse cell load col-mem-bomb-$i $JAILHOUSE_DIR/mem-bomb.bin
	membomb -c $(($i+1))
	membomb -c $(($i+1)) -s $SIZE
done
jailhouse cell list
sleep 1s

for i in `seq 0 1`; do
	jailhouse cell start col-mem-bomb-$i
	membomb -c $(($i+1)) -s $SIZE -e
done

jailhouse cell create $JAILHOUSE_DIR/s32g2-inmate-demo-col.cell
jailhouse cell load s32g-inmate-demo $JAILHOUSE_DIR/gic-demo.bin
jailhouse cell start s32g-inmate-demo
jailhouse cell list

sleep 2s

for i in `seq 0 1`; do
	membomb -v -c $((i+1))
	sleep 1s
	jailhouse cell destroy col-mem-bomb-$i
done

sleep 1s
jailhouse cell destroy s32g-inmate-demo
jailhouse disable
```

## Testing memguard

This is a script for testing that `memguard` is working as intended:

```bash
#!/bin/bash
# params: 100000 1000 0

if [ $# -lt 3 ]; then
	echo "$0 Missing parameters"
	exit -1
fi

SIZE=1048576

JAILHOUSE_DIR=/home/root/jailhouse

modprobe jailhouse
jailhouse enable $JAILHOUSE_DIR/s32g2-col.cell

for i in `seq 0 1`; do
	jailhouse memguard $(($i+1)) $1 $2 $3
	jailhouse cell create $JAILHOUSE_DIR/s32g2-bomb$i-col.cell
	jailhouse cell load col-mem-bomb-$i $JAILHOUSE_DIR/mem-bomb.bin
	membomb -v -c $(($i+1))
done
jailhouse cell list
sleep 3s

for i in `seq 0 1`; do
	jailhouse cell start col-mem-bomb-$i
	membomb -v -c $(($i+1)) -s $SIZE -e
done

jailhouse cell create $JAILHOUSE_DIR/s32g2-inmate-demo-col.cell
jailhouse cell load s32g-inmate-demo $JAILHOUSE_DIR/gic-demo.bin
jailhouse cell start s32g-inmate-demo
jailhouse cell list

sleep 10s
jailhouse cell destroy s32g-inmate-demo
sleep 1s

for i in `seq 0 1`; do
	membomb -v -c $(($i+1))
	jailhouse cell destroy col-mem-bomb-$i
done

sleep 3s

jailhouse disable
```
