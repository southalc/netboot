## Table of Contents

1. [Overview](#overview)
1. [Setup](#setup)
1. [initramfs](#initramfs)
1. [rootfs](#rootfs)
1. [iPXE Example](#ipxe)
1. [mirror](#mirror)

### TL:DR

Repo provides build scripts for a custom initramfs and root filesystem image to support
network deployments of Ubuntu.  The root filesystem image includes a systemd service that
will download and execute a "bootscript" as defined by kernel boot options, providing a
flexible way to customize behavior.

### Overview

This repository contains some templates and build scripts for producing a custom
initramfs and root filesystem for booting Ubuntu over a network.  This was inspired
by changes to the last several Ubuntu releases:
  - Network boot requires download of an entire DVD ISO, or use of an NFS server
  - Without NFS, it is not possible to install Ubuntu desktop via network with 4G RAM
  - Even with NFS, the Ubuntu installer is unstable and generally sucks
  - "Supported" installation methods require stuff you may not need or want (snaps, cloud-init, etc)

I found a number of threads on support forums seeking answers to the above problems, but
Ubuntu devs seem uninterested in providing a robust, light weight, network installation
method.  The logic used to justify removal of the old d-i installer was to actively prevent
people from installing an "unsupported" Ubuntu configuration (whatever that means) but the
replacement method solves none of the above issues.  This method provides a way to deploy
custom, minimal Ubuntu systems over the network with more flexibilty and control.

### Setup

The assumption is that we're starting from a working Ubuntu installation.  You'll have to
get that part done yourself with the usual installation methods.  I'm also assuming you
have PXE infrastructure in place (DHCP, TFTP, HTTP) to host the files built by this repo.
This is not a guide on how to setup PXE services.

Building initramfs and a root filesystem image from here results in files of manageable
size (couple hundred MB) that can be used to boot a Ubuntu host from the network and
install a minimal system based on your own custom script.  I use this to install on a
ZFS root using "debootstrap" and have included a copy of my script for reference at
"files/zfs-installer.sh"

To boot over the network you need to stage these 3 files - I put mine on an HTTP server
and load them with a menu entry from iPXE, but there are other ways to direct systems
to boot from the network:
* kernel - I just use the kernel from my build machine by copying the target of the "/boot/vmlinuz"
soft link.  I installed the ZFS packages on my system so the needed kernel modules are all present.
You can use this directly to PXE boot network systems and the kernel will support ZFS.  This is
easier and faster than installing the ZFS packages later in the PXE booted kernel.
* initrd - The initial ramdisk, or "initramfs" is used by the kernel during boot to identify
and mount the root filesystem, then start an "init" process to hand over execution control.
This repo includes templates and a build script to generate a custom initrd that enables downloading
the rootfs image from the network.
* rootfs - A root filesystem image contains the usual binaries and libraries used by a
functional system.  If a rootfs cannot be found by initrd you end up in an environment with
minimal functionality.  This repo has a build script and some file templates to generate a
rootfs you can customize to your liking.


### initramfs

The "initramfs-tools" package has everything needed for this process and is almost certainly
already installed on an existing Ubuntu system.  A couple simple scripts, mostly borrowed
from the Ubuntu "casper" files, are included to build a custom initramrd that can in turn
download and mount a fully functional environment that can run whatever you like.

The "mkinitramfs" tool is used to create the initrd file we want.  This tool works with the
contents of "/usr/share/initramfs-tools" and "/etc/initramfs-tools", but can be pointed at
an alternative configuration directory as needed.  The contents of "/usr/share/initramfs-tools"
are always used, so you really need just a couple things in the custom configuration directory.
This repo has a "initramfs" directory that contains an empty directory tree with just a few
files to tune behavior:
  - ```modules``` Contains a list of kernel modules to be included/enabled in initrd.  The build may fail if you add modules to this list that are not present on your system.
  - ```hooks/netboot``` A template that allows adding files from the host system to initrd.  We really don't need to add anything here since initrd will just mount the rootfs and exit, but if that fails for some reason it's nice to have some basic tools.
  - ```scripts/netboot``` This script will be called by passing the "boot=netboot" boot argument to the kernel.  This script will then look for the "image_url=" boot argument and will download and mount the image as the root filesystem.

Build the custom initrd by running "build-initrd.sh".  This script does not require any privileges to run:
``` bash
$ ./build-initrd.sh
Building initramfs at /data/software/Linux/netboot/initrd.gz...
```

### rootfs

Building the rootfs is a little more complicated and does require root access because of the tools involved.
The "build-rootfs.sh" tool creates an empty file of a given size (512M default) and creates a filesystem inside
the file where all the needed bits are installed.  Once setup is complete, the contents of the mounted file
are exported to a squashfs file and the file is unmounted.  Either the squashfs or the image file can be used,
with a trade-off of squashfs being compressesed to save on network transfer but needing to be extracted upon
use, where the image file is larger but can be mounted directly.

There are a handful of variables defined at the top of the build script that can be used to tune the build.
Review the variables and update them as needed before running the script.  Here are the defaults, which should
be pretty self-explanatory.

- IMG_SIZE="512M"            # Only applies to "build-root-img.sh" and depends on what you want fit in the image
- VERSION_CODENAME="noble"
- DPKG_ARCH="amd64"
- INCLUDE_PACKAGES="zfsutils-linux,gdisk,openssh-server,openssh-client,wget,parted,debootstrap"
- EXCLUDE_PACKAGES="ubuntu-pro-client"
- APT_MIRROR=""              # Define this to install from local sources
- NETBOOT_HOSTNAME="netboot"
- NETBOOT_LOCALE="en_US.UTF-8"
- NETBOOT_TIMEZONE="US/Central"
- ENABLE_SSH=1               # Enable sshd in the netboot environment?  Enable=1, Disable=0
- ROOT_PASSWORD=""           # You don't need a value here unless you want local login during netboot
 
If "files/authorized_keys" is present and ENABLE_SSH=1, the file will be copied into the image as the root user's
authorized_keys file.

The script uses the "debootstrap" tool to install a minimal Ubuntu system in the working directory.

The "build-rootfs.sh" script does all the heavy lifting, but must be run with "sudo".
```
$ sudo ./build-rootfs.sh
Preparing and mounting the image file...
shred: /data/software/Linux/netboot/rootfs.img: pass 1/1 (000000)...
mke2fs 1.46.5 (30-Dec-2021)
Discarding device blocks: done
Creating filesystem with 131072 4k blocks and 32768 inodes
Filesystem UUID: 76cf916e-77e6-41b0-8585-c39314f48d29
Superblock backups stored on blocks:
        32768, 98304

Allocating group tables: done
Writing inode tables: done
Writing superblocks and filesystem accounting information: done

Installing packages in the new root image...
I: Retrieving InRelease
I: Checking Release signature
I: Valid Release signature (key id F6ECB3762474EDA9D21B7022871920D1991BC93C)
I: Retrieving Packages
I: Validating Packages
...<truncated>...
I: Base system installed successfully.
Generating locales (this might take a while)...
  en_US.UTF-8... done
Generation complete.
Un-mounting the image file...
Completed image file at: /data/software/Linux/netboot/rootfs.img
```

Round up your vmlinuz, initrd.gz, and rootfs.img (or rootfs.squashfs) files and stage them on the PXE boot server to see it in action.

### iPXE

This example ipxe configuration is a menu entry to boot the custom kernel, initrd.gz, and rootfs.img.
Note the use of "boot=netboot" that's needed to start the netboot script from initrd.  The "image_url"
is then downloaded and mounted as the root filesystem.  The "bootscript" option is used by a systemd
service inside the rootfs to download and run an arbitrary script.  This is where I've hooked it to the
ZFS installer script that's included in this repo under files/.  Note that the ipxe environment has
${base-url} defined elsewhere.
``` bash
:ubuntu-zfs-installer
kernel ${base-url}/ubuntu/custom/vmlinuz
initrd ${base-url}/ubuntu/custom/initrd.gz
imgargs vmlinuz \
  boot=netboot \
  bootscript=${base-url}/ubuntu/zfs-installer.sh \
  image_url=${base-url}/ubuntu/custom/rootfs.squashfs \
  console=tty0,115200 console=ttyS0,115200 \
  ---
boot || goto failed
goto start
```

If you use syslinux instead of ipxe, change your settings accordingly.

### mirror

If you want redundancy for the boot disk and root filesystem, these procedures can be used to clone
partitions between identical disks and mirror the data and boot partitions:

1) Install the needed packages:
```
apt-get install -y gdisk parted
```

2) Identify the disks and partitions currently in use by the system.  Helpful commands for this include
"mount", "zpool status", "lsblk", "ls -l /dev/disk/by-id", and "cat /etc/fstab"

3) Clone the disk partitions from the root disk to a second disk.
```
# ---- PAY ATTENTION TO THE SOURCE AND TARGET DISK ORDER ---
sgdisk -R <target_disk> <source_disk>     # Copy the partition table from source to target
sgdisk -G <target_disk>                   # Generate new GUIDs for the new disk partitions
partprobe                                 # Refresh partition table
```

4) Attach the new disk partitions to the root and boot ZFS pools.  The source devices will be from the
existing "zpool status" output.  It's recommended to use devices from /dev/disk/by-id/<device-partition>
```
zpool attach rpool <source> <target>      # Attach mirror to the root zpool
zpool attach bpool <source> <target>      # Attach mirror to the boot zpool
```

5) Identify the EFI partition on the new mirror disk and create a FAT32 file system.  Update "/etc/fstab"
to mount the disk label "EFI2" at "/boot/efi-mirror".  Re-load systemd after updating fstab and then mount
the EFI mirror:
```
mkdosfs -F 32 -s 1 -n EFI2 /dev/nvme1n1p1
<edit /etc/fstab" with your preferred editor>
systemctl daemon-reload
mount -a
```

6) Update grub so it will use both disks.  All disks with the EFS GPT type will be managed by grub.
```
dpkg-reconfigure grub-efi-amd64-signed
```

