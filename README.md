# Linux Kernel

A reference covering the Linux kernel — its UNIX origins, system architecture overview, compiling custom kernels for multiple architectures, and the kernel boot process.

## Origins and History

| #  | Topic                                                  | Description |
|----|--------------------------------------------------------|-------------|
| 01 | [UNIX](docs/01_unix.md)                                | UNIX versions, timeline, BSD, System V, SunOS, HP-UX, AIX, and macOS |
| 02 | [MINIX](docs/02_minix.md)                              | MINIX versions, microkernel design, and influence on Linux |
| 03 | [Linux Kernel](docs/03_linux_kernel.md)                | Kernel version history, versioning scheme, and release types |
| 04 | [Linux Distributions](docs/04_linux_distributions.md)  | Distribution families, categories, code names, and distribution-specific kernels |

## Architecture and Internals

| #  | Topic                                                  | Description |
|----|--------------------------------------------------------|-------------|
| 05 | [Linux System](docs/05_linux_system.md)                | Kernel vs. user space, kernel subsystems, system calls, and device drivers |
| 06 | [System Calls](docs/06_system_calls.md)                | System call reference — process, IPC, memory, file, network, and security |

## Source Code and Image Formats

| #  | Topic                                                  | Description |
|----|--------------------------------------------------------|-------------|
| 07 | [Kernel Source](docs/07_kernel_source.md)              | Obtaining source code via tarballs and Git |
| 08 | [Kernel Image](docs/08_kernel_image.md)                | Image formats, bootloaders, obtaining prebuilt binaries, and embedded build frameworks |

## Building the Kernel

| #  | Topic                                                  | Description |
|----|--------------------------------------------------------|-------------|
| 09 | [Compiling for x86](docs/09_compiling_x86.md)          | Docker-based x86 kernel build, configuration, and bzImage creation |
| 10 | [Compiling for ARM32](docs/10_compiling_arm32.md)      | Cross-compilation for ARM 32-bit, zImage, and uImage |
| 11 | [Compiling for ARM64](docs/11_compiling_arm64.md)      | Cross-compilation for ARM64, Image, and Image.gz |
| 12 | [Compiling with Yocto](docs/12_compiling_yocto.md)     | Yocto Project, Poky, BitBake, and building custom embedded images |

## Boot Process

| #  | Topic                                                  | Description |
|----|--------------------------------------------------------|-------------|
| 13 | [Kernel Boot Process](docs/13_kernel_boot.md)          | Internal kernel boot sequence |
| 14 | [QEMU Direct Linux Boot](docs/14_direct_linux_boot.md) | Loading the kernel bypassing firmware and bootloader |

## Appendix

| Topic                                                      | Description |
|------------------------------------------------------------|-------------|
| [Minimal Root Filesystem](docs/minimal_root_fs.md)         | Mounting a QCOW2 image and populating it with a minimal root filesystem |
