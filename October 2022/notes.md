# October 2022 Luks Asahi Notes (non-encrypted boot, encrypted root)

## Asahi Simple Overview

This overview is necessary in order to understand what we will be doing without going into details. 

The core of Asahi is the m1n1 bootloader (a bootloader receives initial instructions from integrated systems and performs steps to prepare inputs + boot an OS or chainload another bootloader, simplified) + a modified Arch Linux ARM64 kernel (ALARM), which works on Apple Silicon machines, unlike the vanilla version.

The m1n1 bootloader is a bridge between the Silicon boot process (XNU boot protocol, XNU being a kernel (the kernel which is part of MacOS) ), which is only designed to load XNU kernel images, and the Linux boot ecosystem, which is such that it can't be loaded by former protocol. 

XNU boot protocol has _no_ external storage boot support.

So m1n1 is a binary compatible with the XNU boot protocol. It essentially supports loading ARM64 Linux style kernel images, bootloaders via device trees, or chainloading another m1n1 instance.

The boot process for an Asahi-(Base,Plasma) install:

XNU boot selects chosen XNU kernel which is part of an OS container (which is a collection of disk volumes associated with one OS) -> m1n1 stage 1 runs and does stuff, calls stage 2 -> mini stage 2 runs and does stuff, loads bootloader -> U-Boot runs as the first possible bootloader which supports booting from external storage, but can also chainload GRUB etc. -> GRUB runs -> Linux

m1n1 stage 1 and before is essentially part of the Apple Signed process where the stage 1 binary is the first thing we can customize, but grouping it together with former (OS container partition also considered, since m1n1 stage 1 resides in OS volume while stage 2 just resides in separate ESP partition) is useful for separation of concerns, since you only need to specify ESP UUID once and then just configure rest of m1n1 in it.

A shorter boot process could also look like:

XNU -> m1n1 1 -> m1n1 2 -> Linux

or the minimal:

XNU -> m1n1 1 -> Linux

Note that _each_ m1n1 stage 1 binary resides in its associated APFS container..volume, so, there can be multiple boot processes available, given enough space.

We will concentrate on the former boot process with 2 m1n1 stages.

## LUKS Setup (non-encrypted boot, encrypted root)

Our disk will have the following partitions when we are done with our process (most likely out of order):

#### APPLE

Apple APFS ISC (...)

APFS MacOS Container (...)

Apple APFS Recovery (...)

#### ASAHI

ASAHI APFS CONTAINER (APFS) (
volumes,
**m1n1 stage 1**
)

ESP (**vfat**) (
**m1n1 stage 2 (which literally has concatenated initramfs, linux kernel, device trees, kernel params)**
)

CRYPTO LUKS (
    ROOT (**ext4**) (
        root
    )
)

Boot process:

XNU -> m1n1 stage 1 -> m1n1 stage 2 -> Linux kernel (decrypts device, mounts root...)

## Notes

This is not specifically a full install guide but just the bridge steps necessary to set the thing up. The unmentioned rest follows a typical Arch install noting that notes will cover a decent amount of it.

### Preparing the environment:

The user may decide whether they want free space left or not at the end of the installation, or rather which of the following 2 options they would like to use:

- Configure LUKS setup from Asahi install on disk.
- Configure from removable media (has to be UEFI bootable).

I suppose a clean install could be made using only m1n1 (virtualize) but from my perspective it is simpler to choose one of the above, and from them, removable media.  We will install the Asahi Desktop (Plasma) image with the installer and then set things up from a removable drive since we want to make things easier for us (we will just need to delete more at the end).

1. Bookmark or download this webpage https://asahilinux.org/2022/03/asahi-linux-alpha-release/. Run the mentioned script or download check run the installer depending on your security needs.
    - Partition the disks so the new one has something like 80 GB and use 100% of it for the OS install. Install Asahi-Desktop. **Follow the installer instructions (this takes a while)**.
    - We will not be returning immediately to mac os, you can stay inside and configure your kde quickly for ease of use, but note that this OS is throaway.
    - Do get internet running.
    - Once inside asahi desktop, open the KONSOLE app
2. Sudo has no password, configure something easy like "aa", then insert the removable media into the device and mount it.
    - enter root su
    - We want to format the USB to become UEFI bootable, so it needs to be GPT formatted, with an ESP and root part.
    - Do this with parted, or fdisk. I was using parted. Get it with `pacman -S parted`. 
    - `lsblk -f / lsblk -o name,uuid,partuuid` will be used.
    - First set the format right, for this run the following scripts (assuming device is in format /dev/sda):
      * `parted /dev/sda mklabel gpt` (will delete all data on removable)
      * `parted --script --align=optimal mkpart ESP fat32 1MiB 551MiB` - ESP
      * `parted --script --align=optimal mkpart FS ext4 552MiB 100%` - FS
      * `parted --script /dev/sda set 1 boot on`
      * Now format the partitions.
      * `mkfs.vfat -n ASAHI /dev/sda1` - ESP
      * `mkfs.ext4 /dev/sda2` - FS
    - Now get the ESP and FS UUID via `lsblk -f` and move on to the next step.
    - Mount the ESP on /mnt/.
