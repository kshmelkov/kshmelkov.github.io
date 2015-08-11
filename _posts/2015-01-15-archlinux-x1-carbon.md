---
layout: post
title: ArchLinux on LUKS-encrypted Btrfs with UEFI
---

Recently I have upgraded my system to Lenovo ThinkPad X1 Carbon Gen2 laptop. I
decided to install relatively complicated setup: ArchLinux on Btrfs with full drive
encryption (LUKS) and UEFI boot from USB key with /boot and LUKS header.

It allows to have deniable encryption (LUKS header on USB key, laptop's SSD is
entirely encrypted), two-factor authorization (you can't load without LUKS
header which is encrypted with your password). UEFI boot is just for fun.
I highly recommend to read all [ArchWiki pages](https://wiki.archlinux.org/index.php/Dm-crypt) about full disk
encryption to understand different stages of installation.

I have choosed Btrfs because it is a modern, fast filesystem and has some cool
features like snapshots and dynamic subvolumes. It can also manage its
partitions (subvolumes) without partition table. As we put /boot on USB key,
we can give entire SSD for Btrfs (no swap support though). Another important
question is TRIM/discard support. In this setting it is totally possible, but
it is considered [insecure](http://asalor.blogspot.fr/2011/08/trim-dm-crypt-problems.html). It is up to you to decide.

I have read lots of wiki pages and forums to figure out the installation
process. So I want to explain it step-by-step for other people.

As USB key I have chosed [Integral Fusion USB 3.0](http://www.integralmemory.com/product/fusion-usb-3-superspeed-small-metal-flash-drive). It has rather good
read speed, but awful write speed (not important for me). It is also very
small and has no cap.

**DISCLAIMER**: You do everything described on your own risk. By the way, you will _destroy_ all the data present on your laptop. I hope you understand it. If it isn't clear, please re-read ArchWiki.

----

First of all, don't forget to update your BIOS version. It can be done from
Windows slightly easier. You need another USB key to write down ArchLinux
installation image. Then go to BIOS and set the boot mode as UEFI first. Boot
from this USB in UEFI mode. On X1 Carbon you should press F12 after powering
on to choose USB device to boot from.

If you have a laptop with HiDPI you will be surprised with such a small font.
Fix it with command:

```bash
setfont sun12x22
```

It becomes a bit better. Actually it is the largest console font available.
Then check that you are in UEFI mode:

```bash
efivar -l
```

If you get an error, go again to BIOS. Now you need to connect to the
Internet. I connect via WiFi so:

```bash
wifi-menu
```

Double check that you have an Internet access. First of all, wipe the disk. I
amn't sure if it really necessary with the SSD, but it is widely recommended
(especially for deniable security). To fill a disk with pseudo-random data you
can use `/dev/urandom` (it is too slow) or open the whole disk as an encrypted
volume and write zeros inside. The sequence of zeros will be encrypted in very
messy data. If you are really serious about deniable security, choose the same
cipher that you want to use later. I need to put an usual disclaimer that next
actions are irreversible, you destroy all data on `/dev/sdX` (it is your SSD by
the way).

```bash
cryptsetup open --type plain /dev/sdX ssd
```

Enter anything as password.

```bash
ddrescue -f /dev/zero /dev/mapper/ssd
```

It'll take a while. Hopefully SSD is fast, for me it was a bit less than 10 min.

```bash
cryptsetup close ssd
```

Now initialization of bootable USB key. I will denote it as `/dev/sdY`.

```bash
gdisk /dev/sdY
```

Create new GPT partition table and small partition for /boot (about 300-500Mb).
Define its type as EF00 (in gdisk EFI System Partition). Exit from gdisk and
format it in FAT32:

```bash
mkfs.fat -F32 /dev/sdY1
mkdir /efi
mount /dev/sdY1 /efi
cd /efi
```

Next create an empty file for LUKS header (you might modify header size if you choose other encryption settings):

```bash
truncate -s 2M header.img
```

Create device-mapping encrypted device with this header. Before choosing a
cipher, you can test the speed of your system with

```bash
cryptsetup benchmark
```

If you choose XTS mode, remember that effective key size is divided by two.
Let's create an encrypted device:

```bash
cryptsetup --cipher=aes-xts-plain64 --hash=sha512 --verify-passphrase --key-size=256 luksFormat /dev/sdX --header header.img
cryptsetup open --header header.img --type luks /dev/sdX root
```

Next create the filesystem and mount it. If you decide to enable TRIM, add
discard to mount options:

```bash
mkfs.btrfs -L "ARCHROOT" /dev/mapper/root
mount -o defaults,noatime,ssd /dev/mapper/root /mnt/btrfs-root
btrfs subvolume create /mnt/btrfs-root/__active
btrfs subvolume create /mnt/btrfs-root/__active/home
btrfs subvolume create /mnt/btrfs-root/__active/var
```

You can add some other subvolumes if you want.

```bash
mount -o subvol=__active /dev/mapper/root /mnt
```

Change default permissions (700):

```bash
chmod 755 /mnt/btrfs-root/__active
chmod 755 /mnt/btrfs-root/__active/home
chmod 755 /mnt/btrfs-root/__active/var
```

Before chrooting remount USB key:

```bash
umount /efi
mkdir /mnt/boot
mount /dev/sdY1 /mnt/boot
```

Now you can proceed with ordinary install, you can
consult [ArchWiki installation guide](https://wiki.archlinux.org/index.php/installation_guide):

```bash
nano /etc/pacman.d/mirrorlist
pacstrap /mnt base base-devel wpa_supplicant gummiboot
genfstab -U -p /mnt &gt;&gt; /mnt/etc/fstab
```

Later you can add `noauto` option in `fstab` for `/boot` partition. Nevertheless, it
must be mounted to update the kernel. Now we are ready to go inside!

```bash
arch-chroot /mnt /bin/bash
nano /etc/locale.gen
```

Uncomment your favorite locale, for example, en_US.UTF-8 UTF-8.

```bash
locale-gen
echo LANG=en_CA.UTF-8 &gt; /etc/locale.conf
export LANG=en_US.UTF-8
```

Choose your timezone

```bash
ls /usr/share/zoneinfo/
ln -s /usr/share/zoneinfo/&lt;continent&gt;/&lt;city&gt; /etc/localtime
hwclock --systohc --utc
```

In the next file you should enter

```
FONT=sun12x22
KEYMAP=us
```

or another keymap in which you have password:

```bash
nano /etc/vconsole.conf
```

Store the desired network name:

```bash
nano /etc/hostname
```

You could make a `netctl` profile for the first boot.
Then set root password:

```bash
passwd
```

Now it is an important moment. You need to set up
initcpio hooks to successfully unlock the disk.
Unfortunately, default encrypt hook doesn't yet support
remote LUKS header, so you need to modify it.

```bash
nano /etc/mkinitcpio.conf
```

Remove `fsck` hook, insert

```bash
keyboard keymap consolefont encrypt2
```

just before `filesystems` hook (keyboard exists normally, but
you should move it before `encrypt2`). Add also

```
MODULES="loop"
FILES="/boot/header.img"
```

I added also in MODULES i915 to enable early KMS on my laptop.
Create encrypt2 hook as described in [ArchWiki](https://wiki.archlinux.org/index.php/Dm-crypt/Specialties#Encrypted_system_using_a_remote_LUKS_header)

```bash
cp /lib/initcpio/hooks/encrypt{,2}
cp /usr/lib/initcpio/install/encrypt{,2}
nano /lib/initcpio/hooks/encrypt2
```

Now you are ready to regenerate initramfs image:

```bash
mkinitcpio -p linux
```

It is time to install bootloader. I decided to use gummiboot
because it is purely UEFI, simple, text-based. You could use GRUB2
or rEFInd:

```bash
gummiboot install
```

Insert following files. If you don't want menu,
insert timeout 0. In this case you can summon it if you
press space while loading.

Put the following text into file `/boot/loader/loader.conf`:

```
default  arch
timeout 3
```

And create a corresponding file `/boot/loader/entries/arch.conf`:

```
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options cryptdevice=/dev/sdX:root:header root=/dev/mapper/root rootflags=subvol=__active rw
```
Add `,allow-discards` after header option if you decided to go with TRIM/discard support.

Hope that it works. It is time to try!

```bash
exit
umount /mnt/boot
umount /mnt
cryptsetup close root
reboot
```
