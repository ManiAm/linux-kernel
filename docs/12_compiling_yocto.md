# Compiling with the Yocto Project

## From Kernel to Complete System

In the previous chapters we compiled the Linux kernel for [x86](09_compiling_x86.md), [ARM32](10_compiling_arm32.md), and [ARM64](11_compiling_arm64.md). The result in each case was a single kernel image — `bzImage`, `zImage`, or `Image`. But a kernel alone is not a usable system. A real embedded product also needs a root filesystem, libraries, a bootloader, device-tree blobs, init system, user-space utilities, and often custom applications — all built for the target architecture and tested together.

Assembling these pieces by hand is tedious and error-prone. Two popular frameworks automate the entire process — from downloading source code to producing a bootable image:

| Feature                | Yocto Project | Buildroot |
|------------------------|---------------|-----------|
| **Purpose**            | Layered framework for building custom Linux distributions | Simple tool for building complete embedded Linux system images |
| **Complexity**         | High — steep learning curve with layers, recipes, and BitBake | Low — familiar Kconfig interface (`make menuconfig; make`) |
| **Configuration**      | Distributed across metadata layers, recipes, and config files | Single `.config` file edited via `menuconfig` (Kconfig) |
| **Build Time**         | Slow initial build; fast incremental builds via shared-state cache | Fast initial build; some config changes require full rebuilds |
| **Package Management** | Produces RPM, DEB, or IPK packages with optional runtime package manager | Produces fixed root filesystem images with no runtime package manager |
| **Target Use Case**    | Multi-variant product lines with long-term maintenance and compliance needs | Single-purpose devices with a fixed, well-defined software set |

The rest of this chapter focuses on the **Yocto Project**.

## Overview

The Yocto Project is an open-source collaboration that provides tools, templates, and methods for building custom Linux-based operating systems for embedded and IoT devices. It supports cross-compilation, reproducible builds, and package management, making it a standard choice for production embedded systems.

