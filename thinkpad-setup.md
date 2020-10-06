This is unrelated to Linux From Scratch, but I figured I would put it into this
repository anyway.

# Thinkpad X1 Carbon setup

# Windows license key

Windows license key harvested via
https://answers.microsoft.com/en-us/windows/forum/windows_10-windows_install/product-key-for-lenovo-laptop-with-pre-installed/bd1b446d-b7e2-4814-805e-c0e02df1fbcf

> Press Windows key + X
> Click Command Prompt (admin)
> Enter the following command:
> wmic path SoftwareLicensingService get OA3xOriginalProductKey
> Hit Enter
> The product key will be revealed, copy the product key then enter it

# BIOS Setup

`Enter` when lenovo image appears to be given a prompt of other things to do.

Config - Network
- Wake ON LAN - Disabled
- Wake ON LAN from Dock - off
- Lenovo Cloud Services - off
- UEFI Wi-Fi Network Boot - off
- UEFI IPv* Network Stack - off
- Wireless Auto Disconnection - off
- MAC Address Pass Through - disabled

Config - Intel AMT
- Intel AMT Control - Disabled

Config - Power
- Sleep State - Linux

Security - Fingerprint
- Predesktop Authentication - off

Security - Security Chip
- Security Chip - off
- Physical Presence for Clear - off

Security - UEFI BIOS Update Option
- Flash BIOS Updating by End-Users - On
- Secure RollBack Prevention - Off
- Windows UEFI Firmware Update - Off

Security - Memory Protection
- Execution Prevention - Off

Security - Virtualization
- Kernel DMA Protection - On
- Intel Virtualization Technology - On (greyed out)
- Intel VT-d Feature - On (greyed out)
- Enhanced Windows Biometric Security - Off

Security - Absolute Persistence Module
- Absolute Persistence Module Activation - Permanently Disabled

Security - Secure Boot
- Secure Boot - Off

Security - Password
- Password at Unattended Boot - Off

Security - ThinkShield secure wipe
- ThinkShield secure wipe in App Menu - Off

Startup
- Boot
  - Use the super janky interface to put USB HDD at the top, then NVMe0
    second
- Network Boot - NVMe0 - (this is only for wake on lan, which is disabled, so
                          it shouldn't really matter)

# Arch install

Boot from USB.

Using a wired connection to the laptop.

## ssh

Start ssh server and set a root password:

```console
$ systemctl start ssh
$ passwd
```

Find the ip address with `ip a`.

`ssh` in from another machine to make copy pasting things easier.

```console
$ ssh -o UserKnownHostsFile=/dev/null root@192.168.1.151
```

## Disk partitioning

LVM on LUKS

https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS

Drive preparation

https://wiki.archlinux.org/index.php/Dm-crypt/Drive_preparation

NVME sanitization

https://wiki.archlinux.org/index.php/Solid_state_drive/Memory_cell_clearing#NVMe_drive

```console
root@archiso ~ # nvme id-ctrl /dev/nvme0 -H | grep "Format \|Crypto Erase\|Sanitize"
  [1:1] : 0x1   Format NVM Supported
  [29:29] : 0x1 No-Deallocate After Sanitize bit in Sanitize command Not Supported
    [2:2] : 0   Overwrite Sanitize Operation Not Supported
    [1:1] : 0x1 Block Erase Sanitize Operation Supported
    [0:0] : 0x1 Crypto Erase Sanitize Operation Supported
  [2:2] : 0x1   Crypto Erase Supported as part of Secure Erase
  [1:1] : 0     Crypto Erase Applies to Single Namespace(s)
  [0:0] : 0     Format Applies to Single Namespace(s)
```

Alrighty lets do a sanitize.

```console
root@archiso ~ # nvme sanitize /dev/nvme0 --sanact=4
NVMe status: ACCESS_DENIED: Access to the namespace and/or LBA range is denied due to lack of access rights(0x4286)
```

Hmm.

https://community.wd.com/t/sn750-cannot-format-using-the-nvme-command/254374/7

Checking BIOS. Nothing seems to stick out.

Found this which references a BIOS Secure Wipe option.
https://github.com/linux-nvme/nvme-cli/issues/412

Switching this BIOS option to On.

Security - ThinkShield secure wipe
- ThinkShield secure wipe in App Menu - On

Reboot -> F12 -> App Menu -> ThinkShield secure wipe -> ATA Cryptographic
Key Reset

That didn't do it.

Repeat but choose "ATA Secure Erase" this time.

Still getting ACCESS_DENIED.

Back to BIOS

Config - Power
- Sleep State - Linux

Still no go, giving up on the memory cell clearing. The ThinkShield secure wipe
is probably similar.

Back to drive preparation, following
https://wiki.archlinux.org/index.php/Dm-crypt/Drive_preparation#dm-crypt_wipe_on_an_empty_disk_or_partition


Definitely use `bs=1M` options to really speed up zeroing out of encrypted
container. 139 MB/s vs 1.4 GB/s :).

```console
$ dd if=/dev/zero of=/dev/mapper/to_be_wiped status=progress bs=1M
```

