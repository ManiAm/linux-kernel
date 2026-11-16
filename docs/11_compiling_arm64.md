# Compiling the Linux Kernel for ARM 64-bit

This chapter walks through cross-compiling the Linux kernel for ARM64 (AArch64) on an x86 machine using Docker.

Although ARM64 boards are more powerful than their 32-bit predecessors, compiling the kernel natively is still slow compared to running the same build on an x86 host with faster CPUs, more RAM, and SSD storage. Cross-compilation produces identical output — the compiler does the same work regardless of target architecture — so building on a powerful host and copying the result to the board is the standard workflow.

## Prepare Docker Container

Pull and run an Ubuntu 24.04 container:

```
$ docker pull ubuntu:24.04
$ docker run -d -it --name kernel-arm64 --hostname kernel-arm64 ubuntu:24.04
$ docker exec -it kernel-arm64 /bin/bash
```

Install the same base packages listed in [Compiling for x86](09_compiling_x86.md), plus the ARM 64-bit cross compiler:

```
$ apt update

$ DEBIAN_FRONTEND=noninteractive apt install -y \
    build-essential \
    bison \
    flex \
    libncurses-dev \
    libssl-dev \
    libelf-dev \
    bc \
    kmod \
    cpio \
    xz-utils \
    pkg-config \
    wget \
    git \
    file \
    python3 \
    gcc-aarch64-linux-gnu
```

The only new package is `gcc-aarch64-linux-gnu` — the ARM 64-bit cross compiler. See [Compiling for x86 – Prepare Docker Container](09_compiling_x86.md#prepare-docker-container) for what each base package does.

## Download Kernel Source Code

```
$ cd /usr/src
$ wget https://www.kernel.org/pub/linux/kernel/v6.x/linux-6.12.tar.xz
$ tar -xf linux-6.12.tar.xz
$ cd linux-6.12
```

## Configure the Kernel for ARM 64-bit

Set the cross-compilation environment variables:

```
$ export ARCH=arm64
$ export CROSS_COMPILE=aarch64-linux-gnu-
```

- `ARCH` tells the build system to use ARM64 Kconfig files, default configs, and boot image rules.
- `CROSS_COMPILE` is the prefix for the cross toolchain. The build system appends tool names (`gcc`, `ld`, `objcopy`, etc.) to this prefix — note the **trailing dash**.

The kernel includes a generic ARM64 default configuration:

```
arch/arm64/configs/defconfig
```

Generate the default config, then optionally customize:

```
$ make defconfig
$ make menuconfig
```

> For a detailed explanation of `.config`, see [Compiling for x86 – Configure the Kernel](09_compiling_x86.md#configure-the-kernel).

## Compile the Kernel

With `.config` in place, build using all available CPU cores:

```
$ make -j$(nproc)
```

The ARM64 build produces two image formats:

### Image

The uncompressed kernel binary:

```
$ file arch/arm64/boot/Image
arch/arm64/boot/Image: MS-DOS executable PE32+ executable (EFI application) Aarch64 ...
```

Despite the "MS-DOS executable" label, this is a Linux kernel. ARM64 `Image` includes an **EFI stub** — a small PE32+ header that allows UEFI firmware to load and execute the kernel directly, without a separate bootloader. This makes ARM64 kernels natively bootable on UEFI systems. Bootloaders such as U-Boot can also load `Image` directly.

### Image.gz

The gzip-compressed version of `Image`:

```
$ file arch/arm64/boot/Image.gz
arch/arm64/boot/Image.gz: gzip compressed data, max compression, from Unix, original size modulo 2^32 33286656
```

`Image.gz` reduces storage size, making it useful for embedded systems or devices with limited storage. The bootloader (or the kernel's EFI stub) decompresses it into memory before execution.

## How Image Is Created

Unlike x86's `bzImage` or ARM32's `zImage`, the ARM64 build pipeline does not produce a self-extracting compressed image. There is no decompression stub and no `arch/arm64/boot/compressed/` directory. The process is straightforward:

```
                              arch/arm64/boot/
                        ┌──────────────────────────────┐
                        │                              │
vmlinux ──OBJCOPY──────>│ Image ──GZIP──→ Image.gz     │
 (ELF)                  │ (raw binary +   (compressed) │
                        │  EFI stub)                   │
                        └──────────────────────────────┘
```

**1. Build vmlinux** — the full kernel is compiled and linked into `vmlinux`, an ELF binary containing all symbols and debug information.

**2. Convert to raw binary** — `objcopy` strips the ELF headers, symbols, and debug sections from `vmlinux` to produce `arch/arm64/boot/Image`. An EFI stub header is included so the image can be loaded by UEFI firmware.

**3. Compress (optional)** — `Image` is gzip-compressed to produce `Image.gz`.

> **Why no zImage for ARM64?** ARM64 was designed in an era where bootloaders (U-Boot, UEFI) are capable of handling decompression themselves. There is no need for a self-extracting wrapper — the bootloader loads either `Image` directly or decompresses `Image.gz` before transferring control to the kernel.

---

[← Previous: Compiling for ARM32](10_compiling_arm32.md) | [Next: Compiling with Yocto →](12_compiling_yocto.md)
