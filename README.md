# Jailhouse documentation

This repository aims to be the primary reference for working on the version of
[Jailhouse](https://github.com/Minervasys/jailhouse.git) maintained by **MinervaSys**.

The repository is organized in order to provide guides about how to **build** and **run**
**Linux** with **Jailhouse** for each supported board by **MinervaSys**.

## How to use this repository

### `boards` directory

The `boards` directory contains all the resources to end up with a working instance of `Linux + Jailhouse`
for the board of interest.

Each directory contains the following files and subdirectories: `patches`, `linux.md` and `jailhouse.md`.

The `patches` directory stores all the patches required (*typically vendor specific
stuff or memory reservations for Jailhouse*) by each component of the build-chain in order
to support Jailhouse on top of Linux, typically this consists of three subfolders:
- `atf` (*Arm-Trusted-Firmware*): the very first bootloader of the boot chain in each Arm board,
it launches `u-boot`;
- `u-boot`: the bootloader responsible for booting Linux (it can be interactive,
thus it permits booting custom kernels);
- `linux`: the Linux kernel.

The `linux.md` file contains general information about the board and how to run Linux on it.

The `jailhouse.md` file explains the required steps to take in order to enable Jailhouse on Linux for each board.

### `jailhouse-enabling-patches` directory

This directory contains general Linux patches that must be applied to build Jailhouse as a
module for the board's kernel. These patches are kernel version specific rather than board specific,
and they need to be applied each time you use Jailhouse. Consequently, the patches are organized by
kernel version.

## Compatibility table

Each supported board has been tested with a specific Kernel version, this table lists the public repositories
and the specific kernel version used for the testing:

| Board  | Linux repository | Kernel version supported |
| :---- | :------ | :------ |
| `coral-imx8mq` | [google repo](https://coral.googlesource.com/linux-imx) | 5.4.y |
| `rock5b` | [radxa-kernel](https://github.com/radxa/kernel/tree/linux-5.10-gen-rkr3.4) | 5.10.y |
| `s32g274`  | [nxp-auto-linux](https://github.com/nxp-auto-linux/linux/tree/release/bsp36.0-5.15.85-rt) | 5.15.85 |
| `zynq-ultrascale` | [linux-xlnx](https://github.com/Xilinx/linux-xlnx/tree/xlnx_rebase_v6.1_2023.2) | 6.1.30 |

## General concepts related to Jailhouse

> **NOTE**: This paragraph can be thought like a list of relevant information that should be understood and
kept in mind while working with Jailhouse.

### Terminology

- `cell`: Jailhouse is a static partitioning Hypervisor, the **VMs** it runs are called `cells`;
- `inmate`: synonym of `cell`;
- `root cell`: the main VM which creates and manages the other `cells`;
- `cell configuration`: `C` source file which defines the memory regions, interrupts and devices which a
`cell` can use;

### `cell` configuration nomenclature

Usually a `root cell` is defined as `<board_name>.c` if it's a standard `root cell` or `<board_name>-col.c`
if it's a colored one.

A normal `cell` is usually defined as `<board_name>-<inmate_scope>.c`, where the `<inmate_scope>` tells
if the `cell` is used as an `inmate-demo` (i.e. bare-metal), a `memory bomb` or a `linux` virtual machine.

### Compile Jailhouse for the first time

Once you start from a fresh cloned repository and you want to compile Jailhouse for a target system,
be sure to **create** the configuration file `config.h` in the `include/jailhouse/` directory.

Jailhouse does not have a `kconfig` system, so you have to add the desired options directly in that
file as preprocessor macros.

```bash
touch include/jailhouse/config.h
```

### Makefile's variable passed while compiling Jailhouse

A typical compilation of Jailhouse requires the following variables to be passed:

- `ARCH`: architecture of the target system that will run Jailhouse;
- `CROSS_COMPILE`: tool chain to use for cross compilation;
- `KDIR`: path (either relative or absolute) to the directory which contains the compiled Linux kernel;
- `DESTDIR`: **absolute** path to the directory containing the root file system used on the target system;

### Handling `rootfs`

Each target must have a file system in which Jailhouse is installed, the file system can be on an eMMC card
or in a tftp directory, regardless of that, while compiling Jailhouse be sure to pass its path as an
**absolute** one, otherwise the make subprocesses will not be able to create / copy the relevant files!

### Configuring `cells`

The most important part of `cells` configuration is the memory regions one, usually you have to follow a 1:1
mapping between this configuration and the `/proc/iomem` defined for a certain target system. Also, each
memory region has to be defined as `executable` memory or as `io` memory.

> Keep in mind that if some memory is used as `executable` in Linux and is declared as `io` in jailhouse,
this will cause problems!

### Relevant files to copy in the target `rootfs`

When you want to easily test freshly compiled stuff and you want to be quick, these are the files associated
with the main components of Jailhouse:

- `jailhouse.ko` - is the kernel module, thus it's the compilation product of the `driver/` directory in Jailhouse;
    - its location can be `/lib/modules/<kernel_version/...` if you use `modprobe` or can be anywhere if you
    use `insmod`;
- `jailhouse.bin` - is the actual hypervisor code, thus it's obtained by compiling the `hypervisor/` directory in
Jailhouse;
    - its location must be `/lib/firmware`
- `tools/jailhouse` - is the binary which as act as interface to all jailhouse commands from the command line;
    - its location must inside the `$PATH` variable, usually `/usr/local/sbin/`;
- `tools/membomb` - is the binary used for managing the memory bomb `cells` from the `root cell`;
    - its location must inside the `$PATH` variable, usually `/usr/local/sbin/`;
- `<cell_name>.cell` - is the binary representing a `cell` (whether root or not) which has to be passed to the
`jailhouse` command line tool if you want to **enable** the `root cell` or **create** a `cell`;
    - its location is not relevant, it can be anywhere in the `rootfs`;
