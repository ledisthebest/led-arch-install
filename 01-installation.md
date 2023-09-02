# Installation

Download the [Latest Arch Linux ISO](https://archlinux.org/download/) image, and create an installation medium such as USB flash drive.

To check if we are booted in UEFI Mode and 64 bit system, run the following and make sure it returns `64`

```sh
cat /sys/firmware/efi/fw_platform_size
```

## Network Connection

to ensure our network interfaces is listed and enabled with:

```sh
ip link
# or
ip l
```

### Connect to Wifi

We can we `IWD` to connect to a wireless network

```sh
iwctl

[iwd]# device list

[iwd]# station <wlan0> scan

[iwd]# station <wlan0> get-networks

[iwd]# station <wlan0> connect <SSID>

exit
```

and verify the connection with:

```sh
ping archlinux.org
```

5. Sync system clock

```sh
timedatectl

timedatectl set-timezone <REGION/TIMEZONE>
```

## Disk Partitioning

Find the desired disk to install on with:

```sh
fdisk -l
# or
lsblk -fs -p
```

We can use `Gdisk` to clear the whole drive:

```sh
gdisk /dev/sda

# enter expert mode
Command: x

# zap(destroy) the GPT data structures
Command: z
```

Use `CGdisk` to create partitions:

```sh
cgdisk /dev/sda

# new boot partion
1GB, ef00, "boot"

# new root partition
32GB+, 8300, "primary" or "root"

# seperate home partition (optional)
16GB+, 8300, "home"

# swap partition (can be replaced with swap file instead)
# swap size should be at least equal to the square root of the RAM size and at most double the size of RAM
# If hibernation is used: swap size = RAM size + square root of the RAM size
8GB, 8200, "swap"
```

Write and Exit, verify the partitions with `lsblk`.

## Disk Formating

Fo UEFI EFI Partition:

```sh
mkfs.fat -F 32 /dev/sda1
```

For our `root` and `home` partitions, recommend using `EXT4` or `BTRFS` [(Only GRUB supports booting from BTRFS partition)](https://wiki.archlinux.org/title/Arch_boot_process#Feature_comparison)

```sh
mkfs.btrfs /dev/sda2
```

For SWAP partition:

```sh
mkswap /dev/sda3

swapon /dev/sda3
```

## Mounting File Systems

We must mount our `root` partition first

```sh
mount /dev/sda2 /mnt
```

Then mount the `boot` partition for UEFI EFI partition:

```sh
mount --mkdir /dev/sda1 /mnt/boot
```

If there is a seperate `home` partition:

```sh
mount --mkdir /dev/sda3 /mnt/home
```

## Swap file

If we didn't create a swap partition, we can use a swap file instead ([using swap file on BTRFS has limitations](https://wiki.archlinux.org/title/Btrfs#Swap_file)):

```sh
dd if=/dev/zero of=/mnt/swapfile bs=1M count=8k status=progress

chmod 0600 /mnt/swapfile

mkswap -U clear /mnt/swapfile

swapon /mnt/swapfile
```

On `BTRFS` System:

```sh
btrfs subvolume create /mnt/swap

btrfs filesystem mkswapfile --size 8g --uuid clear /mnt/swap/swapfile

swapon /mnt/swap/swapfile
```

## Installing Essential Packages

Use `Reflector` to get the mirrors you want:

```sh
reflector --verbose --country Canada,US --latest 30 --sort rate --save /etc/pacman.d/mirrorlist
```

Installing Linux and some important pacakges, you can choose [other kernels](https://wiki.archlinux.org/title/Kernel).

```sh
pacstrap -Ki /mnt base base-devel linux-zen linux-zen-headers linux-firmware neovim bash-completion btrfs-progs sof-firmware networkmanager
```

---

## Configuring the System

Generate a File System Table:

```sh
genfstab -U /mnt >> /mnt/etc/fstab
```

Check `/mnt/etc/fstab` to find any errors.

## Change root into the new system

```sh
arch-chroot /mnt
```

## Timezone

Create a Timezone symbolic link as we can't use `timedatectl` in chroot：

```sh
ln -sf /usr/share/zoneinfo/US/Pacific /etc/localtime
```

Set hardware clock from system clock:

```sh
hwclock --systohc
```

## Localization

Edit `/etc/locale.gen`, uncomment `en_US.UTF-8 UTF-8` and other need locales, then generate locales by running:

```sh
locale-gen
```

Create `/etc/locale.conf`:

```conf
LANG=en_US.UTF-8
```

## Network Configuration

create a hostname:

```sh
echo arch-pc > /etc/hostname
```

[Configure `/etc/hosts`](https://wiki.archlinux.org/title/Network_configuration#localhost_is_resolved_over_the_network):

```sh
127.0.0.1   localhost
::1         localhost
127.0.1.1   arch-pc.localdomain    arch-pc
```

