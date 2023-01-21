---
title: Using an existing Linux system
weight: 1
---

Assuming you already have a running Linux system with a spare disk or partition,
you can easily grab Mere's `pacman` package and install Mere to it. Setting up
the partitions and boot loader is left as an exercise to the reader, but you can
use the section
[Using the ISO Image](../iso/) as a generic
guide. Note that Mere's kernel currently only supports ext2,3,4 filesystems, but
we are open to adding support for more as we mature. If you would like to use a
different file system, please create a
[new Github issue](https://github.com/jhuntwork/merelinux/issues).

First, download `pacman`.

```sh
curl -LO https://pkgs.merelinux.org/core/pacman-latest-x86_64.pkg.tar.xz
```

You can validate the file using the `pacman-latest.SHA512SUM` file.

```sh
curl -LO https://pkgs.merelinux.org/core/pacman-latest.SHA512SUM
sha512sum -c pacman-latest.SHA512SUM
```

Extract it to a temporary location.

```sh
mkdir -p /tmp/pacman
tar -C /tmp/pacman -xf pacman-latest-x86_64.pkg.tar.xz
```

Now you can use
`/tmp/pacman/usr/bin/pacman --config /tmp/pacman/etc/pacman.conf` to install the
system in the same way that is described under
[Installing the base system](../iso#installing-the-base-system). Simply replace every
usage of `pacman` with
`/tmp/pacman/usr/bin/pacman --config /tmp/pacman/etc/pacman.conf`.

If you want to use Mere's `syslinux` package to boot the system, install that
package as well, either to a temporary location on the host system, or into the
destination system, and use one of the boot methods described in
[Configure the boot loader for a legacy boot](../iso#configure-the-boot-loader-for-a-legacy-boot)
or
[Configure the boot-loader for an EFI boot](../iso#configure-the-boot-loader-for-an-efi-boot).
