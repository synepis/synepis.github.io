---
layout: post
title: "Installing Arch Linux on RAID0 with LUKS encryption" 
date:   2017-02-18 22:37:48 +0000
categories: arch-linux linux LUKS encryption
---
This is essentially a guide of how to install Arch Linux onto a RAID0 array with full-disk encryption with LUKS. The steps are this:
1. Partition the disk and setup encryption
   1. Use only 1 root partition
   2. Use LVM for multiple partitions
2. Install the base system (via pacstrap) 
3. Do inital system config
4. Setup bootloader (syslinux)

## 1. Partition disk
On my system I have a RAID0 array called `raid0`, you can usually configure this in your BIOS. You can then see your array under `/dev/md`
```
$ ls /dev/md
imsm0 raid0_0 raid0_0p1 raid0_0p2
```
In my case `raid0_0` is the actual raid array and the `raid0_0p#` are two partitions on it.

We need atleast 2 partitions:
* Boot - unecrypted and used to start the initial system
* CryptRoot - hold the actual system files (note that partition can have sub-partitions on it 

We will use parted to setup the partitions like so:
```
$ parted /dev/md/raid0_0
$(parted) mklabel msdos // for MBR
$(parted) mkpart primary ext4 0% 250MiB 
$(parted) set 1 boot on
$(parted) mkpart primary ext4 250MiB 100% 
$(parted) quit
```
After the operation the disk should look something like this:
```
$ fdisk -l /dev/md/raid0_0
Disk /dev/md/raid0_0: 238.5 GiB, 256083230720 bytes, 500162560 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 131072 bytes / 524288 bytes
Disklabel type: dos
Disk identifier: 0x027d10c9
 
Device                  Boot  Start       End   Sectors   Size Id Type
/dev/md/raid0_0p1 *    2048    511999    509952   249M 83 Linux
/dev/md/raid0_0p2      512000 500162559 499650560 238.3G 83 Linux
```

Format boot partition:
```
$ mkfs.ext4 -L boot /dev/md/raid0_0p1
```

### 1.1 Use only 1 root partition
Setup LUKS encryption on (crypt)root partition:
```
$ cryptsetup luksFormat /dev/md/raid0_0p2
```

Open/decrypt partition and mount it with name 'root' (it will be under /dev/mapper/root):
```
$ cryptsetup open /dev/md/raid0_0p2 root
```

Format (decrypted) root partition
```
$ mkfs.ext4 -L root /dev/mapper/root/
```

The state of the disk should be something like this:
* sdf and sde are 2 USBs
* sda, sdb, sdc, sdd are 4 disks in RAID0 
* md126 is sym linked from raid0_0
* md126p1 is sym linked from raid0_0p1 which is boot
* md126p2 is sym linked from raid0_0p2 which is the encrypted partition
* root is the mapped/decrypted md126p2 partition
```
$ lsblk
 NAME      MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
 sdf         8:80   1  28.9G  0 disk
 └─sdf1              8:81   1  28.9G  0 part  /usb
 sdd         8:48   0  59.6G  0 disk
 └─md126       9:126  0 238.5G  0 raid0
   ├─md126p1 259:1    0         249M  0 md
   └─md126p2 259:3    0 238.3G  0 md
     └─root  253:0    0 238.3G  0 crypt
 sdb         8:16   0  59.6G  0 disk
 └─md126       9:126  0 238.5G  0 raid0
   ├─md126p1 259:1    0         249M  0 md
   └─md126p2 259:3    0 238.3G  0 md
     └─root  253:0    0 238.3G  0 crypt
 sde         8:64   0  59.6G  0 disk
 └─md126       9:126  0 238.5G  0 raid0
   ├─md126p1 259:1    0         249M  0 md
   └─md126p2 259:3    0 238.3G  0 md
     └─root  253:0    0 238.3G  0 crypt
 loop0       7:0    0 360.3M  1 loop  /run/archiso/sfs/airootfs
 sdc         8:32   0  59.6G  0 disk
 └─md126       9:126  0 238.5G  0 raid0
   ├─md126p1 259:1    0         249M  0 md
   └─md126p2 259:3    0 238.3G  0 md
     └─root  253:0    0 238.3G  0 crypt
 sda         8:0    1  29.2G  0 disk
 ├─sda2              8:2    1    64M  0 part
 └─sda1              8:1    1   867M  0 part  /run/archiso/bootmnt
```
We are now ready to install the actual system.

### 1.2 Use LVM for multiple partitions:
If you wish to setup more partition rather than just 1 root like the example above, 
you can use LVM to accomplish that.

We setup LUKS encryption on (crypt)root partition (just like above):
```
$ cryptsetup luksFormat /dev/md/raid0_0p2
$ cryptsetup open /dev/md/raid0_0p2 cryptroot
```

Create logical volumes (root and swap partitions):
```
$ pvcreate /dev/mapper/cryptroot
$ vgcreate vol /dev/mapper/cryptroot
$ lvcreate -L 8G vol -n swap
$ lvcreate -l 100%FREE vol -n root
```
Format the partitions
```
$ mkfs.ext4 /dev/mapper/vol-root
$ mkswap /dev/mapper/vol-swap
```

Mount and enable swap partition
```
$ mount /dev/mapper/vol-root /mnt
$ mkdir /mnt/boot 
$ mount /dev/md/raid0_0p1 /mnt/boot
$ swapon /dev/mapper/vol-swap
```

## 2. Install the base system (via pacstrap)

Refresh mirrorlist and get top 3 fastest ones:
```
$ cd /etc/pacman.d/
$ wget "https://www.archlinux.org/mirrorlist/?country=GB" -O tmplist
$ sed -i 's/^#//' tmplist
$ rankmirrors -n 3 tmplist > mirrorlist
```

Mount boot and root partition:
```
$ mount /dev/mapper/root /mnt
$ mkdir /mnt/boot
$ mount /dev/md/raid0_0p1 /mnt/boot/
```

Install base packages:
```
$ pacstrap /mnt base base-devel
```

Save fstab to new system:
```
$ genfstab -p -U /mnt > /mnt/etc/fstab
$ cat /mnt/etc/fstab
# /dev/mapper/root LABEL=root
UID=1f73e5f1-6a20-4fce-9c48-c0a85f1c8c99     /               ext4            rw,relatime,stripe=128,data=ordered     0 1
 
# /dev/md126p1 LABEL=boot
UUID=650783be-242b-4235-b95b-b6bee99d8264     /boot           ext4            rw,relatime,stripe=512,data=ordered     0 2
```

Chroot into new system:
```
$ arch-chroot /mnt
```

## 3. Do inital system config

Setup language
```
$ echo en_US.UTF-8 UTF-8 > /etc/locale.gen
$ locale-gen
Generating locales...
  en_US.UTF-8... done
Generation complete.
$ echo LANG=en_US.UTF-8 > /etc/locale.conf
$ rm /etc/localtime
$ ln -s /usr/share/zoneinfo/Europe/London /etc/localtime
```

Setup hostname:
```
$ echo portbox > /etc/hostname
```

Edit hosts file:
```
$ cat /etc/hosts
#
# /etc/hosts: static lookup table for host names
#
 
#<ip-address> <hostname.domain.org>   <hostname>
127.0.0.1     localhost.localdomain   localhost portbox
::1           localhost.localdomain   localhost portbox
```
Add root password:
```
$ passwd
```

Add user:
```
$ useradd -m -g users synepis
$ passwd synepis
$ usermod -aG wheel synepis
```

## 4. Setup bootloader
Install boot manager (I use syslinux)
```
$ pacman -S syslinux
$ syslinux-install_update -iam
```

Now we need to configure the boot loader with encrypted drive.
Your /boot/syslinux/syslinux.cfg file should look something like this
(I've stripped out everything except bare bones here - minimal config):
```
$ cat /boot/syslinux/syslinux.cfg
DEFAULT arch
LABEL arch
    MENU LABEL Arch Linux
    LINUX ../vmlinuz-linux
    APPEND cryptdevice=/dev/md/raid0_0p2:root root=/dev/mapper/root
    INITRD ../initramfs-linux.img
```

**Note:If you used LVM then the APPEND line would look like this:**
```
APPEND cryptdevice=/dev/md/raid0_0p2:cryptroot root=/dev/mapper/vol-root rw
```

Lastly, we need to build an initram disk. Hooks need to be added in order for it (initramfs)
to decrypt the partition and load from it.

Open  `/etc/mkinipcio.conf` and add `/sbin/mdmon` to BINARIES and  `mdadm_udev` and `encrypt` to HOOKS. 

*Note:* It is important to add the hooks in the correct order.

Contents of: `/etc/mkinipcio.conf`:
```
...
BINARIES="/sbin/mdmon"
...
HOOKS="base udev autodetect modconf `block mdadm_udev` encrypt filesystems keyboard fsck"
...
```

**Note:If you used LVM then the HOOKS section needs `lvm2` as well (insert it after the `encrypt` in the HOOKS line)**


Build the initramfs and reboot the system:
```
$ mkinitcpio -p linux
$ exit
$ reboot
```

That's it! We're done! Now just reboot the system, you should be prompted for a password upon booting.
