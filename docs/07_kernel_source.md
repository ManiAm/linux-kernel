# Kernel Source Code

The Linux kernel is primarily written in C, with some portions in assembly language for low-level hardware interactions. You can obtain the kernel source code in two ways: as a tarball (a compressed archive) or from a Git repository.

## Tarball

The official Linux kernel website is [kernel.org](https://www.kernel.org/). Compressed tarballs of the source code are available at [kernel.org/pub/linux/kernel](https://www.kernel.org/pub/linux/kernel/). Each kernel version is provided in two compressed formats, along with a PGP signature file for verification:

```
linux-6.0.1.tar.gz      12-Oct-2022 07:54    204M
linux-6.0.1.tar.xz      12-Oct-2022 07:54    128M
linux-6.0.1.tar.sign    12-Oct-2022 07:54     987
```

- `.tar.gz` — gzip-compressed tarball. Faster to decompress but produces larger files.
- `.tar.xz` — xz-compressed tarball. Better compression ratio, resulting in smaller downloads.
- `.tar.sign` — PGP signature file used to verify the tarball's authenticity.

Most downloads use `.tar.xz` for its smaller size. The `.tar.gz` format is provided for compatibility with older systems that may not support xz.

Extract the tarball with one of the following commands:

```
$ tar -xzf linux-6.0.1.tar.gz     # extract gzip-compressed tarball
$ tar -xJf linux-6.0.1.tar.xz     # extract xz-compressed tarball
```

### Verifying the Download

The `.tar.sign` file signs the *uncompressed* `.tar` archive, not the compressed `.tar.xz` or `.tar.gz`. To verify, first import the signing key, then decompress and check the signature:

```
$ gpg --locate-keys torvalds@kernel.org              # import the signing key
$ xz -cd linux-6.0.1.tar.xz | gpg --verify linux-6.0.1.tar.sign -
```

A successful verification confirms that the tarball has not been tampered with and was signed by a trusted kernel maintainer.

## Git Repository

Tarballs are suited for one-time downloads or static builds. For active development, tracking changes, or contributing patches, the Git repository is preferred.

- The primary Git repository is hosted at: https://git.kernel.org/
- A read-only mirror is available on GitHub: https://github.com/torvalds/linux

Using the Git repository offers several advantages over tarballs:

- **Incremental updates** — pull only new changes instead of re-downloading the entire source
- **Full version history** — browse commit logs for debugging and tracking changes
- **Branching** — work on different versions or experimental features in parallel
- **Collaboration** — submit patches and participate in code reviews
- **CI/CD integration** — automate builds and testing against specific commits

Clone the repository:

```
$ git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

The full repository is several gigabytes. To save time and disk space, use a shallow clone that downloads only the latest commit:

```
$ git clone --depth 1 https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

Switch to a specific release by checking out its tag:

```
$ cd linux
$ git checkout v6.12
```

List all available release tags:

```
$ git tag
```

---

[← Previous: System Calls](06_system_calls.md) | [Next: Kernel Image →](08_kernel_image.md)
