# Kernel Boot Process

The [OS Boot Process](https://github.com/ManiAm/os-boot-process/blob/master/docs/01_os_boot_process.md) project covers the full boot chain from power-on through firmware (BIOS/UEFI) to the bootloader (GRUB, U-Boot, etc.). That chain ends at the point where the bootloader loads the Linux kernel into memory and transfers control to it.

This chapter picks up from that handoff. It follows the kernel's internal initialization — from the first instruction the kernel executes all the way to the launch of the first userspace process (PID 1).

## Overview

The kernel's internal boot sequence can be summarized as:

```
Bootloader
  → 1. Decompression stub (for compressed images)
    → 2. Architecture-specific entry (assembly)
      → 3. start_kernel() (C entry point)
          → 4. rest_init()
              → 5. kernel_init (PID 1) → /sbin/init
              →    kthreadd (PID 2)
```

Each stage is described in detail below.

## 1. Decompression

For compressed kernel images such as `bzImage` (x86) or `zImage` (ARM32), the bootloader does not jump directly to the kernel code. Instead, it jumps to a **decompression stub** — a small program prepended to the compressed kernel.

The decompression stub:

1. Executes from the address where the bootloader placed the image
2. Decompresses the kernel into a target memory location
3. Verifies the CRC-32 checksum embedded during the build
4. Jumps to the uncompressed kernel's entry point

The decompression stub source is located at:

| Architecture | Decompression Entry Point |
|--------------|--------------------------|
| x86 (64-bit) | `arch/x86/boot/compressed/head_64.S` |
| x86 (32-bit) | `arch/x86/boot/compressed/head_32.S` |
| ARM (32-bit) | `arch/arm/boot/compressed/head.S` |

ARM64 (`Image`) and uncompressed images skip this stage entirely — the bootloader jumps straight to the kernel entry point.

> For details on how compressed images are built, see [Compiling for x86](09_compiling_x86.md) and [Compiling for ARM32](10_compiling_arm32.md).

## 2. Architecture-Specific Entry

After decompression (or directly from the bootloader for uncompressed images), execution begins at an architecture-specific assembly entry point. This code performs the minimum hardware setup required before the kernel's C code can run.

| Architecture | Entry Point | Key File |
|--------------|-------------|----------|
| x86 (64-bit) | `startup_64` | `arch/x86/kernel/head_64.S` |
| x86 (32-bit) | `startup_32` | `arch/x86/kernel/head_32.S` |
| ARM (32-bit) | `stext` | `arch/arm/kernel/head.S` |
| ARM64 | `_head` / `primary_entry` | `arch/arm64/kernel/head.S` |

### What the Assembly Entry Code Does

The specifics vary by architecture, but the common tasks include:

**x86:**

- Verifies the CPU supports long mode (64-bit)
- Builds initial page tables for identity mapping
- Loads the Global Descriptor Table (GDT)
- Enables paging and switches to long mode
- Sets up the initial kernel stack
- Jumps to `x86_64_start_kernel()`, which calls `start_kernel()`

**ARM32:**

- Validates the processor type and machine ID
- Creates initial page tables
- Enables the MMU
- Sets up the initial stack
- Calls `start_kernel()`

**ARM64:**

- Reads the processor ID and validates it
- Configures the MMU and creates initial page tables
- Enables the MMU
- Sets up the initial stack pointer
- Jumps to `start_kernel()`

The assembly entry code is intentionally minimal. Its sole purpose is to establish the environment (paging, stack, basic CPU state) needed to execute C code.

## 3. init/main.c: `start_kernel()`

`start_kernel()` in `init/main.c` is the **main C entry point** of the Linux kernel. It is architecture-independent and called by every platform's assembly entry code. This function performs the bulk of the kernel's initialization by calling subsystem init functions in a specific order.

Source: https://github.com/torvalds/linux/blob/v6.12/init/main.c

The major initialization steps, in order:

### Early Setup

| Function | Purpose |
|----------|---------|
| `set_task_stack_end_magic()` | Marks the end of the init task's stack for overflow detection |
| `boot_cpu_init()` | Marks the boot CPU as online and active |
| `setup_arch()` | Architecture-specific setup — memory layout, CPU features, device tree parsing |
| `setup_command_line()` | Saves the kernel command line passed by the bootloader (e.g., `root=/dev/sda1 console=ttyS0`) |
| `setup_per_cpu_areas()` | Allocates per-CPU data areas for SMP systems |

`setup_arch()` is the most complex of these. It is architecture-specific and handles:

- Parsing the memory map from the bootloader or firmware (e.g., E820 on x86, device tree on ARM)
- Reserving memory regions (kernel image, initrd, device tree)
- Detecting CPU capabilities and errata
- Setting up the initial memory zones

### Core Subsystems

| Function | Purpose |
|----------|---------|
| `trap_init()` | Sets up CPU exception and fault handlers |
| `mm_core_init()` | Initializes the memory management subsystem — buddy allocator, slab allocator, vmalloc |
| `sched_init()` | Initializes the process scheduler and creates the idle task for each CPU |
| `init_IRQ()` | Configures the interrupt controller and interrupt descriptors |
| `time_init()` | Initializes timekeeping — clock sources, timers, and tick infrastructure |
| `softirq_init()` | Sets up software interrupt (softirq) handling |
| `console_init()` | Initializes the early console for `printk` output |

### Remaining Initialization

| Function | Purpose |
|----------|---------|
| `vfs_caches_init()` | Initializes the Virtual File System — dentry cache, inode cache, and mount infrastructure |
| `signals_init()` | Sets up signal handling data structures |
| `proc_root_init()` | Creates the `/proc` filesystem |
| `cpuset_init()` | Initializes CPU and memory set (cgroup) infrastructure |
| `rest_init()` | Creates the first kernel threads and completes the boot |

By the end of `start_kernel()`, all core subsystems are initialized and ready. The final call to `rest_init()` transitions the system from a single-threaded boot sequence into a multithreaded environment.

## 4. init/main.c: `rest_init()` and Kernel Threads

`rest_init()` is the last function called by `start_kernel()`. It creates the kernel's first threads:

| Thread | PID | Role |
|--------|-----|------|
| `kernel_init` | 1 | Becomes the first userspace process (`/sbin/init`) |
| `kthreadd` | 2 | Kernel thread daemon — parent of all other kernel threads |

After spawning these threads, the boot CPU enters the **idle loop** (`cpu_startup_entry()`), where it waits for the scheduler to assign work.

## 5. init/main.c: `kernel_init` — From Kernel to Userspace

The `kernel_init` thread (PID 1) is responsible for the final steps of the boot process. It bridges the gap between kernel initialization and userspace.

### Temporary Root Filesystem

Before the kernel can mount the real root filesystem (e.g., an ext4 partition on disk), it often needs drivers and tools that are not compiled into the kernel itself — storage controller drivers, filesystem drivers, or utilities for LVM, RAID, or disk encryption. These components live on the very disk the kernel cannot yet access. A temporary root filesystem in RAM solves this chicken-and-egg problem by providing just enough userspace to locate and mount the real root.

There are two mechanisms for this. The older **initrd** and the modern **initramfs** both serve the same purpose but work differently:

| | initrd (legacy) | initramfs (modern) |
|---|---|---|
| **Format** | Filesystem image (e.g., ext2) loaded into a RAM-backed block device | Compressed cpio archive extracted into tmpfs |
| **Kernel requirement** | Needs a built-in driver for the image's filesystem format | No filesystem driver needed — the kernel has a built-in cpio extractor |
| **Storage** | Separate file loaded by the bootloader alongside the kernel | Can be embedded in the kernel image or loaded as a separate file |
| **Init program** | Runs `/linuxrc`, then returns to the kernel to mount the real root | Runs `/init`, which is responsible for mounting the real root and calling `switch_root` |
| **Root switch** | Kernel performs `pivot_root` and unmounts the ramdisk | Userspace calls `switch_root` (or `pivot_root`) to replace the tmpfs |
| **Memory** | Fixed-size ramdisk — wastes memory if oversized, fails if undersized | tmpfs grows and shrinks dynamically to fit its contents |

Modern Linux distributions use **initramfs** exclusively. The term "initrd" is still widely used in documentation and boot configurations (e.g., GRUB's `initrd` command), but the actual file is almost always an initramfs cpio archive.

The temporary root filesystem typically provides:

- Kernel modules for storage, filesystems, and other hardware not built into the kernel
- The `/init` script or binary that probes devices, assembles volumes, and mounts the real root
- Utilities for device discovery, RAID assembly, LVM activation, or encrypted volume unlocking

> For a step-by-step walkthrough of how the initramfs mounts the real root partition, hands off control via `switch_root`, and how the init system then mounts the remaining filesystems from `/etc/fstab`, see [Mounting](https://github.com/ManiAm/storage-and-file-systems/blob/master/docs/15_mounting.md).

### do_initcalls()

`kernel_init` calls `do_basic_setup()`, which in turn calls `do_initcalls()`. This function iterates through all compiled-in initialization functions registered with the `*_initcall()` macros, organized into levels:

| Level | Macro | Typical Users |
|-------|-------|--------------|
| early | `early_initcall` | CPU and memory early setup (runs before SMP) |
| 0 | `pure_initcall` | Minimal infrastructure with no dependencies |
| 1 | `core_initcall` | Core kernel subsystems |
| 2 | `postcore_initcall` | Bus and class registration |
| 3 | `arch_initcall` | Architecture-specific init |
| 4 | `subsys_initcall` | Subsystem init (PCI, USB, SCSI buses) |
| 5 | `fs_initcall` | Filesystem registration |
| 6 | `device_initcall` | Most device drivers (default for `module_init`) |
| 7 | `late_initcall` | Late-stage initialization |

`early_initcall` functions run separately before SMP initialization via `do_pre_smp_initcalls()`. Levels 0 through 7 are processed in order by `do_initcalls()`. This is when built-in device drivers are initialized and hardware is probed.

### Transitioning to Userspace

After all initialization is complete, `kernel_init`:

1. Frees memory used only during initialization (marked with `__init`)
2. Attempts to execute the init program, trying these paths in order:
   - `/init` inside the initramfs (overridden by the `rdinit=` kernel parameter)
   - The path specified by the `init=` kernel command-line parameter
   - The compile-time `CONFIG_DEFAULT_INIT` path (if configured)
   - `/sbin/init`
   - `/etc/init`
   - `/bin/init`
   - `/bin/sh` (as a last resort)

If none of these are found, the kernel panics with:

    No working init found.

When an initramfs is present, its `/init` typically runs first — it loads drivers, mounts the real root filesystem, and then calls `switch_root` to hand control to the real init program (e.g., `systemd` or `sysvinit`).

Once the init program starts, the kernel's boot process is complete. PID 1 (now running in userspace) takes over and manages the rest of the system startup — mounting filesystems, starting services, and presenting a login prompt.

## Kernel Boot Messages

The kernel logs every stage of the boot process through `printk`. These messages can be viewed after boot with:

    $ dmesg

Example output (abbreviated):

    [    0.000000] Linux version 6.12.0 (root@builder) (gcc 9.4.0) #1 SMP ...
    [    0.000000] Command line: root=/dev/sda1 console=ttyS0
    [    0.000000] x86/fpu: x87 FPU on board
    [    0.004000] Memory: 2048MB available
    [    0.010000] CPU: Intel Core i7 ...
    [    0.100000] pid_max: default: 32768 minimum: 301
    [    0.200000] Mount-cache hash table entries: 4096
    [    0.300000] smpboot: CPU0: Intel Core i7 stepping 2
    [    0.500000] smp: Brought up 1 node, 4 CPUs
    [    1.000000] NET: Registered PF_INET protocol family
    [    1.200000] EXT4-fs (sda1): mounted filesystem with ordered data mode
    [    1.500000] Run /sbin/init as init process

---

[← Previous: Compiling with Yocto](12_compiling_yocto.md) | [Next: Direct Linux Boot →](14_direct_linux_boot.md)
