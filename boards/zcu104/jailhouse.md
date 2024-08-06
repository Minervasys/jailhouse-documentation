# Jailhouse on the `ZCU104`

## Executing Jailhouse on the `ZCU104`

Jailhouse is already supported on the `ZCU104`, you can find the root cells (a standard
one and a *colored* one) in the following files:

- `configs/arm64/zynqmp-zcu104.c`
- `configs/arm64/zynqmp-zcu104-root-col.c`

## Resources

- [Jailhouse tutorial for the ZCU102](https://github.com/siemens/jailhouse/blob/master/Documentation/setup-on-zynqmp-zcu102.md);

## `ZCU104` memory regions

The device tree reserves, by default, the following address range for main memory:

```
0x00000000 - 0x7FF00000
```

Which is less than the 2 GB available on the board, the whole address range is:

```
0x00000000 - 0x80000000
```

The memory reserved for the `root cell` and the `inmates` is the following:

```
reserved-memory {
		#address-cells = <0x2>;
		#size-cells = <0x2>;
		ranges;

		jailhouse@0x7e000000 {
				no-map;
				reg = <0x0 0x7e000000 0x0 0x2000000>;
		};

		inmates@0x25000000 {
				no-map;
				reg = <0x0 0x25000000 0x0 0x37000000>;
		};
};
```

But this configuration needs that Linux uses the whole available memory, thus we have
to **update** the `system.dtb` (i.e. the device tree) of PetaLinux.
According to the documentation, you can customize the device tree by adding stuff
inside the following file in the project folder of PetaLinux:

```bash
project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi
```

Then you can just replace the file with this device tree and you have now reserved the
memory for Jailhouse correctly:

```
/include/ "system-conf.dtsi"
/ {
        memory@0 {
                device_type = "memory";
                reg = <0x0 0x0 0x0 0x80000000>;
        };

        reserved-memory {
                #address-cells = <0x2>;
                #size-cells = <0x2>;
                ranges;

                jailhouse@0x7e000000 {
                        no-map;
                        reg = <0x0 0x7e000000 0x0 0x2000000>;
                };

                inmates@0x25000000 {
                        no-map;
                        reg = <0x0 0x25000000 0x0 0x37000000>;
                };
        };
};
```

## Building Linux

> **NOTE**: this guide relies on PetaLinux for building stuff on the `ZCU104`.

> :warning: Make sure to **apply** the required
[patches](jailhouse-enabling-patches/6.1.y/) to support Jailhouse.

> **NOTE**: an already prepared configuration for the kernel is available [here](boards/zcu104/configs).

### Configure the Kernel

Jailhouse needs the following configuration variable to be enabled:

- `CONFIG_OF_OVERLAY`;
- `CONFIG_KALLSYMS_ALL`;

You can enable them by giving this command inside the PetaLinux's project folder:

```bash
petalinux-config -c kernel
```

### Build the kernel

After the configuration, you can build the kernel as usual, with PetaLinux:

```bash
petalinux-build
```

## Building Jailhouse

Jailhouse can be easily built with the following command:

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- \
	KDIR=<path/to/petalinux_project>/build/tmp/work/xilinx_zcu104-xilinx-linux/linux-xlnx/6.1.30-xilinx-v2023.2+git999-r0/linux-xlnx-6.1.30-xilinx-v2023.2+git999 \
	DESTDIR=<absolute/path/to/your/rootfs> \
	install
```

> **NOTE**: If you are building stuff on a remote workstation, you have to find a suitable
way to update the file system of the SD card attached to your laptop
(i.e. you will spend time copying the *compressed* file system multiple times between your
laptop and the workstation).
**Alternatively**, you can set up on your laptop a `tftp` server and boot the board
in `tftpboot` mode (not covered in this guide).

### BONUS: How to copy the `rootfs` between your laptop and a remote workstation without breaking everything

Moving files every time between the workstation, your laptop and the SD card could be very
annoying and you could end with a **corrupted** file system.

The *safest* method that I've found is the following:

1. Copy the SD's `rootfs` in the remote workstation:

```bash
rsync -avz /path/to/rootfs <ws_user>@<ws_ip>:/home/<user>/
```

2. Build Jailhouse on the remote workstation;

3. Copy the file system back to your laptop (you have to use a non-root intermediate folder):

```bash
rsync -avzu <ws_user>@<ws_ip>:/path/to/rootfs /path/to/intermediate/folder
```

4. Copy only the updated file with `sudo` in the SD's file system:

```bash
sudo rsync -ruv /path/to/intermediate/folder/rootfs /path/to/mounted/sdcard/
```

> :warning: In order to copy correctly stuff, the destination path should not include the `rootfs` folder name!
Otherwise, you will copy the new `rootfs` inside the original one.

## Testing Jailhouse

> **NOTE**: for simplicity, you can copy the jailhouse folder with all the compiled cells
in the board's file system.

Here you can find an example of Jailhouse with `cache-coloring` and `memguard`:

> NOTE: be sure to enable UART2 prints with this
[option](boards/zcu104/linux#bonus-enable-output-from-the-uart2).

```bash
getty 115200 ttyPS1 &

modprobe jailhouse

jailhouse enable jailhouse/configs/arm64/zynqmp-zcu104-root-col.cell
jailhouse memguard 1 100000 1000 0
jailhouse cell create jailhouse/configs/arm64/zynqmp-zcu104-bomb0-col.cell
jailhouse cell load col-mem-bomb-0 jailhouse/inmates/demos/arm64/mem-bomb.bin
jailhouse cell start col-mem-bomb-0

membomb -c 1 -r -v -s 1048576 -e
membomb -c 1 -r -v -s 1048576
```
