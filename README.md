# A Personal Arch Installation Guide

This is a personal guide so if you are lost and just found this guide from somewhere, I recommend you to read the official [`wiki`](https://wiki.archlinux.org/index.php/Installation_guide)!  This guide will focus on `grub2` and `UEFI`. This is a customized fork of [`this repo`](https://github.com/manilarome/A-Personal-Arch-Installation-Guide) by https://github.com/manilarome

## Pre-installation

Before installing, make sure to:

+ Read the [official wiki](https://wiki.archlinux.org/index.php/installation_guide). It is advisable to read that instead. I wrote this guide for myself.
+ Acquire an installation image from [here](https://www.archlinux.org/download/).
+ Verify signature.
+ Prepare an installation medium.
+ Boot the live environment.

## Set the keyboard layout

The default console keymap is US. Available layouts can be listed with:

```
# ls /usr/share/kbd/keymaps/**/*.map.gz
```

To modify the layout, append a corresponding file name to loadkeys, omitting path and file extension. For example, to set a DE keyboard layout:  

```
# loadkeys de
```

## Verify the boot mode

If UEFI mode is enabled on an UEFI motherboard, Archiso will boot Arch Linux accordingly via systemd-boot. To verify this, list the efivars directory:  

```
# ls /sys/firmware/efi/efivars
```

If the command shows the directory without error, then the system is booted in UEFI mode. If the directory does not exist, the system may be booted in **BIOS** (or **CSM**) mode.

## Connect to the internet

We need to make sure that we are connected to the internet to be able to install Arch Linux `base` and `linux` packages. Let’s see the names of our interfaces.

```
# ip link
```

You should see something like this:

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
		link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state DOWN mode DEFAULT group default qlen 1000
		link/ether 00:00:00:00:00:00 brd ff:ff:ff:ff:ff:ff
3: wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DORMANT group default qlen 1000
		link/ether 00:00:00:00:00:00 brd ff:ff:ff:ff:ff:ff permaddr 00:00:00:00:00:00
```

+ `enp0s0` is the wired interface  
+ `wlan0` is the wireless interface  

### Wired Connection

If you are on a wired connection, you can enable your wired interface by systemctl start `dhcpcd@<interface>`.  

```
# systemctl start dhcpcd@enp0s0
```

### Wireless Connection

If you are on a laptop, you can connect to a wireless access point using `iwctl` command from `iwd`. Note that it's already enabled by default. Also make sure the wireless card is not blocked with `rfkill`.

Scan for network.

```
# iwctl station wlan0 scan
```

Get the list of scanned networks by:

```
# iwctl station wlan0 get-networks
```

Connect to your network.

```
# iwctl -P "PASSPHRASE" station wlan0 connect "NETWORKNAME"
```

Ping archlinux website to make sure we are online:

```
# ping archlinux.org
``` 

If you receive Unknown host or Destination host unreachable response, means you are not online yet. Review your network configuration and redo the steps above.

## Update the system clock

Use `timedatectl` to ensure the system clock is accurate:

```
# timedatectl set-ntp true
```

To check the service status, use `timedatectl status`.

## Partition the disks

When recognized by the live system, disks are assigned to a block device such as `/dev/sda`, `/dev/nvme0n1` or `/dev/mmcblk0`. To identify these devices, use lsblk or fdisk.  The most common main drive is **sda**.

```
# lsblk
```

Results ending in `rom`, `loop` or `airoot` may be ignored.

In this guide, I'll create a two different ways to partition a drive. One for a normal installation, the other one is setting up with an encryption(LUKS/LVM). Let's start with the unecrypted one:

### Unencrypted filesystem

+ Let’s clean up our main drive to create new partitions for our installation. And yeah, in this guide, we will use `/dev/sda` as our disk.

	```
	# gdisk /dev/sda
	```

+ Press <kbd>x</kbd> to enter **expert mode**. Then press <kbd>z</kbd> to *zap* our drive. Then hit <kbd>y</kbd> when prompted about wiping out GPT and blanking out MBR. Note that this will ***zap*** your entire drive so your data will be gone - reduced to atoms after doing this. THIS. CANNOT. BE. UNDONE.

+ Open `cgdisk` to start partitioning our filesystem

	```
	# cgdisk /dev/sda
	```

+ Press <kbd>Return</kbd> when warned about damaged GPT.

	Now we should be presented with our main drive showing the partition number, partition size, partition type, and partition name. If you see list of partitions, delete all those first.

+ Create the `boot` partition

	- Hit New from the options at the bottom.
	- Just hit enter to select the default option for the first sector.
	- Now the partion size - Arch wiki recommends 200-300 MB for the boot + size. Let’s make 1GiB in case we need to add more OS to our machine. I’m gonna assign mine with 1024MiB. Hit enter.
	- Set GUID to `EF00`. Hit enter.
	- Set name to `boot`. Hit enter.
	- Now you should see the new partition in the partitions list with a partition type of EFI System and a partition name of boot. You will also notice there is 1007KB above the created partition. That is the MBR. Don’t worry about that and just leave it there.

+ Create the `swap` partition

	- Hit New again from the options at the bottom of partition list.
	- Just hit enter to select the default option for the first sector.
	- For the swap partition size, I always assign mine with 1GiB. Hit enter.
	- Set GUID to `8200`. Hit enter.
	- Set name to `swap`. Hit enter.

+ Create the `root` partition

	- Hit New again.
	- Hit enter to select the default option for the first sector.
	- Hit enter again to input your root size.
	- Also hit enter for the GUID to select default(`8300`).
	- Then set name of the partition to `root`.

+ Create the `root` partition

	- Hit New again.
	- Hit enter to select the default option for the first sector.
	- Hit enter again to use the remainder of the disk.
	- Also hit enter for the GUID to select default.
	- Then set name of the partition to `home`.

+ Lastly, hit `Write` at the bottom of the patitions list to *write the changes* to the disk. Type `yes` to *confirm* the write command. Now we are done partitioning the disk. Hit `Quit` *to exit cgdisk*. Go to the [next section](#formatting-partitions).


## Verifying the partitions

Use `lsblk` again to check the partitions we created. *We? I thought I'm doing this guide for myself lol*

```
# lsblk
```

You should see *something like this*:

### Unencrypted filesystem

| NAME | MAJ:MIN | RM | SIZE | RO | TYPE | MOUNTPOINT |
| --- | --- | --- | --- | --- | --- | --- |
| sda | 8:0 | 0 | 477G | 0 |   |   |
| sda1 | 8:1 | 0 | 1 | 0 | part |   |
| sda2 | 8:2 | 0 | 1 | 0 | part |   |
| sda3 | 8:3 | 0 | 175G | 0 | part |   |
| sda4 | 8:4 | 0 | 300G | 0 | part |   |

**`sda`** is the main disk  
**`sda1`** is the boot partition  
**`sda2`** is the swap partition  
**`sda3`** is the home partition  
**`sda4`** is the root partition

## Format the partitions

### Unencrypted filesystem

+ Format `/dev/sda1` partition as `FAT32`. This will be our `/boot`.

	```
	# mkfs.fat -F32 /dev/sda1
	```

+ Create and enable our `swap` under the `/dev/sda2` partition.

	```
	# mkswap /dev/sda2
	# swapon /dev/sda2
	```

+ Format `/dev/sda3` and `/dev/sda4` partition as `EXT4`. This will be our `root` and `home`  partition.

	```
	# mkfs.ext4 /dev/sda3
	# mkfs.ext4 /dev/sda4
	```



## Mount the filesystems

### Unencryped partition

+ Mount the `/dev/sda` partition to `/mnt`. This is our `/`:

	```
	# mount /dev/sda3 /mnt
	```

+ Create a `/boot` mountpoint:

	```
	# mkdir /mnt/boot  
	```

+ Mount `/dev/sda1` to `/mnt/boot` partition. This is will be our `/boot`:

	```
	# mount /dev/sda1 /mnt/boot
	```

+ Create a `/home` mountpoint:

	```
	# mkdir /mnt/home  
	```

+ Mount `/dev/sda4` to `/mnt/home` partition. This is will be our `/home`:

	```
	# mount /dev/sda1 /mnt/home
	```

	We don’t need to mount `swap` since it is already enabled.  


## Installation

Now let’s go ahead and install `base`, `linux`, `linux-firmware`, `base-devel`, `neovim`, `dhcpcd` and `iwd` packages into our system.
**_TIP:_** Please replace `neovim` with your [text editor](https://wiki.archlinux.org/index.php/Category:Text_editors) of choice.
**Just make sure that it runs in the terminal or else it _won't work!_**
Some terminal based editors are located farther down in the documentation.

```
# pacstrap /mnt base base-devel linux linux-firmware neovim dhcpcd iwd
```

The `base` package does not include all tools from the live installation, so installing other packages may be necessary for a fully functional base system. In particular, consider installing: 

+ userspace utilities for the management of file systems that will be used on the system,
	
	- `ntfs-3g`: NTFS filesystem driver and utilities
	- `unrar`: The RAR uncompression program
	- `unzip`: For extracting and viewing files in `.zip` archives
	- `p7zip`: Command-line file archiver with high compression ratio
	- `unarchiver`: `unar` and `lsar`: Objective-C tools for uncompressing archive files
	- `gvfs-mtp`: Virtual filesystem implementation for `GIO` (`MTP` backend; Android, media player)
	- `libmtp`: Library implementation of the Media Transfer Protocol
	- `android-udev`: Udev rules to connect Android devices to your linux box
	- `mtpfs`: A FUSE filesystem that supports reading and writing from any MTP devic
	- `xdg-user-dirs`: Manage user directories like `~/Desktop` and `~/Music`

+ utilities for accessing `RAID` or `LVM` partitions,

	- `lvm2`: Logical Volume Manager 2 utilities (*if you are setting up an encrypted filesystem with LUKS/LVM, include this on pacstrap*)

+ specific firmware for other devices not included in `linux-firmware`,
	
+ software necessary for networking,

	- `dhcpcd`: RFC2131 compliant DHCP client daemon
	- `iwd`: Internet Wireless Daemon
	- `inetutils`: A collection of common network programs
	- `iputils`: Network monitoring tools, including `ping`

+ a text editor(s),

	- `nano`
	- `vim`
	- `vi`

+ packages for accessing documentation in man and info pages,

	- `man-db`
	- `man-pages`

+ and more useful tools:

	- `git`: the fast distributed version control system
	- `tmux`: A terminal multiplexer
	- `less`: A terminal based program for viewing text files
	- `usbutils`: USB Device Utilities
	- `bash-completion`: Programmable completion for the bash shell

These tools will be useful later. So **future me**, install these.

## Generating the fstab

```
# genfstab -U /mnt >> /mnt/etc/fstab
```

Check the resulting `/mnt/etc/fstab` file, and edit it in case of errors. 

## Chroot

Now, change root into the newly installed system  

```
# arch-chroot /mnt /bin/bash
```

## Time zone

A selection of timezones can be found under `/usr/share/zoneinfo/`. Since I am in the Philippines, I will be using `/usr/share/zoneinfo/Asia/Manila`. Select the appropriate timezone for your country:

```
# ln -sf /usr/share/zoneinfo/Asia/Manila /etc/localtime
```

Run `hwclock` to generate `/etc/adjtime`: 

```
# hwclock --systohc
```

This command assumes the hardware clock is set to UTC.

## Localization

The `locale` defines which language the system uses, and other regional considerations such as currency denomination, numerology, and character sets. Possible values are listed in `/etc/locale.gen`.

**Uncomment everything that begins with** `de_DE.` and **uncomment** `en_US.UTF-8` (for fallback) and other needed locales in `/etc/locale.gen`, **save**, and generate them with:  


_More info [here](https://wiki.archlinux.org/index.php/Locale#Generating_locales)_

```
# locale-gen
```

Create the `locale.conf` file, and set the LANG variable accordingly:  

```
# locale > /etc/locale.conf
```

If you set the keyboard layout earlier, make the changes persistent in `vconsole.conf`:

```
# echo "KEYMAP=de-latin1" > /etc/vconsole.conf
```

Not using `de` layout? Replace it, with `us` or some other keyboard layout.

## Network configuration

Create the hostname file. In this guide I'll just use `MYHOSTNAME` as hostname. Hostname is the host name of the host. Every 24 hours, a day passes in Africa.

**_BTW:_** I would normally use `archlolotl`. This is one of the main reasons I use ARCH Linux ᕦ(⌐■⩌■)ᕥ... mmh...

```
# echo "MYHOSTNAME" > /etc/hostname
```

Open `/etc/hosts` to add matching entries to `hosts`:

```
127.0.0.1    localhost  
::1          localhost  
127.0.1.1    MYHOSTNAME.localdomain	  MYHOSTNAME
```

If the system has a permanent IP address, it should be used instead of `127.0.1.1`.

## Initramfs  

Creating a new initramfs is usually not required, because mkinitcpio was run on installation of the kernel package with pacstrap. **This is important** if you are setting up a system with encryption!

### Unencrypted filesystem

	```
	# mkinitcpio -p linux
	```



## Installing an AUR Helper

There are a **trillion** AUR Helpers out there. Everyone has it's own unique features and tradeoffs.

I use [`paru`](https://github.com/Morganamilo/paru) - [`AUR link`]

Clone and **install** it with:
```
git clone https://aur.archlinux.org/paru-bin.git
cd paru-bin/
makepkg -si
```
or Clone and **Build** it with:
```
git clone https://aur.archlinux.org/paru.git
cd paru/
makepkg -si
```

### `pacman` easter eggs

You can enable the "easter-eggs" in `pacman`, the package manager of archlinux.

Open `/etc/pacman.conf`, then find `# Misc options`. 

To add colors to `pacman`, uncomment `Color`. Then add `Pac-Man` to `pacman` by adding `ILoveCandy` under the `Color` string:

```
Color
ILoveCandy
```

### Sudo insults

You can enable sudo insults with:

Run `sudo visudo`

Under `# Defaults specification`: 

Add:

```
Defaults insults
```

Now, whenever you run sudo and type in a wrong password, the shell will insult you.

**_~~How very uselful!~~_**

### Update repositories and packages

To check if you successfully enabled the easter-eggs, run:

```
# pacman -Syu
```

If updating returns an error, open the `pacman.conf` again and check for human errors. My pet ferret has better typing skills than you!.

## Root password

Set the `root` password:  

```
# passwd
```

## Add a user account

Add a new user account. In this guide, I'll just use `MYUSERNAME` as the username of the new user aside from `root` account.

Of course, change the example username with your own:  

```
# useradd -m -g users -G wheel,storage,power,video,audio,rfkill,input -s /bin/bash MYUSERNAME
```

This will create a new user and its `home` folder.

Set the password of user `MYUSERNAME`:  

```
# passwd MYUSERNAME
```

## Add the new user to sudoers:

If you want a root privilege in the future by using the `sudo` command, you should grant one yourself:

```
# EDITOR=vim visudo
```

Uncomment the line (Remove #):

**Before:**

```
# %wheel ALL=(ALL) ALL
```

**After:**
```
%wheel ALL=(ALL) ALL
```
~~_**I'm so proud of you! You have uncommented a line! Impressive!**_~~

## Install the boot loader

First, lets install grub:

```
pacman -S grub2 efibootmgr
```

### UEFI Systems

If you wan't, replace `ArchLinuxGrub` with your own custom ID. **Just make sure to put the ID in between double quotes if it has spaces.**

```
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=ArchLinuxGrub
```

Aaaand create the config file

```
grub-mkconfig -o /boot/grub/grub.cfg
```

### BIOS Systems

Install grub with: (Replace _Y_ with your disk name. for example /dev/sda)
```
grub-install /dev/sd_Y_
```

```
grub-mkconfig -o /boot/grub/grub.cfg
```



## Enable internet connection for the next boot

To enable the network daemons on your next reboot, you need to enable `dhcpcd.service` for wired connection and `iwd.service` for a wireless one.

```
# systemctl enable dhcpcd iwd
```

## Exit chroot and reboot:  

Exit the chroot environment by typing `exit` or pressing <kbd>Ctrl + d</kbd>. You can also unmount all mounted partition after this. 

Finally, `reboot`.

##  Finale

If your installation is a success, then ~~***yay!!!***~~ **_paru!_**

## [[POST INSTALLATION]](./POST.md)		[[EXTRAS]](./EXTRAS.md)
