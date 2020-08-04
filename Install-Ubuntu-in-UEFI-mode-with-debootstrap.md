---
title: Install Ubuntu in UEFI mode with debootstrap
date: 2017-11-03 15:28:19
tags: [Ubuntu, UEFI]
categories: Linux
aliases:
  - 2017/11/03/Install-Ubuntu-in-UEFI-mode-with-debootstrap/
---

在 UEFI 模式下想要安裝 Ubuntu 卻總是失敗：

![Ubuntu boot failed](https://i.imgur.com/E9j9j2p.png)

於是決定改採用全手動安裝的方式。

<!--more-->

## Installation

### Boot your PC/server from Ubuntu live CD

Boot your PC/server from Ubuntu live CD, configure network to connect to Internet.

### Prepare the partition

Create partitions on the system disk, the first one is for EFI system; the other is for system.

```bash
sudo parted /dev/sda
(parted) mklabel gpt
(parted) mkpart ESP fat32 1MiB 128MiB
(parted) mkpart primary xfs 128MiB 100%
(parted) set 1 boot on
(parted) quit

sudo partprobe /dev/sda
sudo mkfs.vfat -F 32 /dev/sda1
sudo mkfs.xfs -f /dev/sda2
```

Mount root file system:

```bash
sudo mount /dev/sda2 /mnt
```

### Install Ubuntu core with debootstrap

First, you need to install debootstrap:

```bash
sudo apt install debootstrap
```

Then, you can install ubuntu to the `/mnt` with `debootstrap` command:

```bash
sudo debootstrap --arch amd64 xenial /mnt http://tw.archive.ubuntu.com/ubuntu
```

Copy the apt repository configuration to new file system:

```bash
sudo cp /etc/apt/sources.list /mnt/etc/apt/sources.list
```


### Mount other file system

```bash
sudo mkdir -p /mnt/boot/efi
sudo mount /dev/sda1 /mnt/boot/efi
sudo mount --bind /dev /mnt/dev
sudo mount -t devpts /dev/pts /mnt/dev/pts
sudo mount -t proc proc /mnt/proc
sudo mount -t sysfs sysfs /mnt/sys
sudo mount -t tmpfs tmpfs /mnt/tmp
```

Then, chroot to `/mnt`:

```bash
sudo chroot /mnt /bin/bash
export HOME=/root
```

### Install Linux kernel, grub and other packages

```bash
apt update
apt install linux-image-generic linux-headers-generic grub-efi
```

### Configure grub

```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu --recheck --debug
update-grub
```

---
The reboot system.

## Reference

1. [GNU Parted - ArchWiki](https://wiki.archlinux.org/index.php/GNU_Parted#UEFI.2FGPT_examples)
2. [GRUB - ArchWiki](https://wiki.archlinux.org/index.php/GRUB_(正體中文)#UEFI.E7.B3.BB.E7.B5.B1.28UEFI_systems.29)