Official documentation: [docs.yoctoproject.org](https://docs.yoctoproject.org/index.html)

## Releases

The Yocto Project follows a time-based release schedule, issuing new versions approximately every six months. LTS releases receive extended maintenance and security updates, making them suitable for production environments. Regular releases adopt newer features but are supported for a shorter period.

| Version   | Codename  | Release Date | Support Level |
|-----------|-----------|--------------|---------------|
| 3.1 (LTS) | Dunfell   | April 2020   | EOL |
| 4.0 (LTS) | Kirkstone | April 2022   | EOL |
| 5.0 (LTS) | Scarthgap | April 2024   | Supported until April 2028 |
| 6.0 (LTS) | Wrynose   | May 2026     | Supported until April 2030 |
| 6.1       | Blacksail | October 2026 | Supported until April 2027 |

For the full release history, see the [Yocto Project Releases](https://wiki.yoctoproject.org/wiki/Releases) wiki page.

## Poky

Poky is the reference distribution of the Yocto Project. It provides the core build system and tools required to create a complete Linux image tailored for embedded systems.

Poky includes **BitBake**, the task scheduler and build engine used to compile source code and manage dependencies. BitBake is organized around **layers** (directories of metadata, prefixed `meta-`) and **recipes** (`.bb` files that describe how to fetch, configure, compile, and package individual software components). For a detailed introduction to BitBake, see the [BitBake chapter](https://github.com/ManiAm/primer-build-tools/blob/master/docs/25_bitbake.md).

### Project Structure

> **Note:** Prior to Wrynose (6.0), all Poky components were combined in a single [`poky`](https://git.yoctoproject.org/poky) Git repository. Starting with Wrynose, the components are maintained in three separate repositories and assembled automatically by `bitbake-setup`.

```
layers/
├── bitbake/                        ← git.openembedded.org/bitbake
├── openembedded-core/              ← git.openembedded.org/openembedded-core
│   ├── meta/
│   ├── meta-selftest/
│   └── scripts/
└── meta-yocto/                     ← git.yoctoproject.org/meta-yocto
    ├── meta-poky/
    ├── meta-yocto-bsp/
    └── meta-skeleton/
```

- **bitbake/** — the BitBake build engine. Analogous to Make but tailored for complex embedded Linux builds.

- **meta/** — the core metadata layer. Provides fundamental recipes and classes needed to build embedded Linux systems, including base recipes for packages like busybox, glibc, and systemd.

- **meta-poky/** — metadata for the Poky reference distribution. Extends the base metadata in `meta/` and provides default configurations, machine settings, and sample images.

- **meta-yocto-bsp/** — Board Support Packages (BSPs) for supported hardware platforms. Contains configurations and drivers for specific hardware targets (e.g., QEMU).

- **meta-skeleton/** — template recipes and layers for developers. Serves as an example structure for writing new recipes and creating custom layers.

- **meta-selftest/** — self-test cases for validating the Yocto Project build system.

- **scripts/** — helper scripts to simplify development tasks, automate environment setup, and generate patches.


## Building a Custom Image

This section walks through building and booting a minimal image using the **Wrynose** (6.0 LTS) release.

### Setting Up Docker

Pull and run an Ubuntu 24.04 container:

```
$ docker pull ubuntu:24.04
$ docker run -d -it --name yocto-build --hostname yocto-build ubuntu:24.04
$ docker exec -it yocto-build /bin/bash
```

### Install Dependencies

Install the host packages required by the Yocto build system:

```
$ apt update

$ DEBIAN_FRONTEND=noninteractive apt install -y \
    build-essential chrpath cpio debianutils diffstat file gawk gcc git \
    iputils-ping libacl1 libcrypt-dev locales python3 python3-git \
    python3-jinja2 python3-pexpect python3-pip python3-subunit \
    python3-venv socat texinfo unzip wget xz-utils zstd
```

BitBake requires a UTF-8 locale. Generate one to prevent encoding errors during the build:

```
$ locale-gen en_US.UTF-8
```

### Create a Build User

BitBake refuses to run as root for security and build-reproducibility reasons. Create a dedicated user and switch to it:

```
$ useradd -m -s /bin/bash yocto
$ su - yocto
```

All remaining commands in this chapter run as the `yocto` user.

### Set Up with bitbake-setup

Starting with Wrynose (6.0), the Yocto Project uses [`bitbake-setup`](https://docs.yoctoproject.org/bitbake/dev/bitbake-user-manual/bitbake-user-manual-environment-setup.html) to clone the required repositories and configure the build environment. Install it in a Python virtual environment:

```
$ cd ~
$ python3 -m venv --clear ./bitbake-setup-venv
$ source ./bitbake-setup-venv/bin/activate
$ pip install bitbake-setup
```

Initialize a Wrynose build targeting QEMU x86-64 with the Poky reference distribution:

```
$ bitbake-setup init --non-interactive poky-wrynose poky distro/poky machine/qemux86-64
```

This command clones BitBake, OpenEmbedded-Core, and meta-yocto into a new setup directory and generates the build configuration. The resulting layout is:

```
~/bitbake-builds/poky-wrynose/
├── build/          ← BitBake build directory (conf/, output)
├── config/         ← bitbake-setup configuration
└── layers/         ← cloned source repositories
```

### Initialize the Build Environment

```
$ source ~/bitbake-builds/poky-wrynose/build/init-build-env
Poky - The Yocto Project testing distribution
```

This sets environment variables and changes the working directory to `build/`. You need to run this command in every new terminal session before using BitBake.

The build directory contains two key configuration files:

- **`conf/local.conf`** — machine, distro, and build settings
- **`conf/bblayers.conf`** — the list of metadata layers included in the build

### Key Configuration Variables

The `bitbake-setup init` command already set the target machine (`qemux86-64`) and distribution (`poky`). You can review and adjust these in `conf/local.conf`.

**`MACHINE`** — specifies the target machine architecture for the build.

Built-in emulation targets (included in Poky):

- `qemux86` — 32-bit x86 emulator
- `qemux86-64` — 64-bit x86 emulator (selected during init)
- `qemuarm` — ARM emulator
- `qemuarm64` — ARM64 emulator

Hardware board targets (may require additional BSP layers):

- `beaglebone-yocto` — BeagleBone Black (included in `meta-yocto-bsp`)
- `raspberrypi4-64` — Raspberry Pi 4 64-bit (requires the [`meta-raspberrypi`](https://git.yoctoproject.org/meta-raspberrypi) layer)

**Root login** — disabled by default for security. For development and testing, enable the root-login configuration fragment:

```
$ bitbake-config-build enable-fragment core/yocto/root-login-with-empty-password
```

> **Warning:** Never enable this in production images. It allows anyone to log in as root without a password.

**`PACKAGE_CLASSES`** — defines the package format for built software packages.

- `package_rpm` — uses RPM packages; suitable for Red Hat-based systems (default)
- `package_deb` — uses DEB packages; compatible with Debian/Ubuntu-based systems
- `package_ipk` — uses IPK packages (lightweight); preferred for embedded systems

### Core Image Recipes

List all available recipes:

```
$ bitbake -s
```

The `core-image-*` recipes define pre-configured system images tailored for different purposes:

**Minimal Images**

| Recipe | Purpose |
|--------|---------|
| `core-image-minimal` | Very basic image with essential components for small embedded systems. |
| `core-image-minimal-dev` | Minimal image with additional development tools. |
| `core-image-minimal-initramfs` | Minimal image with an initramfs for RAM-based file systems. |

**Graphical Interface Images**

| Recipe | Purpose |
|--------|---------|
| `core-image-sato` | Sato graphical environment for embedded devices with touchscreen and GUI. |
| `core-image-weston` | GUI image with the Weston Wayland compositor. |
| `core-image-x11` | Graphical image based on the X11 Window System. |

**Development and Debugging Images**

| Recipe | Purpose |
|--------|---------|
| `core-image-full-cmdline` | Command-line based image with additional tools for debugging and development. |
| `core-image-kernel-dev` | Includes kernel development tools and headers. |

### Build and Boot

Build the minimal core image:

```
$ bitbake core-image-minimal
```

> **Note:** The first build downloads all source packages and compiles everything from scratch. Expect it to take one to several hours depending on your machine's CPU, RAM, and internet speed. Subsequent builds are incremental and complete much faster.

The output files are located in:

```
tmp/deploy/images/qemux86-64/
```

Key output files:

| File               | Description |
|--------------------|-------------|
| `bzImage-*.bin`    | Bootable Linux kernel in bzImage format. |
| `modules-*.tgz`    | Archive of kernel modules for dynamic loading. |
| `*.rootfs.ext4`    | Root filesystem in ext4 format. |
| `*.rootfs.tar.bz2` | Compressed tarball of the root filesystem. |
| `*.qemuboot.conf`  | Boot configuration for launching the image in QEMU. |

### Boot with QEMU

```
$ runqemu qemux86-64 slirp nographic
```

- `slirp` — enables user-mode networking, which does not require a TAP device or root privileges.
- `nographic` — redirects the virtual machine's serial console to the terminal, allowing you to interact with it without a graphical window.

The system boots to a login prompt:

```
Poky (Yocto Project Reference Distro) 6.0 qemux86-64 /dev/ttyS0
qemux86-64 login:
```

Log in with username `root` (no password required).

Verify the kernel version:

```
$ uname -a
Linux qemux86-64 6.18.54-yocto-standard #1 SMP PREEMPT_DYNAMIC ... x86_64 GNU/Linux
```

> **Note:** The kernel version (6.18 in Wrynose) is determined by the Yocto recipes, not by the kernel version used in the manual compilation chapters. Each Yocto release pins its own default kernel.

To terminate QEMU, press `Ctrl+A` followed by `X`.

---

[← Previous: Compiling for ARM64](11_compiling_arm64.md) | [Next: Kernel Boot Process →](13_kernel_boot.md)
