# Installation Guide

## Prepare Live Environment

Set the console keyboard layout.

```
# loadkeys trq
```

Show current console font and set another.

```
# showconsolefont
# ls /usr/share/kbd/consolefonts/
# setfont ter-v16b
```

Verify boot mode for EFI variables.

```
# efivar -l
# ls /sys/firmware/efi/efivars
```

Check network interfaces.

```
# ip link
# ip addr
```

> If you want connect to wifi use `iwctl`.

```
# ping era.yayd.in
```

Update system clock.

```
# timedatectl set-ntp true
# timedatectl
```

## Prepare Actual Environment

Check disks

```
# fdisk -l
# lsblk
```

Create partition table and partitions for SSD.

```
# fdisk /dev/nvme0n1

Command: g
Command: n
Command: w
```

Do same for HDD.

### Encryption disks

Create encryption for the SSD and HDD.

```
# cryptsetup benchmark
# cryptsetup --cipher aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 5000 --use-random --verify-passphrase luksFormat /dev/nvme0n1p1
# cryptsetup --cipher aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 5000 --use-random --verify-passphrase luksFormat /dev/sdb1
```

Open encrypted partitions.

```
# cryptsetup open --type luks /dev/nvme0n1p1 ssdlvm
# cryptsetup open --type luks /dev/sdb1 hddlvm
```

Wipe partitions.

```
# dd if=/dev/zero of=/dev/mapper/hddlvm bs=1M
# dd if=/dev/zero of=/dev/mapper/ssdlvm bs=1M
```

### LVM

```
# pvcreate /dev/mapper/hddlvm
# pvcreate /dev/mapper/ssdlvm
```

```
# vgcreate hddvg /dev/mapper/hddlvm
# vgcreate ssdvg /dev/mapper/ssdlvm
```

```
# lvcreate -L 16G ssdvg -n swap
# lvcreate -l 100%FREE ssdvg -n root
# lvcreate -l 100%FREE hddvg -n home
```

### Filesystems

```
# mkswap -L arch-swap /dev/ssdvg/swap
# mkfs.ext4 -L arch-root /dev/ssdvg/root
# mkfs.ext4 -L arch-home /dev/hddvg/home
```

```
# tune2fs -m 1.0 /dev/ssdvg/root
# tune2fs -m 0.0 /dev/hddvg/home
```

```
# tune2fs -c 30 -i 30d /dev/hddvg/home
# tune2fs -c 30 -i 4w /dev/ssdvg/root
```

```
# swapon /dev/ssdvg/swap
# mount /dev/ssdvg/root /mnt
# mkdir /mnt/{home,boot}
# mount /dev/hddvg/home /mnt/home
# mount /dev/sda1 /mnt/boot
```

Install system.

```
# pacstrap /mnt base base-devel linux linux-firmware dosfstools ntfs-3g lvm2 neovim iwd man-db man-pages texinfo dhcpcd cryptsetup efivar
```

Set file system tab configuration

```
# genfstab -U /mnt >> /mnt/etc/fstab
```

Change `relatime` to `noatime` for boot partition.

## Configuration System

```
# arch-chroot /mnt
# ln -sf /usr/share/zoneinfo/Europe/Istanbul /etc/localtime
# hwclock --systohc
```

```
# vim /etc/locale.gen
# locale-gen
# echo "LANG=en_US.UTF-8" > /etc/locale.conf
# echo "KEYMAP=trq" > /etc/vconsole.conf
```

```
# echo "Eray-ArchDesktop" > /etc/hostname
# nvim /etc/hosts
```

```
127.0.0.1 localhost
::1 localhost
127.0.1.1 Eray-ArchDesktop.localdomain Eray-ArchDesktop
```

```
# nvim /etc/mkinitcpio.conf
```

Add `keyboard keymap` before `modconf`.

Add `encrypt lvm2` before `filesystems`.

Add `shutdown` at the end.

```
# mkinitcpio -P
```

```
# passwd
```

```
# pacman -S intel-ucode
```

```
# bootctl --path=/boot install
```

Create file `/boot/loader/entries/arch.conf` (use `:r !blkid`)

```
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options cryptdevice=UUID=YOUR-DEVICE-UUID:ssdlvm root=/dev/mapper/ssdvg-root resume=/dev/mapper/ssdvg-swap rw
```

Update file `/boot/loader/loader.conf`

```
default arch
timeout 3
```

```
# nvim /etc/crypttab
```

```
hddlvm  /dev/sdb1   /etc/luks-keys/hddlvm-key
```

```
# mkdir /etc/luks-keys
# chmod 600 /etc/luks-keys
# dd if=/dev/urandom of=/etc/luks-keys/hddlvm-key bs=512 count=4
```

Add key to `keyslot`.

```
# cryptsetup luksAddKey /dev/sdb1 /etc/luks-keys/hddlvm-key
# cryptsetup luksDump /dev/sdb1
```

Unmount disks
```
# exit
# umount -R /mnt
```
