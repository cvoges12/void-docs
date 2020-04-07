# Installation via chroot (x86 or x86_64)

This guide details the process of manually installing Void Linux (the chroot
method) on an x86 or x86_64 PC architecture. It is assumed that you have a
familiarity with Linux but not necessarily that you have performed a chroot
installation before. If you are unfamiliar with the process, this guide can be
followed literally to create a "typical" setup (one partition is used on one
disk that is connected by SATA, IDE or USB). If you are comfortable with the
process, this guide should serve as an outline for reference, where each step
may be modified to create less typical setups (such as [full disk
encryption](./fde.md)).

Void Linux provides two options for bootstrapping the new installation.
Following the **XBPS method**, the [XBPS Package Manager](../../xbps/index.md)
running on the host operating system installs the base distribution. Following
the **ROOTFS method**, the base operating system is installed by unpacking a
ROOTFS tarball available to download.

The **XBPS method** requires that the host operating system have XBPS installed.
This may be an already working installation of Void Linux, an official [live
image](../live-images/prep.md) or a [statically linked
XBPS](../../xbps/troubleshooting/static.md) running on any working Linux
installation.

The **ROOTFS method** requires only a host operating system that can enter a
Linux chroot and has [tar(1)](https://man.voidlinux.org/tar.1) and
[xz(1)](https://man.voidlinux.org/xz.1) installed. This method may be preferable
if you wish to install Void Linux using a different Linux distribution.

## Prepare Filesystems

[Partition your disks](../live-images/partitions.md) and format them using
[mke2fs(8)](https://man.voidlinux.org/mke2fs.8),
[mkfs.xfs(8)](https://man.voidlinux.org/mkfs.xfs.8),
[mkfs.btrfs(8)](https://man.voidlinux.org/mkfs.btrfs.8) or whatever tools are
necessary for your filesystem(s) of choice.

[mkfs.vfat(8)](https://man.voidlinux.org/mkfs.vfat.8) is also available to
create FAT32 partitions. However, due to restrictions associated with FAT
filesystems, it should only be used when no other filesystem is suitable (such
as the EFI System Partition).

[cfdisk(8)](https://man.voidlinux.org/cfdisk.8) and
[fdisk(8)](https://man.voidlinux.org/fdisk.8) are available on the live images
for partitioning, but you may wish to use
[gdisk(8)](https://man.voidlinux.org/gdisk.8) (from the package `gptfdisk`) or
[parted(8)](https://man.voidlinux.org/parted.8) instead.

For a UEFI booting system, make sure to create an EFI System Partition (ESP).
The ESP should have the partition type "EFI System" (code `EF00`) and be
formatted as fat32 using [mkfs.vfat(8)](https://man.voidlinux.org/mkfs.vfat.8).

If unsure what partitions to create, create (1) 1GB in size with type `EFI
System` (code `EF00`) (2) remainder of the drive with type `Linux Filesystem`
(code `8300`).

Then format these partitions with fat32 and ext4 respectively.

```
# mkfs.vfat /dev/sda1
# mkfs.ext4 /dev/sda2
```

### Create a New Root and Mount Filesystems

This guide will assume the new root filesystem is mounted on `/mnt`. You may
wish to mount it elsewhere.

If using UEFI, mount the EFI System Partition as `/mnt/boot/efi`.

For example, if `/dev/sda2` is to be mounted as `/` and `dev/sda1` is the EFI
System Partition

```
# mount /dev/sda2 /mnt/
# mkdir -p /mnt/boot/efi/
# mount /dev/sda1 /mnt/boot/efi/
```

Initialize swap space, if desired, using
[mkswap(8)](https://man.voidlinux.org/mkswap.8).

## Base Installation

This step is separated into the XBPS method and the ROOTFS method. Follow only
one of the two following subsections.

### The XBPS Method

[Select an XBPS mirror.](../../xbps/repositories/mirrors/index.md) For
simplicity, save this URL to a shell variable for later use. Append `/current`
to this repository url to install a glibc distribution, or `/current/musl` for a
musl distribution.

> Note: musl binaries are not available for i686.

For example, this is the url for glibc from the Tier 1 server in Finland,

```
# REPO=https://alpha.de.repo.voidlinux.org/current
```

XBPS also needs to know what architecture is being installed, available options
are `x86_64`, `x86_64-musl` and `i686` for PC architecture computers.

For example,

```
# ARCH=x86_64
```

This architecture must be compatible with your current operating system, but
does not need to be the same. If your host is running an x86_64 operating
system, any of the three architectures can be installed (whether the host is
musl or glibc), but an i686 host can only install i686 distributions.

Use [xbps-install](https://man.voidlinux.org/xbps-install.1) to boot strap the
installation by installing the `base-system` metapackage.

```
# XBPS_ARCH=$ARCH xbps-install -S -r /mnt -R "$REPO" base-system
```

### The ROOTFS Method

[Download a ROOTFS
tarball](https://voidlinux.org/download/#download-installable-base-live-images-and-rootfs-tarballs)
matching your architecture.

Unpack the tarball into the newly configured filesystems.

```
# tar xvf void-[...]-ROOTFS.tar.xz -C /mnt
```

## Configuration

> Note: With the exception of the section "Install base-system (ROOTFS method
> only) the remainder of this guide is common to both XBPS and ROOTFS
> installation methods.

### Entering the Chroot

Mount the pseudo-filesystems needed for a chroot,

```
# mount --rbind /sys /mnt/sys && mount --make-rslave /mnt/sys
# mount --rbind /dev /mnt/dev && mount --make-rslave /mnt/dev
# mount --rbind /proc /mnt/proc && mount --make-rslave /mnt/proc
```

Copy the DNS configuration into the new root so that XBPS can still download new
packages inside the chroot,

```
# cp /etc/resolv.conf /mnt/etc/
```

Chroot into the new installation

```
# PS1='(chroot) # ' chroot /mnt/ /bin/bash
```

### Install base-system (ROOTFS method only)

> Note: This section can be skipped if following the XBPS method. You already
> have base-system installed.

ROOTFS images generally contain out of date software, due to being a snapshot of
the time when they were built, and do not come with a complete `base-system`.
Update the package manager and install `base-system`.

```
# xbps-install -Su xbps
# xbps-install -u
# xbps-install base-system
# xbps-remove base-voidstrap
```

### Installation Configuration

Write the hostname to `/etc/hostname`. Go through the options in
[`/etc/rc.conf`](../../config/rc-files.md#rcconf). If installing a glibc
distribution, edit `/etc/default/libc-locales`, uncommenting desired
[locales](../../config/locales.md).

[nvi(1)](https://man.voidlinux.org/nvi.1) is available in the chroot, but you
may wish to install your preferred text editor at this time.

For glibc builds, generate locale files with

```
(chroot) # xbps-reconfigure -f glibc-locales
```

### Set a Root Password

[Configure at least one super user account.](../../config/users-and-groups.md)
Other user accounts can be configured later, but there should either be a root
password, or a new user account with [sudo(8)](https://man.voidlinux.org/sudo.8)
privileges.

To set a root password, run

```
(chroot) # passwd
```

### Configure fstab

Configure [fstab(5)](https://man.voidlinux.org/fstab.5). This file can be
automatically generated from currently mounted filesystems by copying the file
`/proc/mounts`.

```
(chroot) # cp /proc/mounts /etc/fstab
```

Remove lines in `/etc/fstab` that refer to `proc`, `sys`, `devtmpfs` and `pts`.

Replace references to `/dev/sdXX`, `/dev/nvmeXnYpZ`, etc. with their respective
UUID, which can be found by found by running
[blkid](https://man.voidlinux.org/blkid.8). Referring to filesystems by their
UUID guarantees they will be found even if they are assigned a different name at
a later time. In some situations, such as booting from USB, this is absolutely
essential. In other situations, such as SATA or NVME, drives will usually always
have the same name, unless drives are physically added or removed. Therefore,
this step may not be strictly necessary but is almost always recommended.

Change the last zero of the entry for `/` to `1`, and the last zero of every
other line to `2`. These values configure the behaviour of
[fsck(8)](https://man.voidlinux.org/fsck.8).

For example, the partition scheme used throughout previous examples yields the
following fstab:

```
/dev/sda1       /boot/EFI   vfat    rw,relatime,[...]       0 0
/dev/sda2       /           ext4    rw,relatime             0 0
```

The fstab is modified thus with information from blkid:

```
UUID=6914[...]  /boot/EFI   vfat    rw,relatime,[...]       0 2
UUID=dc1b[...]  /           ext4    rw,relatime             0 1
```

> Note: The output of `/proc/mounts` will have a single space between each
> field. The columns are aligned here for readability.

Add an entry to mount `/tmp` in ram:

```
tmpfs           /tmp        tmpfs   defaults,nosuid,nodev   0 0
```

If using swap space, add an entry for any swap partitions:

```
UUID=1cb4[...]  swap        swap    rw,noatime,discard      0 0
```

## Installing GRUB

Use
[grub-install](https://www.gnu.org/software/grub/manual/grub/html_node/Installing-GRUB-using-grub_002dinstall.html)
to install grub onto your boot disk.

**On a BIOS computer**, install the package `grub`, then run `grub-install
/dev/sdX` where `/dev/sdX` is the drive (not partition) that you wish to install
GRUB to.

```
(chroot) # xbps-install grub
(chroot) # grub-install /dev/sda
```

**On a UEFI computer**, install either `grub-x86_64-efi` or `grub-i386-efi`,
depending on your architecture then run `grub-install`, optionally specifying a
bootloader label (this label may be used by your computer's firmware when
manually selecting a boot device)

```
(chroot) # xbps-install grub-x86_64-efi
(chroot) # grub-install --bootloader-id="Void"
```

If installing onto a removable disk (such as USB) add the option `--removable`
to the `grub-install` command.

## Finalization

Use [xbps-reconfigure(1)](https://man.voidlinux.org/xbps-reconfigure.1) to
ensure all installed packages are configured properly. Notably, this is
necessary to make [dracut(8)](https://man.voidlinux.org/dracut.8) generate an
initramfs and for GRUB to generate a working configuration.

```
(chroot) # xbps-reconfigure -fa
```

At this point, the installation is complete. Exit the chroot and reboot your
computer.

```
(chroot) # exit
# shutdown -r now
```

After booting into your Void installation for the first time, be sure to do a
system [update](../../xbps/updating.md).
