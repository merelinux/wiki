---
title: Using the raw disk image
weight: 2
---

There is a raw disk image located at
[https://pkgs.merelinux.org/images/](https://pkgs.merelinux.org/images/).

There are a lot of ways that this image could be used, but it works really well
as a starting point for a virtual machine. It has been successfully used as a
Digital Ocean droplet and an Oracle Cloud instance. It should be usable in any
cloud provider that supports MBR booting and the `VIRTIO` kernel drivers.

To use it as a VirtualBox disk, you could do the following, resizing the disk
to the size you wish to use.

```sh
VBoxManage convertfromraw meredisk.raw meredisk.vdi
VBoxManage modifymedium disk meredisk.vdi --resize <megabytes>
```

Then simply attach it to a new Linux virtual machine. In the machine settings,
under Network->Advanced, be sure to set the Adapter Type to
`Paravirtualized Network (virtio-net)`. When the image boots, it will attempt to
resize the root file system to fill the entire disk space available.

Unlike the ISO, the disk image has a default password set to `merepass`. Be sure
to change it if you deploy it anywhere.