This took 13 minutes, not too bad.

Back to LVM on LUKS https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Preparing_the_disk_2

Something like:

```console
$ fdisk /dev/nvme0n1
n
+512M
t
1
n
(defaults all the way through)
w
```

Should end up looking like:

```console
$ root@archiso ~ # fdisk -l /dev/nvme0n1
Disk /dev/nvme0n1: 953.87 GiB, 1024209543168 bytes, 2000409264 sectors
Disk model: WDC PC SN730 SDBQNTY-1T00-1001
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: <REDACTED>

Device           Start        End    Sectors   Size Type
/dev/nvme0n1p1    2048    1050623    1048576   512M EFI System
/dev/nvme0n1p2 1050624 2000409230 1999358607 953.4G Linux filesystem
```

Continuing along, creating lvm partitions and formating and mounting things:

```console
root@archiso ~ # lsblk /dev/nvme0n1
NAME            MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
nvme0n1         259:0    0 953.9G  0 disk
├─nvme0n1p1     259:3    0   512M  0 part  /mnt/boot
└─nvme0n1p2     259:4    0 953.4G  0 part
  └─cryptlvm    254:0    0 953.4G  0 crypt
    ├─a-os1     254:1    0    50G  0 lvm   /mnt
    ├─a-os2     254:2    0    50G  0 lvm
    ├─a-os3     254:3    0    50G  0 lvm
    ├─a-swap    254:4    0    16G  0 lvm   [SWAP]
    └─a-storage 254:5    0 787.4G  0 lvm   /mnt/storage
```

`pacstrap` with some things we'll need:

```
pacstrap /mnt base linux linux-firmware lvm2 vim tmux iproute2 e2fsprogs dosfstools man-db man-pages texinfo
```


Now edit `/mnt/etc/mkinitcpio.conf`
https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Configuring_mkinitcpio_2

```
HOOKS=(base udev autodetect keyboard keymap modconf block encrypt lvm2 filesystems fsck)
```

Now back to the main setup
https://wiki.archlinux.org/index.php/Installation_guide#Chroot

```console
# pacman -S networkmanager
# systemctl enable NetworkManager
```

Running `mkinitcpio` lists some missing firmware:

```
==> WARNING: Possibly missing firmware for module: wd719x
==> WARNING: Possibly missing firmware for module: aic94xx
```

Let's get `yay` so we can easily install them from the AUR.

https://github.com/Jguer/yay

```
# pacman -S base-devel git
# mkdir -p /storage/src
# cd /storage/src
```

Now we actually need a user to do `makepkg` properly so lets get that set up.

sudo config:

```console
# visudo
(uncomment sudo group line)
# groupadd sudo
```

Make and configure a user

```console
# useradd -m -G sudo -s /bin/bash conky
# passwd conky
# su conky
$ sudo chown -R root:sudo /storage
$ sudo chmod -R 775 /storage
```

Follow install https://github.com/Jguer/yay#installation

Now we can get those missing firmware packages straightened out:

```
[conky@archiso yay]$ yay -S --search wd719x
aur/wd719x-firmware 1-7 (+225 5.33)
    Driver for Western Digital WD7193, WD7197 and WD7296 SCSI cards
[conky@archiso yay]$ yay -S --search aic94xx
aur/aic94xx-firmware 30-9 (+278 5.59)
    Adaptec SAS 44300, 48300, 58300 Sequencer Firmware for AIC94xx driver
[conky@archiso yay]$ yay -S wd719x-firmware aic94xx-firmware
```

```
$ sudo mkinitcpio -P
```

EFI Boot loader
https://wiki.archlinux.org/index.php/Systemd-boot#Installing_the_EFI_boot_manager

```console
$ sudo pacman -S efibootmgr intel-ucode
$ sudo bootctl install
```

`/boot/loader/loader.conf`:

```
# man loader.conf
default arch.conf
timeout 30

# standard UEFI 80x25 mode
console-mode 0
```

`/boot/loader/entries/arch.conf`:

Find UUID with `sudo blkid | grep crypto_LUKS`

```
title arch
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options cryptdevice=UUID=<UUID-here>:cryptlvm root=/dev/mapper/a-os1 resume=/dev/mapper/a-swap
```

Reboot.

Enter the luks passphrase and you should be good to go.

# Further setup

```console
$ sudo pacman -S alacritty sway firefox openssh linux-firmware
```

And a bunch of other stuff...

Sound:
`pulseaudio`
`sof-firmware`

# Hibernation

Battery said 98% (which it always says...)
Closed lid at 9:48 and unplugged power. We'll see how much power it uses.
89% 24 hours later - great!!

# Clipboard

Normal `vim` doesn't seem to support `wl-clipboard` so I'm going to finally
move to `neovim` which apparently supports it out of the box.

https://github.com/vim/vim/issues/5157
https://github.com/neovim/neovim/pull/9229

Followed the guide to migrate
https://neovim.io/doc/user/nvim.html#nvim-from-vim
https://neovim.io/doc/user/provider.html#provider-clipboard

Added `set clipboard+=unnamedplus` to my `~/.vimrc` and it works!


