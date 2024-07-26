# Jailhouse on the `Rock5B`

## Executing Jailhouse on the `Rock5B`

Jailhouse is already supported on the `ZCU106`, you can find the root cells (a standard
one and a *colored* one) in the following files:

- `configs/arm64/rk3588-rock5b.c`
- `configs/arm64/rk3588-rock5b-col.c`

## Resources

- [Patches for Rock5B kernel](jailhouse-enabling-patches/5.10.y/);

## `Rock5B` memory regions

The `Rock5B` can have 4 GB, 8 GB or even 16 GB of main memory, these are the address ranges
which have been checked for the 4 GB and 8 GB versions:

```
First bank of around 3.8 GB: 0x00200000 - 0xefffffff
[8 GB only] - Second bank of 4 GB: 0x100000000 - 0x1ffffffff
[8 GB only] - Third bank of 256 MB: 0x2f0000000 - 0x2ffffffff
```

In order to make Jailhouse compatible for all the possible memory configurations, the memory
required has been reserved inside the first memory bank:

```dts
/* Reserve memory for Jailhouse range: 20000000 - 2D000000 */
jailhouse@20000000 {
	no-map;
	reg = <0x0 0x20000000 0x0 0x0D000000>;
};
```

This property has been written in `Rock5B` dts: `arch/arm64/boot/dts/rockchip/rk3588-rock-5b.dts`.

## Building Linux

> :warning: Make sure to **apply** the required patches linked in the [Resources](#resources) section.

### Configure the Kernel

Jailhouse needs the following configuration variable to be **enabled**:

- `CONFIG_OF_OVERLAY`;
- `CONFIG_KALLSYMS_ALL`;

Also, you have to **disable** the following options to avoid compilation and runtime errors:

- `CONFIG_ARM64_VHE`;
- `CONFIG_VIRTUALIZATION`;
- `CONFIG_MODVERSIONS`;

The Linux kernel can be configured inside the `.src/linux` directory
(inside the `bsp` project directory) with the standard:

```bash
make ARCH="arm64" CROSS_COMPILE="aarch64-linux-gnu-" menuconfig`
```

Then it can be compiled with the same instruction listed in the main `Rock5B` guide:

```bash
./bsp --dirty -r <revision-number> linux rockchip
```

## Building Jailhouse

Building Jailhouse for the `Rock5B` can be a bit more difficult than with other boards,
this because Linux has been compiled with a podman container which has native arm64 support
through `qemu-user-static`, so, if you don't compile Jailhouse with the same container
(or one with the same packages installed) you will end up with the following compilation error:

```bash
qemu-aarch64-static: Could not open '/lib/ld-linux-aarch64.so.1': No such file or directory
make[2]: *** [scripts/Makefile.modpost:169: /home/filip/Minerva/Projects/product/jailhouse/Module.symvers] Error 255
make[1]: *** [Makefile:1829: modules] Error 2
make: *** [Makefile:40: modules] Error 2
```

You can avoid using the container for building Linux with the `--native-build` flag, but this
approach will not be covered in this guide. This because you can easily build with the same
Podman container used by the `bsp`.

> **NOTE**: The container's Dockerfile is located inside the `container`
directory of the `bsp` repository.

Before running the container, be sure to be in a directory which can access both the `linux` directory
inside `bsp/.src/` and the `jailhouse` directory, then you can launch it with this command:

```bash
podman run --rm --name bsp --workdir <your/custom/directory> \
    --mount type=bind,source=<your/custom/directory>,destination=<your/custom/directory> \
    -it \
    --privileged \
    ghcr.io/radxa-repo/bsp:main \
    bash
```

Before building Jailhouse, be sure to create the `include/jailhouse/config.h` file with at least
these options enabled:

```C
/* Print error sources with filename and line number to debug console */
#define CONFIG_TRACE_ERROR             1
#define CONFIG_ARM_GIC_V3              1
/*
 * Set instruction pointer to 0 if cell CPU has caused an access violation.
 * Linux inmates will dump a stack trace in this case.
 */
#define CONFIG_CRASH_CELL_ON_PANIC 1
#define CONFIG_DEBUG 1
#define CONFIG_MACH_RK3588 1
```

Then you can build Jailhouse as usual:

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
	KDIR=</absolute_path/to/bsp/.src/linux> \
	DESTDIR=</absolute_path/to/rock5b/rootfs> \
	install
```

> **NOTE**: this guide will not cover how to pass file between your laptop and the `Rock5B`.

## Testing Jailhouse

> **NOTE**: for simplicity, you can copy the jailhouse folder with all the compiled cells
in the board's file system.

Here you can find an example of Jailhouse with `cache-coloring` and `memguard`:

> **NOTE**: be sure to launch it as **root**.

```bash
insmod jailhouse.ko
jailhouse enable jailhouse/configs/arm64/rk3588-rock5b-col.cell
jailhouse cell create jailhouse/configs/arm64/rk3588-rock5b-bomb0-col.cell
jailhouse cell load col-mem-bomb-0 jailhouse/inmates/demos/arm64/mem-bomb.bin
jailhouse cell start col-mem-bomb-0
membomb -v -c 1 -s 1048576 -t 100000 -m 1000 -e
```