3. Move to desktop and `git clone https://github.com/AsahiLinux/asahi-alarm-builder.git`
    - check the scripts out and read out what they do so you know your initial minimal image config, it is short
    - vim into the `build.sh` script and first set ROOT_UUID and EFI_UUID to the UUIDs noted
    - remove building UEFI minimal and plasma (desktop) images at the bottom, we only want BASE (minimal), don't remove `run_scripts base`
    - `./build.sh` inside `asahi-alarm-builder` should build the image _relatively_ quickly
    - from `asahi-alarm-builder`, `cd images/asahi-base/ && mkdir mproot`
    - find `/EFI/` in `/esp/` (esp being in `/asahi-base/`) and copy it (+ contents) into `/mnt/` (ESP)
    - `umount /mnt/` and `mount /dev/sda2 /mnt/` (FS)
    - `mount root.img mproot`
    - try `chroot mproot` and then `passwd` to set the root password, the normal user is alarm with password alarm
    - `exit` to quit chroot
    - `cp -r mproot/* /mnt/`
    - if there is no /mnt/usr/share/kbd/keymaps: `mkdir -p /usr/share/kbd/keymaps`
    - `dumpkeys > /mnt/usr/share/kbd/keymaps/defkeymap.map` (this sets up def keys to autoload - will be available with proper hooks for decryption password - if you will need to blind enter with en_US keymap)
    - `umount -a`
4. Reboot the device and let it boot into the splash screen where it shows asahi logo and u-boot. Notice that console on the left side of the screen. The USB must be plugged in.
    - when it starts counting down quickly hit any key.
    - `run bootcmd_usb0`
    - you will be in grub, now hit the up arrow quickly, or down, to stop the countdown, but stay on Asahi Linux
    - IF chroot didn't work do or you didn't execute following steps:
        - hit `e`, then find kernel parameters and add `init=/bin/bash` to them, and then hit boot key combo (look at the bottom)
        - when you enter the console: `passwd`, set root password, reboot and goto 4 (above) until you don't reach "if clause" again
    - else:
        - hit `enter` for Asahi Linux
    - login root
