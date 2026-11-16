# Kernel Image

A Linux kernel image is a binary file containing the compiled kernel — the code, built-in device drivers, and data structures required to boot and manage the system. During boot, the bootloader loads this image into memory and transfers control to it.

The kernel build process starts by compiling the source into `vmlinux`, an uncompressed ELF (Executable and Linkable Format) binary. From `vmlinux`, architecture-specific tools produce a compressed, bootable image suited to the target platform and bootloader:

| Architecture | Kernel Image Format | Bootloader Support |
|--------------|---------------------|--------------------|
| x86 / x86_64 | `bzImage`           | GRUB, Syslinux, LILO |
| ARM (32-bit) | `zImage`, `uImage`  | U-Boot, GRUB, Barebox |
| ARM64        | `Image`, `Image.gz` | GRUB, U-Boot, EFI stub |
| PowerPC      | `zImage`, `uImage`  | GRUB, U-Boot, custom |
| MIPS         | `vmlinux`, `vmlinuz`, `uImage` | U-Boot, CFE, custom |
| RISC-V       | `Image`, `Image.gz` | U-Boot, GRUB |

Kernel image formats:

- **vmlinux** — uncompressed ELF binary; the base kernel output from which all other formats are derived. Also used for debugging and loaded directly on some architectures (e.g., MIPS).

- **bzImage** — compressed, bootable image for x86 systems. The name stands for "big zImage." Contains a self-extracting decompressor and is the standard format loaded by GRUB.

- **zImage** — compressed, self-extracting bootable image. The default format for ARM 32-bit and PowerPC.

- **uImage** — a kernel image wrapped with a U-Boot header containing load address and entry point metadata. Used by the U-Boot bootloader, mainly on embedded ARM, PowerPC, and MIPS systems.

- **Image / Image.gz** — raw flat binary or gzip-compressed kernel image for ARM64 and RISC-V. The bootloader handles decompression rather than the kernel itself.

## Bootloaders

A bootloader is a small program that runs before the operating system. Its job is to load the kernel image into memory, pass configuration parameters (such as the root filesystem and console device), and transfer control to the kernel. The most common Linux bootloaders are:

- **GRUB** (GRand Unified Bootloader) — the default bootloader on most desktop and server Linux distributions. Supports x86, ARM64, and PowerPC. Provides a menu to choose between multiple kernels or operating systems.

- **Syslinux / ISOLINUX** — lightweight bootloaders for x86 systems. Syslinux boots from FAT filesystems; ISOLINUX boots from CD/DVD ISO images.

- **U-Boot** (Das U-Boot) — the standard bootloader for embedded systems. Supports ARM, PowerPC, MIPS, and RISC-V. Reads `uImage` or `Image` formats and can load kernels from flash memory, network (TFTP), or storage devices.

- **Barebox** — a second-stage bootloader for embedded ARM systems, designed as a modern alternative to U-Boot.

- **EFI stub** — on UEFI systems, the kernel can be booted directly by the firmware without a separate bootloader, using the built-in EFI stub.

> For a deeper look at how bootloaders work, see the [Bootloaders](https://github.com/ManiAm/os-boot-process/blob/master/docs/03_bootloaders.md) chapter in the OS Boot Process project.

> For what happens after the bootloader hands control to the kernel, see [Kernel Boot Process](13_kernel_boot.md).

## Obtaining Kernel Images

There are three main ways to obtain a Linux kernel image:

### Compile from Source

You can compile the kernel from source code hosted at [kernel.org](https://www.kernel.org/), allowing full customization of features, drivers, and modules. For step-by-step instructions, see the next chapter.

The Linux kernel uses **Kbuild**, a custom build framework based on GNU Make. Kbuild uses Makefiles for build rules and Kconfig files for feature selection. It supports:

- **Interactive configuration** — tools like `menuconfig` or `xconfig` let you select which features, drivers, and modules to include
- **Incremental builds** — only recompiles files that have changed
- **Cross-compilation** — build for a different architecture than the host system
- **Module building** — compile drivers as loadable modules (`.ko` files) separate from the kernel image

For full Kbuild documentation, see: [docs.kernel.org/kbuild](https://docs.kernel.org/kbuild/index.html)

### Prebuilt Binaries from Package Managers

Linux distributions provide prebuilt kernel images through their package managers, ready to install without compiling:

**Ubuntu / Debian:**

```
$ sudo apt update
$ sudo apt install linux-image-<version>
```

**Fedora / CentOS / RHEL:**

```
$ sudo dnf install kernel-core
```

**Arch Linux:**

```
$ sudo pacman -S linux
```

After installation, the kernel and related files are placed under `/boot/`:

```
$ ls -l /boot/

-rw------- 1 root root  6261203 Nov 15 14:17 System.map-5.15.0-127-generic
-rw-r--r-- 1 root root   262519 Nov 15 14:17 config-5.15.0-127-generic
-rw-r--r-- 1 root root 27757982 Jan  7 20:13 initrd.img-5.15.0-127-generic
-rw------- 1 root root 11573416 Nov 15 14:34 vmlinuz-5.15.0-127-generic
```

- `vmlinuz-*` — the compressed kernel image (the `z` indicates compression)
- `System.map-*` — symbol table mapping kernel function names to memory addresses, used for debugging
- `config-*` — the kernel configuration file listing all options used to build this kernel
- `initrd.img-*` — initial RAM disk loaded alongside the kernel to provide early-boot drivers and mount the root filesystem

A kernel image alone is not a complete system. Building a full embedded Linux distribution — with a root filesystem, libraries, bootloader, and user-space utilities — is covered in the [Yocto Project chapter](12_compiling_yocto.md).

---

[← Previous: Kernel Source](07_kernel_source.md) | [Next: Compiling for x86 →](09_compiling_x86.md)
