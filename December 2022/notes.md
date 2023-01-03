#  December 2022 Asahi Notes

This is a 'diff' update of the original notes accounting for the changes that have happened in approximately the last two months. 

We will cover:

1. The LUKS setup, once again.
    * **NOTE:** the end of the text contains IMPORTANT information in regards to there being a **PKGBUILD** you can use to set up the systemd stuff necessary for your LUKS to work. There is still manual configuration required which is documented in the notes of my asahi-scripts fork repo.
2. Some stuff related to apparmor.d.
3. Next goals.

If you build according to the last note, you will BREAK YOUR SYSTEM.

## Links

1. [LUKS](#LUKS)
2. [apparmor.d](#apparmord)
3. [Next goals](#Next)

# LUKS

## Updating from October to December

The below should similarly to the October notes be considered and approximate workflow for getting LUKS up and running, copying and editing info from the last notes is cumbersome so do not consider all edge cases covered and instead:

- Think for yourself how an install should proceed.
- Then go through below notes to check where you can fill in the gaps. READ EVERYTHING before doing ANYTHING.
- Also note that bold means that I added something on to the last notes, strikethrough and _absence_ is removing for all points from old note **BELOW _NOTES_**.

### Preparing the environment:

The user may decide whether they want free space left or not at the end of the installation, or rather which of the following 2 options they would like to use:

- Configure LUKS setup from Asahi install on disk. **(Point 1: please use this install method, instead of using the removable media, it should be way easier to install stuff, since you do not have to fumble around with a removable drive and can instead just use an Asahi which you can keep around to troubleshoot your info later on)**.

    - You really only need a minimal base install with approx 50GB of extra space, which you can overwrite later, meaning, take it from your Mac OS partition. The only reason you would be using the USB, then, is if you really want to "extremely" partition your device such that MacOS is minimal size. But this will make harder to debug stuff, and you anyways probably need MacOS for some stuff that simply doesn't work currently on Asahi.

- Configure from removable media (has to be UEFI bootable).

I suppose a clean install could be made using only m1n1 (virtualize) but from my perspective it is simpler to choose one of the above, and from them, removable media.  We will install the Asahi Desktop (Plasma) image with the installer and then set things up from a removable drive since we want to make things easier for us (we will just need to delete more at the end).

1. Bookmark or download this webpage https://asahilinux.org/2022/03/asahi-linux-alpha-release/. Run the mentioned script or download check run the installer depending on your security needs.
    - Partition the disks so the new one has something like 80 GB and use 100% of it for the OS install. Install Asahi-Desktop. **Follow the installer instructions (this takes a while)**. **You can keep this OS around and use it as mentioned above. Just make sure that your partitions are nicely organized.**
    - We will not be returning immediately to mac os, you can stay inside and configure your KDE (if you installed with a desktop environment) quickly for ease of use, ~~but note that this OS is throaway.~~
    - Do get internet running.
    - ~~Once inside asahi desktop, open the KONSOLE app~~ just use any terminal

2. If not running KDE, you will need the user and root passwords and as mentioned above you will have to have internet up and running
    - **THE DEFAULT** passwords are: alarm - alarm, root - root, so the username IS the password
    - You will log into root and immediately configure your new partitions for which you should have freed space from your MacOS, the thing is that this is pretty direct because we can immediately do this from a minimal image install, you can also choose to _first_ do point 4 and then 3, it is the same, the basic point is that you slowly do stuff.

3. Prepare new partitions for the "actual install" ESP and FS.
    - You can use the commands from above, but edit them to proper sizes to make the ESP identical to the side install:
      * You will have to check exactly how to simplify below 2 commands, because I've formatted via the USB install, in any case it is trivial to find this out
      * `parted --script --align=optimal mkpart ESP fat32 ESP_SIZE_LOWER ESP_SIZE_UPPER` - ESP
      * `parted --script --align=optimal mkpart FS ext4 FS_SIZE_LOWER FS_SIZE_UPPER` - FS
      * format the partitions.
      * `mkfs.vfat -n ASAHI /dev/<ESP_PART_DEVICE>` - ESP
      * `mkfs.ext4 /dev/<FS_PART_DEVICE>` - FS
    - Now get the ESP and FS UUID via `lsblk -f` and move on to the next step.
    - Mount the ESP on /mnt/.

So 2 explanations going forward on topic of which files to use to actually create your asahi distro by copying them into the above created parts.

The following step has worked for my install, meaning it is the safe way to go. The following step is essentially building the images from script and copying them into the necessary partitions _for usb_, as it was originally written. You can instead copy onto the created full OS partitions immediately.

You can also instead opt to copy the files (once LUKS encrypted) from your side install onto the one you want to use, though I am not sure of the consequences and don't want to be testing it out right now. It should work, but with tweaks, at least considering partition differences. Again, I still think that building the base image (properly configured) is safer.

4. Move to desktop **or some work folder** and `git clone https://github.com/AsahiLinux/asahi-alarm-builder.git`
    - check the scripts out and read out what they do so you know your initial minimal image config, it is short
    - vim into the `build.sh` script and first set ROOT_UUID and EFI_UUID **- those should be from the disk partitions**
    - remove building UEFI minimal and plasma (desktop) images at the bottom, we only want BASE (minimal), don't remove `run_scripts base`
    - `./build.sh` inside `asahi-alarm-builder` should build the image _relatively_ quickly
    - from `asahi-alarm-builder`, `cd images/asahi-base/ && mkdir mproot`
\

5. Setting up LUKS:
    - since we are doing a m1n1 stage 2 direct boot, we are going to be making a LUKS partition which supports hooks such as encrypt or systemd-encrypt
      * for "plain mode" encryption, you would require UBOOT or GRUB to boot a kernel image + initramfs from a USB to unlock the plain partition, unless you are using the plain encrypted partition next to an open arch install
      * i also suggest not concerning oneself with prewiping data on disk with random data because of how SSDs work (also if there are not partitions TRIM is going to remove everything not registered a partition) - if plausible deniability is necessary only and then i would suggest using a different device
    - the next step is in fact reading through the cryptsetup manual, specifically `man cryptsetup-luksFormat` in most cases, and find the setup that you want, learning how to set it up, this is not covered because it follows the exact instructions on the arch linux wiki -> https://wiki.archlinux.org/title/Dm-crypt
   - now setup the partition with cryptsetup or any other tool according to your needs

6. **This is where** you are copying data if running from a minimal base install
    - mount the ESP partition (this one should NOT be encrypted) onto `/mnt/`
    - remember to go into the work directory in which we built the asahi image
    - find `/EFI/` in `/esp/` (esp being in `/asahi-base/`) and copy it (+ contents) into `/mnt/` (ESP)
    - `umount /mnt/` and `mount /dev/<FS_PART_DEVICE> /mnt/` (FS)
    - `mount root.img mproot`
    - `mkdir -p mproot/usr/share/kbd/keymaps && dumpkeys > mproot/usr/share/kbd/keymaps/defkeymap.map`
    - try `chroot mproot` and then `passwd` to set the root password, the normal user is alarm with password alarm
    - `exit` to quit chroot
    - `cp -r mproot/* /mnt`
    - if there is no /mnt/usr/share/kbd/keymaps: `mkdir -p /usr/share/kbd/keymaps`
    - umpkeys > /mnt/usr/share/kbd/keymaps/defkeymap.map` - you should also copy these keys later into your LUKS install
    - `umount -a`

7. Now that the encrypted drive is open **and there is data inside** there are still numerous things to configure:
    - first copy all of the data on your root filesystem parititon on the removable drive onto the fs partition on-device, we did not even test the encrypted partition so this will be a first goal
    - note that the dev/sda1 (or however your ESP/EFI partition on removable is called) should not be mounted on /boot/ in the removable drive chroot
    - you should also in forward pacstrap the fs partition with networkmanager because iwctl did not work for my install, by itself at least -> there is probably some configuring nm does which solves the issue which i did not look into exactly, which weird because nm uses iwd as a backend in asahi linux configurations
    - now as part of all folders on the actual device, you should also have a /boot/ folder that should contains an intiramfs, initramfs fallback and vmlinuz-linux-asahi
    - the same folder should have an efi directory, if not, make one
    - now, you should configure fstab to mount your apple device ESP/EFI partition onto /boot/efi, making /boot/efi/m1n1 available, configure this in your fstab

THE BELOW, is probably the most important thing for which you came for. Most of it is now CROSSED out. Because it is simply part of the following repository -> https://github.com/b-crumb/asahi-scripts and https://github.com/b-crumb/PKGBUILDs.asahi-scripts and https://github.com/b-crumb/asahi-infra. This is how I am CURRENTLY building my distro, which enables me to use LUKS, this is because the former method of only using systemd is **BROKEN**.

The thing you have to essentially do is install MY PKGBUILD by cloning the PKGBUILDs.asahi-scripts repository and also probably take a look at the README in MY (b-crumbs) asahi-scripts repo which does contain some unnecessary information which is handled by the PKGBUILD but, also contains others which are REQUIRED.

Meaning, everything missing is added auto, everything not missing is updated.

#### hooks (in /etc/mkinitcpio.conf)
`HOOKS=(base systemd sd-asahi autodetect keyboard sd-vconsole modconf block sd-encrypt filesystems fsck)`

#### kernel args (see below)
`rd.luks.name=[ENTER UUID HERE]=root rd.luks.options=allow-discards,no-read-workqueue,no-write-workqueue root=/dev/mapper/root rootwait rw quiet splash`
rd.luks.* -> sd-encrypt kernel args

root -> root partition path when opened by hook scripts after password input

rootwait -> wait indefinitely on root filesystem ... this arg might be a problem if some boot script exits during startup and leaves you in an indefinite black screen + underscore state

rw .. quiet ..splash - well known

**WHEN BOOTING THE FIRST FEW TIMES** definitely also add: earlycon debug - such that you have debug output at start

#### simple script to _build_ m1n1 bin (in /etc/default/update-m1n1 , see NOTE)
```bash
#!/bin/bash
echo "zipping vmlinuz-linux-asahi..."

gzip -fqk /boot/vmlinuz-linux-asahi # or with -edge if using the edge kernel

echo "...zipped to /boot/vmlinuz-linux-asahi.gz"

NAME="$(date +"%Y%j%H%M")-sd-asahi.bin"

# add a date identifier
# note that only the file named `boot.bin` will be booted

cat /lib/asahi-boot/m1n1.bin \
        <(echo 'chosen.bootargs=rd.luks.name=<ROOT PART UUID HERE!!>=root rd.luks.options=allow-discards,no-read-workqueue,no-write-workqueue root=/dev/mapper/root apparmor.debug=1 rootwait rw quiet splash') \
        /lib/modules/*-edge-ARCH/dtbs/*.dtb \
        /boot/initramfs-linux-sd-asahi.img \
        /boot/vmlinuz-linux-asahi.gz \
        >/boot/efi/m1n1/"$NAME"

echo "New m1n1 binary built: $NAME"

# set so /usr/bin/update-m1n1 does not continue to default process

M1N1_UPDATE_DISABLED=1
```

Note that you need to `mv` the `$NAME` binary manually to `boot.bin` once ready.

If you are booted into a kernel with a /proc/cmdline which you are fine with, then also:

```bash
cat /lib/asahi-boot/m1n1.bin \
	<(echo "chosen.bootargs=$(cat /proc/cmdline)") \
    /lib/modules/*-edge-ARCH/dtbs/*.dtb \
    /boot/initramfs-linux-sd-asahi.img \
    /boot/vmlinuz-linux-asahi.gz \
    >/boot/efi/m1n1/"$NAME"
```

works

#### fstab (in /etc/fstab)
```
UUID=[UUID HERE] / ext4 rw,relatime,x-systemd.growfs 0 1
UUID=[SHORT ESP UUID HERE eg XXXX-XXXX] /boot/efi vfat rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro    0 2
/swapfile none swap defaults 0 0
```

8. continuing
	- With most of this stuff you should be able to set up a bootable asahi install. Note that, you will essentially first configure (some) of these files for your install, after which you should regenerate your initramfs - vmlinuz not - and only then build the m1n1 bin similarly to above build script. _boot.bin_, again, is the entrypoint.
	- After all of this is done, you want to probably connect to the internet with networkmanager, `pacman -Syyu` to get all of the latest packages and see if anything breaks, create a new user with a home directory with the /etc/skel files offered by asahi
	- You might also want to continue on following the asahi-alarm-builder scripts for plasma which set up a desktop environment - read them through and execute them
		* note that you probably do not need scripts such as calamares, especially if they don't work first boot for you as they didn't for me, it is just a configuration script
		* wayland has been working for me better than x11 - don't go too high yet with dpi in kde config since we still need gpu accel support

# apparmor.d

I have started patching apparmor.d such that you can have a safe apparmor setup on your mac in the repository https://github.com/b-crumb/apparmor.d-asahi-patches.

Instructions are inside of the README. The `makefile install-disabled` install BOOTS, at least on my device, and allows you to work _relatively_ normally on the device, as far as I know, for simple usecases. Brightness also works.

# Next

The actual next goal is TOSSING OUT systemd and instead discovering why the BASE hook doesn't work with a luks encryption, fix that, and then never deviate from upstream again.