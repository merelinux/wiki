---
title: Using the ISO image
weight: 0
---

The current images can be found at
[https://pkgs.merelinux.org/images/](https://pkgs.merelinux.org/images/).

The Mere Linux ISO runs in a `tmpfs`, meaning you can change its contents like a
real system during your session, including adding or upgrading packages.
However, since it is all happening in memory, those changes will not persist
through a reboot.

The ISO will by default attempt to start a network device and retrieve an IP
address via DHCP. You can check that it has by running `ifconfig eth0`. A
working internet connection will be necessary in order to install any packages.

If you don't see your network device, probably the kernel module for your device
has not been built and included. Although we intend to support more hardware,
we are currently adding those in slowly. In this case, please run `lspci` and
add the output to a
[new Github issue](https://github.com/jhuntwork/merelinux/issues).

The ISO also includes the `dropbear` package to provide ssh access to the
`tmpfs` system, if you should wish. To use it, set a root password, and start
the service.

```sh
passwd
service start dropbear
```

## Set up a disk for legacy booting

Here is a _sample_ way to install Mere Linux from the running ISO. We'll assume
that the physical disk is called `/dev/sda`. Alternatively, if you want to use
EFI booting, skip to
[Set up a disk for EFI booting](#set-up-a-disk-for-efi-booting).

First, create a single partition, filling the entire disk. *Note: This will
erase the contents of the disk*

```sh
sgdisk -Z /dev/sda
sgdisk -N=1 -A 1:set:2 /dev/sda
```

Inspect the details of the created partition, and note the Partition unique
GUID.

```sh
sgdisk -i=1
# Grab the unique GUID - we'll need this later
partuuid=$(sgdisk -i=1 /dev/sda 2>&1 | grep unique | awk '{print $NF}')
```

Now format the new partition. Because we are using a single partition in this
example, we have to turn off ext4's 64bit support. You can avoid this by using
a separate boot partition.

```sh
mkfs.ext4 -O ^64bit /dev/sda1
```

Mount the new file system:

```sh
mount /dev/sda1 /mnt
```

Now skip ahead to [Installing the base system](#installing-the-base-system).

## Set up a disk for EFI booting

Assuming our disk is called `/dev/sda`, set up two partitions, one for for the
EFI boot partition, and one for the root partition. *Note: This will erase the
contents of the disk*

```sh
sgdisk -Z /dev/sda
sgdisk -n=1:0:100M /dev/sda
sgdisk -N=2 /dev/sda
```

Inspect the details of the created root partition, and note the Partition unique
GUID.

```sh
sgdisk -i=2
# Grab the unique GUID - we'll need this later
partuuid=$(sgdisk -i=2 /dev/sda 2>&1 | grep unique | awk '{print $NF}')
```

Format the partitions.

```sh
mkfs.vfat /dev/sda1
mkfs.ext4 /dev/sda2
```

Mount the partitions.

```sh
mount /dev/sda2 /mnt
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
```

## Installing the base system

First create the database directory `pacman` needs to run, then install the
base-layout and `busybox` packages. Although base-layout and `busybox` are both
part of the base group, we install them first separately because the other
packages need those to be present to correctly run some of their install hook
scripts.

```sh
mkdir -p /mnt/var/lib/pacman
pacman -Sy -r /mnt -b /mnt/var/lib/pacman base-layout busybox
```

Now install the rest of the packages in the base group. When prompted, choose
the default selection of 'all'.

```sh
pacman -Sy -r /mnt -b /mnt/var/lib/pacman --needed base
```

Change the root password in the installed system.

```sh
chroot /mnt passwd
```

Set the hostname for the system.

```sh
echo 'myhostname' >/mnt/etc/hostname
```

Tell the initial tty screen to clear itself:

```sh
clear >/mnt/etc/issue
```

Install a basic networking configuration.

```sh
cat >/mnt/etc/network/interfaces << EOF
auto lo eth0

iface lo inet loopback

iface eth0 inet dhcp
EOF

echo '127.0.0.1 localhost' >/mnt/etc/hosts
```

Add an initial `/etc/fstab` file.

```sh
cat > /mnt/etc/fstab << EOF
devpts /dev/pts devpts defaults 0 0
tmpfs  /dev/shm tmpfs  defaults 0 0
EOF
```

## Configure the boot loader for a legacy boot

If you set up your disks for EFI boot, skip to
[Configure the boot loader for an EFI boot](#configure-the-boot-loader-for-a-legacy-boot).

Create the configuration. *Make sure the `partuuid` variable is set to the GUID
of the filesystem we created earlier*.

```sh
mkdir -p /mnt/boot/extlinux
cat >/mnt/boot/extlinux/extlinux.conf <<EOF
DEFAULT mere
PROMPT 1
TIMEOUT 30

LABEL mere
  LINUX /boot/vmlinux
  APPEND root=PARTUUID=${partuuid} quiet
EOF
```

Install the bootloader and prepare the MBR.

```sh
extlinux -i /mnt/boot/extlinux
cat /usr/share/syslinux/gptmbr.bin >/dev/sda
```

At this point you should be able to unmount the volume and restart the machine.

```sh
umount /mnt
reboot
```

## Configure the boot loader for an EFI boot

Create the EFI directory and install `syslinux` to it

```sh
mkdir -p /mnt/boot/EFI/BOOT
cp /usr/share/syslinux/efi64/syslinux.efi /mnt/boot/EFI/BOOT/bootx64.efi
cp /usr/share/syslinux/efi64/ldlinux.e64 /mnt/boot/EFI/BOOT/
```

Create the `syslinux` configuration. *Make sure the `partuuid` variable is set
to the GUID of the filesystem we created earlier*.

```sh
cat >/mnt/boot/EFI/BOOT/syslinux.cfg << EOF
DEFAULT mere
PROMPT 1
TIMEOUT 30

LABEL mere
  LINUX ../../vmlinux
  APPEND root=PARTUUID=${partuuid} quiet
EOF
```

At this point you should be able to unmount the volume and restart the machine.

```sh
umount /mnt/boot
umount /mnt
reboot
```
