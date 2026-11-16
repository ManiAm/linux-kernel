# Linux Kernel

Linux is a UNIX-like operating system kernel created by Linus Torvalds in 1991, originally developed on a MINIX system. Designed to be POSIX-compliant, it provides a UNIX-compatible programming interface while being distributed under the **GNU General Public License (GPL)**, allowing free usage, modification, and distribution. This open licensing model set Linux apart from proprietary UNIX systems.

Linux forms the foundation of numerous distributions such as Ubuntu, Debian, Fedora, and Red Hat Enterprise Linux (RHEL), and powers devices ranging from servers and supercomputers to smartphones and embedded systems.

## Kernel Version History

| Version    | Date           | Key Features |
|------------|----------------|--------------|
| Linux 0.01 | September 1991 | Initial prototype; process management, MINIX file system, terminal I/O. |
| Linux 0.12 | January 1992   | First GPL-licensed version; virtual file systems, TTY support. |
| Linux 1.0  | March 1994     | First stable release; TCP/IP networking, ext2 file system. |
| Linux 1.2  | March 1995     | Multi-architecture (Alpha, SPARC, MIPS); SCSI, NFS, loadable modules. |
| Linux 2.0  | June 1996      | SMP (Symmetric Multi-Processing); expanded architecture support. |
| Linux 2.2  | January 1999   | IPv6, USB support, improved networking and scalability. |
| Linux 2.4  | January 2001   | Journaling file systems (ext3), iptables firewall, improved USB. |
| Linux 2.6  | December 2003  | NPTL threading, NUMA, SELinux, O(1) scheduler. |
| Linux 3.0  | July 2011      | New versioning scheme; KVM virtualization, Btrfs improvements. |
| Linux 4.0  | April 2015     | Live kernel patching, improved ARM support. |
| Linux 5.0  | March 2019     | Energy-aware scheduling, Btrfs swap file support. |
| Linux 5.15 | November 2021  | NTFS3 driver, DAMON memory monitoring, realtime preemption. LTS. |
| Linux 6.0  | October 2022   | Network driver and filesystem performance improvements. |
| Linux 6.1  | December 2022  | Rust language support, Btrfs and networking improvements. LTS. |
| Linux 6.5  | August 2023    | USB4 v2, AMD P-State preferred cores, Wi-Fi 7 preparations. |
| Linux 6.6  | October 2023   | EEVDF scheduler (replaced CFS), Intel shadow stack. LTS. |
| Linux 6.7  | January 2024   | Bcachefs filesystem, Itanium (IA-64) architecture removed. |
| Linux 6.12 | November 2024  | PREEMPT_RT merged, sched_ext userspace scheduler extensions. LTS. |
| Linux 6.18 | November 2025  | Slub sheaves, TCP PSP encryption, BPF signed programs. LTS. |
| Linux 7.0  | April 2026     | Fsnotify I/O error API, XFS health monitoring, io_uring filters. |
| Linux 7.3  | October 2026   | Btrfs/Ext4 I/O improvements, cache-aware scheduling, AES APIs. |

## Version Numbering

Linux kernel versions follow a structured numbering scheme:

    major.minor.patch

**Major Version** — originally indicated significant architectural shifts (e.g., the jump from 1.x to 2.x introduced SMP support). Starting with 3.0 (2011), Linus Torvalds adopted a time-based model: the major number increments roughly every one to two years to keep version numbers manageable, regardless of whether the release contains architectural changes.

| Version | Years        | Key Developments |
|---------|--------------|------------------|
| 0.x     | 1991–1994    | Development prototypes; basic process management, MINIX file system, and i386 support. |
| 1.x     | 1994–1996    | First stable releases; TCP/IP networking, ext2 file system, and expanded hardware support. |
| 2.x     | 1996–2011    | SMP support, modular drivers, journaling file systems (ext3), improved threading, NUMA, and SELinux. |
| 3.x     | 2011–2015    | Time-based versioning adopted; improved scalability, ext4 enhancements, and better power management. |
| 4.x     | 2015–2019    | Live kernel patching, improved ARM support, and enhanced security features. |
| 5.x     | 2019–2022    | WireGuard VPN, io_uring for high-performance I/O, and expanded hardware support. |
| 6.x     | 2022–2026    | Rust language support, EEVDF scheduler, real-time (PREEMPT_RT) merge, and bcachefs. |
| 7.x     | 2026–present | Continued from 6.x (would have been 6.20); fsnotify API, cache-aware scheduling, and I/O improvements. |

**Minor Version** — reflects new features, drivers, or enhancements within a major series. For example, the jump from kernel 5.15 to 5.16 introduced new hardware support and performance improvements.

**Patch Version** — focuses on bug fixes, security updates, and small refinements. For example, kernel 6.12.1 contains fixes applied after the initial 6.12.0 release.

## Release Types

The Linux kernel follows a time-based release cycle of approximately 9–10 weeks:

1. A **2-week merge window** during which new features, drivers, and major changes are accepted into the mainline tree.
2. A **7–8 week stabilization period** with weekly release candidates (`-rc1` through `-rc7`) for bug fixes and regression testing.
3. The **final mainline release** (e.g., 7.3), after which the cycle repeats.

Kernel.org defines four release categories:

**Prepatch (RC)** — pre-release kernels produced during the stabilization phase. These weekly `-rc` snapshots are aimed at developers and testers, not production use.

**Mainline** — the official release that concludes each development cycle, maintained by Linus Torvalds. The mainline tree is where all new features and architectural changes are introduced. Each mainline release becomes the basis for a stable branch.

**Stable** — after each mainline kernel is released, it enters stable maintenance. The stable team (currently Greg Kroah-Hartman and Sasha Levin) backports bug fixes and security patches from the mainline tree, releasing updates as needed (e.g., 7.3.1, 7.3.2). A stable branch is typically maintained until the next mainline release, unless it is designated as longterm.

**Longterm (LTS)** — selected stable releases maintained for an extended period, starting with a 2-year projected end-of-life (EOL) that can be extended to 6 years or more based on industry demand. LTS kernels receive only important bug fixes and security updates, making them ideal for enterprise servers, embedded systems, and Android devices. Several LTS branches are maintained simultaneously.

Refer to the official [Linux kernel website](https://www.kernel.org/) for the latest mainline, stable, and longterm releases.

---

[← Previous: MINIX](02_minix.md) | [Next: Linux Distributions →](04_linux_distributions.md)