5. You should be in the minimal OS now logged in as root. If this works we come to our next fork:
    - you can either choose to make a test luks partition so we test m1n1 stage 2 booting etc or we can actually start configuring the real thing
    - in any case, you will need a partition for the filesystem, one way you can gain it being either claiming free space (making a partition from sector to sector), or shinking an apfs container (diskutil handles this well, but as noted from asahi docs use the CLI utility and read it well through, there is a lot of options)
    - in any case, what you will be doing when setting up the ACTUAL system is wiping your asahi installs and then partition anew again (https://github.com/AsahiLinux/docs/wiki/Partitioning-cheatsheet - `curl -L https://alx.sh/wipe-linux | sh` - BE CAREFUL WITH THIS - READ THE CHEATSHEET), such that the install is easier and cleaner, and this include cleaning https://github.com/AsahiLinux/asahi-installer/blob/main/tools/cleanbp.sh boot policy files (the wipe script uses this so this is only if you are doing it manually!), and then installing a minimal asahi the filesystem partition of which you will overwrite including the m1n1 files such that they point at the luks partition you will set up
    - since we are doing a m1n1 stage 2 direct boot, we are going to be making a LUKS partition which supports hooks such as encrypt or systemd-encrypt
      * for "plain mode" encryption, you would require UBOOT or GRUB to boot a kernel image + initramfs from a USB to unlock the plain partition, unless you are using the plain encrypted partition next to an open arch install
      * i also suggest not concerning oneself with prewiping data on disk with random data because of how SSDs work (also if there are not partitions TRIM is going to remove everything not registered a partition) - if plausible deniability is necessary only and then i would suggest using a different device
    - the next step is in fact reading through the cryptsetup manual, specifically `man cryptsetup-luksFormat` in most cases, and find the setup that you want, learning how to set it up, this is not covered because it follows the exact instructions on the arch linux wiki -> https://wiki.archlinux.org/title/Dm-crypt
    - now setup the partition with cryptsetup or any other tool according to your needs, the next step covers POST-ENCRYPT AND OPENING THE PARTITION
6. Now that the encrypted drive is open there are still numerous things to configure:
    - first copy all of the data on your root filesystem parititon on the removable drive onto the fs partition on-device, we did not even test the encrypted partition so this will be a first goal
    - note that the dev/sda1 (or however your ESP/EFI partition on removable is called) should not be mounted on /boot/ in the removable drive chroot
    - you should also in forward pacstrap the fs partition with networkmanager because iwctl did not work for my install, by itself at least -> there is probably some configuring nm does which solves the issue which i did not look into exactly, which weird because nm uses iwd as a backend in asahi linux configurations
    - now as part of all folders on the actual device, you should also have a /boot/ folder that should contains an intiramfs, initramfs fallback and vmlinuz-linux-asahi
    - the same folder should have an efi directory, if not, make one
    - now, you should configure fstab to mount your apple device ESP/EFI partition onto /boot/efi, making /boot/efi/m1n1 available, configure this in your fstab
    - I will now post the most important configuration I am using for reference when building the system:

#### hooks (in /etc/mkinitcpio.conf)
`HOOKS=(base asahi systemd autodetect keyboard sd-vconsole modconf block sd-encrypt filesystems fsck)`

encrypt - this hook is to be used if you would not like use sd-encrypt

#### modified mkinitcpio.d/linux-asahi.preset
```bash
# mkinitcpio preset file for the 'linux-asahi' package

ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux-asahi"

PRESETS=('default' 'fallback')

#default_config="/etc/mkinitcpio.conf"
default_image="/boot/initramfs-linux-asahi.img.gz"
#default_options=""

#fallback_config="/etc/mkinitcpio.conf"
fallback_image="/boot/initramfs-linux-asahi-fallback.img.gz"
fallback_options="-S autodetect"
```
explicit .gz for initramfs

#### kernel args (see below)
`rd.luks.name=[ENTER UUID HERE]=root rd.luks.options=allow-discards,no-read-workqueue,no-write-workqueue lsm=landlock,lockdown,yama,integrity,apparmor,bpf root=/dev/mapper/root rootwait rw quiet splash`
rd.luks.* -> sd-encrypt kernel args

lsm -> linux security modulles - newer Asahi kernels should have apparmor by default enabled.

root -> root partition path when opened by hook scripts after password input

rootwait -> wait indefinitely on root filesystem ... this arg might be a problem if some boot script exits during startup and leaves you in an indefinite black screen + underscore state

rw .. quiet ..splash - well known

**WHEN BOOTING THE FIRST FEW TIMES** definitely also add: earlycon debug - such that you have debug output at start

args such as cryptdevice are to be used if encrypt is used instead of sd-encrypt

#### simple script to _build_ m1n1 bin (in /etc/default/update-m1n1 , see NOTE)
```bash
#!/bin/bash
echo "zipping vmlinuz-linux-asahi..."

gzip -fk /boot/vmlinuz-linux-asahi

echo "...zipped"

cat /lib/asahi-boot/m1n1.bin \
	<(echo 'rd.luks.name=[ENTER UUID HERE]=root rd.luks.options=allow-discards,no-read-workqueue,no-write-workqueue lsm=landlock,lockdown,yama,integrity,apparmor,bpf root=/dev/mapper/root rootwait rw quiet splash') \
	/lib/modules/*-ARCH/dtbs/*.dtb \
	/boot/initramfs-linux-asahi.img.gz \
	/boot/vmlinuz-linux-asahi.gz \
	>/boot/efi/m1n1/boot.bin

echo "New boot.bin (over)written."

M1N1_UPDATED=1
```
NOTE: this is an _old_ script I used for building the m1n1 bin - it should work correctly given you are chrooted in and mkinitcpio.d/linux-asahi.preset is used - otherwise make initramfs without .gz suffix
Remember that m1n1 requires a gzipped kernel image too.

I actually never wrote directly to _boot.bin_ which is the stage 2 entrypoint, and rather always wrote another file and backed up the old one.

If you are booted into a kernel with a /proc/cmdline which you are fine with, then also:

```bash
cat /lib/asahi-boot/m1n1.bin \
	<(echo "chosen.bootargs=$(cat /proc/cmdline)") \
	/lib/modules/*-ARCH/dtbs/*.dtb \
	/boot/initramfs-linux-asahi.img.gz \
	/boot/vmlinuz-linux-asahi.gz \
	>/boot/efi/m1n1/boot.bin
```

works

#### fstab (in /etc/fstab)
```
UUID=[UUID HERE] / ext4 rw,relatime,x-systemd.growfs 0 1
UUID=[SHORT ESP UUID HERE eg XXXX-XXXX] /boot/efi vfat rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro    0 2
/swapfile none swap defaults 0 0
```



7. continuing
	- With most of this stuff you should be able to set up a bootable asahi install. Note that, you will essentially first configure (some) of these files for your install, after which you should regenerate your initramfs - vmlinuz not - and only then build the m1n1 bin similarly to above build script. _boot.bin_, again, is the entrypoint.
	- After all of this is done, you want to probably connect to the internet with networkmanager, `pacman -Syyu` to get all of the latest packages and see if anything breaks, create a new user with a home directory with the /etc/skel files offered by asahi
	- You might also want to continue on following the asahi-alarm-builder scripts for plasma which set up a desktop environment - read them through and execute them
		* note that you probably do not need scripts such as calamares, especially if they don't work first boot for you as they didn't for me, it is just a configuration script
		* wayland has been working for me better than x11 - don't go too high yet with dpi in kde config since we still need gpu accel support
