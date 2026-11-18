# QEMU Direct Linux Boot

**QEMU** is an open-source machine emulator that can run virtual machines for many CPU architectures (x86, ARM, RISC-V, and others). It can emulate a complete system — CPU, memory, storage, and peripherals — entirely in software. For a detailed introduction to QEMU and virtualization concepts, see the [Primer on Virtualization](https://github.com/ManiAm/primer-virtualization) project.

The [previous chapter](13_kernel_boot.md) covered the kernel's internal boot sequence — from decompression through `start_kernel()` to the first userspace process. In that flow, a bootloader (GRUB, U-Boot, etc.) is responsible for loading the kernel into memory.

Normally, a QEMU virtual machine follows the same boot path as physical hardware: firmware → bootloader → kernel. QEMU's [**Direct Linux Boot**](https://www.qemu.org/docs/master/system/linuxboot.html) feature bypasses this entire chain by loading the kernel binary directly into the guest's memory, along with an optional initrd and kernel command-line parameters. This is useful for kernel development, testing, and debugging — it eliminates bootloader configuration and reduces the edit-compile-boot cycle to seconds.

The three QEMU options that enable direct boot are:

| Option     | Description                                                     |
|------------|-----------------------------------------------------------------|
| `-kernel`  | Path to the kernel image (e.g., `bzImage`)                      |
| `-initrd`  | Path to the initial RAM disk image                              |
| `-append`  | Kernel command-line parameters (e.g., `console=ttyS0 root=...`) |

## Obtaining a Kernel and initrd

Before using direct boot, you need two files: a **kernel image** and an **initrd**. On most Linux systems, both are readily available in `/boot/`:

```
$ ls /boot/vmlinuz-*
$ ls /boot/initrd.img-*      # Debian/Ubuntu
$ ls /boot/initramfs-*       # Fedora/RHEL
```

> **Note on naming:** Despite the `initrd.img` filename, Debian and Ubuntu have used the modern **initramfs** format (a cpio archive extracted into tmpfs) for over a decade. The `initrd` name is kept for backward compatibility with bootloader configurations. Fedora/RHEL use the more accurate `initramfs` filename. Both are the same format — see [Temporary Root Filesystem](13_kernel_boot.md#temporary-root-filesystem) for the full comparison.

Copy the matching pair to your working directory:

```
$ cp /boot/vmlinuz-$(uname -r) bzImage
$ cp /boot/initrd.img-$(uname -r) initrd.img    # adjust filename for your distro
```

> **Important:** The kernel and initrd must come from the same version. A mismatched initrd may fail to load the kernel modules it expects.

If you have compiled a custom kernel (see [Compiling for x86](09_compiling_x86.md), [ARM32](10_compiling_arm32.md), or [ARM64](11_compiling_arm64.md)), use the resulting image. In that case, you will also need to generate a matching initrd — see [Generating a Custom initrd with Dracut](#generating-a-custom-initrd-with-dracut) below.

## Install QEMU

On Debian/Ubuntu:

```
$ sudo apt install qemu-system-x86
```

For ARM64 emulation (used in the [last section](#direct-alpine-linux-boot-arm64)):

```
$ sudo apt install qemu-system-arm
```

> **Note:** On Debian/Ubuntu, the `qemu-system-arm` package provides both the `qemu-system-arm` (32-bit) and `qemu-system-aarch64` (64-bit) binaries. The examples in this chapter use `qemu-system-aarch64`.

## Booting a Kernel Without initrd

To see direct boot in action, start with just a kernel and no root filesystem:

```
$ qemu-system-x86_64 -kernel bzImage -append "console=ttyS0" -nographic
```

The `console=ttyS0` parameter tells the kernel to use the first serial port as its console. This is necessary when running with `-nographic` so that kernel output appears in the terminal instead of a graphical window.

Without an initrd or root filesystem, the kernel will panic after completing its initialization:

```
Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(8,0)
```

This is expected — the kernel booted successfully but could not find a root filesystem to mount. The next sections show two ways to solve this: booting into a temporary RAM disk, or mounting a persistent disk image.

## Generating a Custom initrd with Dracut

If you are using a pre-built kernel and initrd from `/boot/`, skip this section. If you compiled a custom kernel or need a minimal initrd for testing, you will need to generate your own.

> For an explanation of what an initrd/initramfs is and how it differs from the real root filesystem, see [Temporary Root Filesystem](13_kernel_boot.md#temporary-root-filesystem).

**Dracut** is a tool for generating initrd images, common on Red Hat-based distributions and available on Debian/Ubuntu:

```
$ sudo apt install dracut
```

Generate a minimal initrd that includes a bash shell:

```
$ sudo dracut --force --no-kernel --hostonly --no-hostonly-cmdline --add "bash" initrd.img
```

| Option                    | Description                                                                |
|---------------------------|----------------------------------------------------------------------------|
| `--force`                 | Overwrite an existing `initrd.img` if present                              |
| `--no-kernel`             | Exclude kernel modules — suitable when all required drivers are built into the kernel   |
| `--hostonly`              | Include only drivers needed for the current host hardware (produces a smaller image)  |
| `--no-hostonly-cmdline`   | Do not embed `/proc/cmdline` into the initrd — kernel parameters will be supplied via `-append` instead |
| `--add "bash"`            | Include the `bash` dracut module so an interactive shell is available inside the initrd |

To list all available dracut modules:

```
$ dracut --list-modules
```

To inspect the contents of the generated initrd:

```
$ lsinitrd initrd.img
```

## Booting with initrd

Whether you copied the initrd from `/boot/` or generated one with Dracut, you can pass it to QEMU alongside the kernel. This example boots into the initrd itself as a self-contained RAM environment — no disk image is needed:

```
$ qemu-system-x86_64 \
    -kernel bzImage \
    -initrd initrd.img \
    -append "console=ttyS0 rdinit=/bin/bash" \
    -nographic
```

The `rdinit=/bin/bash` parameter tells the kernel to run `/bin/bash` as the init process inside the initrd, instead of the default `/init` script. This drops you into an interactive shell within the RAM disk environment — useful for testing or inspecting the initrd contents.

Since this is a RAM disk with no real init system (no systemd, no OpenRC), standard shutdown commands are unavailable. To restart, use:

```
$ /sbin/reboot -f
```

> **Note:** The initrd is a temporary in-memory filesystem. Any files created or changes made are lost when the VM reboots.

## Booting with a Persistent Disk Image

The previous section booted into a temporary RAM environment. For persistent storage, attach a QCOW2 disk image containing a root filesystem. The kernel boots using the initrd for early initialization (loading drivers and mounting the disk), then pivots to the disk image as the permanent root filesystem.

### Step 1 — Prepare the root filesystem

Create a minimal root filesystem in `rootfs.qcow2` as described in [Minimal Root Filesystem](15_minimal_root_fs.md).

### Step 2 — Obtain an initrd

Use the host's initrd from `/boot/` if available:

```
$ cp /boot/initrd.img-$(uname -r) initrd.img
```

Alternatively, generate one with Dracut. The `--add "bash"` flag is not needed here — the root filesystem on disk provides the shell and init system:

```
$ sudo dracut --force --no-kernel --hostonly --no-hostonly-cmdline initrd.img
```

### Step 3 — Boot with the disk image

```
$ qemu-system-x86_64 \
    -kernel bzImage \
    -initrd initrd.img \
    -hda rootfs.qcow2 \
    -append "console=ttyS0 root=/dev/sda rw" \
    -nographic
```

| Parameter       | Description                                                    |
|-----------------|----------------------------------------------------------------|
| `-hda`          | Attach `rootfs.qcow2` as the primary hard disk (`/dev/sda`)   |
| `root=/dev/sda` | Tell the kernel the root filesystem is on the first hard disk  |
| `rw`            | Mount the root filesystem as read-write                        |

Notice that `rdinit=/bin/bash` is **not** used here. With `root=/dev/sda`, the initrd's default init scripts take over — they load the necessary drivers, mount `/dev/sda` as the root filesystem, and hand off control to the init process on disk.

### Step 4 — Verify

Once booted, confirm the root filesystem is mounted from the disk:

```
$ findmnt /
```

```
TARGET SOURCE   FSTYPE OPTIONS
/      /dev/sda ext4   rw,relatime
```

Verify the kernel command-line parameters:

```
$ cat /proc/cmdline
```

```
console=ttyS0 root=/dev/sda rw
```

Changes to the root filesystem now persist across reboots. To verify:

```
$ touch /test.txt
$ echo b > /proc/sysrq-trigger
```

The `echo b > /proc/sysrq-trigger` command sends a reboot request through the kernel's **SysRq** (System Request) mechanism — a set of low-level kernel commands accessible via `/proc/sysrq-trigger`. This is a quick way to reboot when no standard init system is available.

After the VM restarts, `/test.txt` will still be present.

## Direct Alpine Linux Boot (ARM64)

Direct boot also works for cross-architecture emulation. This example boots an Alpine Linux ARM64 system using pre-built kernel and initrd files — no firmware setup is required because the kernel is loaded directly by QEMU.

Download `vmlinuz-lts` and `initramfs-lts` from the [Alpine ARM64 netboot directory](https://dl-cdn.alpinelinux.org/alpine/edge/releases/aarch64/netboot/). For the disk image, prepare a QCOW2 image with an Alpine ARM64 root filesystem (see [Minimal Root Filesystem](15_minimal_root_fs.md) for the general approach):

```
$ qemu-system-aarch64 \
    -M virt \
    -cpu cortex-a57 \
    -m 512M \
    -hda alpine_arm.qcow2 \
    -kernel vmlinuz-lts \
    -initrd initramfs-lts \
    -append "console=ttyAMA0 ip=dhcp alpine_repo=http://dl-cdn.alpinelinux.org/alpine/edge/main/" \
    -nographic
```

Note that ARM uses `ttyAMA0` (the PL011 UART) as the serial console instead of x86's `ttyS0`.

> **Why no firmware?** Because `-kernel` loads the kernel directly into memory, no UEFI firmware is needed. Without direct boot, ARM64 VMs require explicit UEFI firmware (`-bios` or pflash) since QEMU does not bundle a default firmware for ARM — unlike x86, which includes SeaBIOS out of the box. For the full firmware-based ARM64 boot setup, see [QEMU System Emulation (ARM)](https://github.com/ManiAm/primer-virtualization/blob/master/docs/09_qemu_system_emulation_arm.md).

---

[← Previous: Kernel Boot Process](13_kernel_boot.md) | [Next: Minimal Root Filesystem →](15_minimal_root_fs.md)
