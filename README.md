# Archlinux on Macbook Pro 11,1 (Mid 2014)

## Basic info
I wrote this document, because I have some troubles when installing Archlinux to my Macbook Pro. I spent long time solving some crucial issues, googling, ... This might help other people to spend less time if they want to do same mad things as me. But it's primary purpose is I want to have some "TODO list" for Linux Installation on Macbook. I dont want to end up solving same issues near future (when I had to reinstall, or install another Macbook).

This document is based on many other sources. My primary source when installing was [Macbook Pro ArchWiki article](https://wiki.archlinux.org/index.php/MacBookPro11,x), but this does not cover all issues issues I experienced. So other sources are various forums and bugreports with workarounds explained. Sadly I do not remember all my sources. But I will try to link all my known sources to in document.

Also this documents does not explain all steps of ArchLinux setup. You can find these informations elsewhere. Documents where detailed explanation can be found are linked (for case you are not familiar with it).

Document may contain some mistakes, caused by my own improper deep knowledge of selected problematic. For that case, feel free to contact me. I'll be glad if you fix my mistakes.

This might work also for other Macbook models as well (probably 2013 and 2014 models).

## Initial situation
I was happy user of macOS for many years (since 2008). For me this system was ideal combination between Windows (many fun applications) and Unix systems (they do what I want, have great tools for scripting/developing, some great apps I need, but sucks when it comes to fun - games, etc...). Unfortunately Apple decided to make macOS unconfortable for everyday use. They started removing good features, adding features which tries to take control over my system and data, the system just became very unfriendly and "out of control". It ended up I must stop using all Apple apps (Photos, iTunes, iWork, replacing them with [Swinsian](http://swinsian.com/) and Libre Office) and fight the system to make it to do what I want, because OS had it's own opinion what I want to do.

So I finally managed to do big step: Remove macOS, install some better OS. I did not want to change my HW now, just because I like it, so it is little bit more challenging than installing on regular laptop. My final choice was [ArchLinux](https://www.archlinux.org/) because it is very customisable and very friendly when it comes to solving some "special" problems (and because I have a "special" hardware, this feature proved as crucial). I have tried other Linux distributions. But they were not so flexible, some of them very buggy (for example ElementaryOS), at least on my Macbook.

I have also one bootcamp partition on SSD with Windows. So this document describes dual boot setup with Windows. Actually this is funny, because I have only two games on Windows partition, nothing more.

## Preparation
Before we proceed with destroying macOS and installing Linux, I suggest few things:
- Backup your display color profile from `/Library/ColorSync/Profiles/Displays/` in macOS to some flashdrive
- Boot some linux flashdrive and backup content of your EFI system partition. Just mount it as fat32 partition and get all data to flashdrive. This partition should contain firmware for your Macbook and also Windows loader to boot BOOTCAMP partition.

When you are ready, boot ArchLinux installation flashdrive or some other linux live distro to destroy macOS partitions and create partitions for future Linux installation. For example my setup:

|                                            SSD                                                  |
|-----------|-------|-----------------------------------------------------------------------------|
| /dev/sda1 |  1GB  | LABEL=BOOT, fs=fat32, ESP, future mountpoint = /boot                        |
| /dev/sda2 |  17GB | LABEL=SWAP, fs=swap, future mountpoint = swap, will be used for hybernation |
| /dev/sda3 |  173GB  | LABEL=ROOT, fs=ext4, future mountpoint = /                                  |
| /dev/sda3 |  60GB | LABEL=BOOTCAMP, fs=ntfs, Windows partition                                  |

**Beware when creating partition for /boot. It mmust be set to type=ESP (EF00).**

I made it 1 GB large, because this is where kernel(s) and initramdisk(s) will be placed. When you are testing multiple kernels (which I was) it is useful to have bigger /boot, like this.

## Base installation
Just create installation media like usual. Follow [ArchWiki](https://wiki.archlinux.org/index.php/USB_flash_installation_media).
Alternatively if you are lazy just like me and dd in your macOS is acting strange, you can use [Etcher](https://etcher.io/) to write ISO to USB drive. It just works!

When you start ArchLinux from flashdrive, make console text readable on retina display: `setfont sun12x22`

Then proceed with instalation as [Installation guide](https://wiki.archlinux.org/index.php/Installation_guide) says. I used thunderbolt ethernet adapter for installation, but there are other methods described on [wiki](https://wiki.archlinux.org/index.php/MacBookPro11,x).

### Mounting partitions and generating /etc/fstab
When you are done with installation before generating fstab and chrooting to installed system, doublecheck if you mounted all your partitions, including your EFI partition (/boot). Then you can generate fstab using:

`genfstab -L /mnt >> /mnt/etc/fstab` (if you are using partition labels like me)

Then chroot into installed system and continue with setup using [Installation Guide](https://wiki.archlinux.org/index.php/Installation_guide).

### Bootloader setup
When you have don everything you want and reached boot loader section, you can use same setup as me. I used systemd-boot, since it is easy (very readable configuration file) and do not require any additional packages.

Since we have our ESP mounted in `/boot` we can just run `bootctl install`.

It is good idea to install [Intel microcode updates](https://wiki.archlinux.org/index.php/Microcode#Enabling_Intel_microcode_updates) with
`pacman -S intel-ucode`. We will enable microcode updates in next step when creating configuration files for systemd-boot.

Then we create configuration file for loader (adjust all configuration to your needs) in `/boot/loader/loader.conf`:
<pre>
default  archlinux-vanilla
timeout  4
editor   0
</pre>

And configuration to launch ArchLinux with vanilla kernel in `/boot/loader/entries/archlinux-vanilla.conf`:
<pre>
title          Arch Linux (vanilla)
linux          /vmlinuz-linux
initrd         /intel-ucode.img
initrd         /initramfs-linux.img
options        root=PARTLABEL=ROOT rw
</pre>

I also preffer to have configuration for booting to textmode (in case I broke my graphics mode) in `/boot/loader/entries/archlinux-textmode.conf`:
<pre>
title          Arch Linux (textmode)
linux          /vmlinuz-linux
initrd         /intel-ucode.img
initrd         /initramfs-linux.img
options        root=PARTLABEL=ROOT rw 3
</pre>

Now we finished setting bootloader.

You might want later to enable automatic updates of systemd-boot. I suggest to install [systemd-boot-pacman-hook](https://aur.archlinux.org/packages/systemd-boot-pacman-hook/) later if you do not want to care about systemd-boot updates.

### Install and set console font
To make console text readable on laptop's display in installed system, i suggest to use font `terminus`.

Install font using `pacman -S terminus-font`

Then set it as default console font by putting in `/etc/vconsole.conf`:
<pre>FONT=ter-224n</pre>

You can also add `consolefont` hook in `/etc/mkinitpcio.conf`. [Se details](https://wiki.archlinux.org/index.php/Fonts#Persistent_configuration).

### NetworkManager
If you did not in steps before, it is time to enable network in our future system. There are plenty options on ArchLinux (including basic simple systemd-networkd if you dont want to instal additional packages). See [wiki page here](https://wiki.archlinux.org/index.php/Network_configuration) and pick what suits you.

Because I plan to use [Gnome](http://gnome.org/) as my main desktop. Only possible choice for me is [NetworkManager](https://wiki.archlinux.org/index.php/NetworkManager), because thats what you need if you want to configure network through Gnome GUI in future.

To have network connection available in installed system just do:
<pre>
pacman -S networkmanager dhclient
systemctl enable NetworkManager
</pre>

And you are ready to start installed system. Do it:
<pre>
umount -R /mnt
reboot
</pre>

## After installation steps
We have just installed base system and setup booting. Also we have network connection in booted system.

Now we can start building our desktop system. Here is few tips.

### Creating user, installing sudo, disabling root login
You probably do not want to run your entire desktop under `root` user. So we will create user in group admins who has right to run commands with sudo:

Create user
<pre>
groupadd admins
useradd -m -g admins -s /bin/bash USERNAME
passwd USERNAME
</pre>

Install sudo with `pacman -S sudo`.
Setup sudo to enable users in group admins to run any command (with password) by calling `visudo` and adding this line to sudoers:
<pre>
%admins      ALL=(ALL) ALL
</pre>

Now logout and login with new user. We can now disable root login by `sudo passwd -l root` and replacing root's encrypted password in `/etc/shadow` with `!`

### Installing graphics drivers, Gnome desktop environment with Wayland
When we have user created, we should install some [graphics environment](https://wiki.archlinux.org/index.php/Desktop_environment). Pick one that suits you. I will continue with [Gnome desktop](https://wiki.archlinux.org/index.php/GNOME). To do that we first need graphics driver.

#### Graphics driver
Since my Macbook has only [Intel graphics](https://wiki.archlinux.org/index.php/Intel_graphics), setup is pretty simple. Just run `pacman -S mesa`

If you are interested in video acceleration, [see wiki](https://wiki.archlinux.org/index.php/Hardware_video_acceleration).

When you have discrete graphics in your laptop, [this article](https://wiki.archlinux.org/index.php/Hybrid_graphics) seems interesting.

#### Gnome Desktop
With graphics driver installed we can install [Gnome desktop](https://wiki.archlinux.org/index.php/GNOME): `sudo pacman -S gnome gnome-initial-setup`

Since [Wayland](https://wiki.archlinux.org/index.php/Wayland) is now installed as Gnome dependecy, you do not need to install it manualy. If it does not installs with Gnome, just do `sudo pacman -S wayland xorg-server-xwayland`

Finally we can enable graphical [display manager](https://wiki.archlinux.org/index.php/Display_manager) by running `sudo systemctl enable gdm`.

On next reboot you should have fully functional graphical environment on your Macbook.

### Installing AUR helper (yaourt)
To make it comfortable installing packages from ArchLinux's famous [AUR](https://wiki.archlinux.org/index.php/Arch_User_Repository) you probably want some helper. I choosed [yaourt](https://archlinux.fr/yaourt-en).

So open `gnome-terminal` and install git and other needed tools with `sudo pacman -S git base-devel`.

Then follow instructions at [Yaourt website](https://archlinux.fr/yaourt-en) to install it. It is very comfortable to work with AUR after this is installed. You'll see.

### Install fan control daemon
To prevent overheating of Macbook you should install some [fan control daemon](https://wiki.archlinux.org/index.php/MacBookPro11,x#Fan_control) for Macbooks. I suggest to use `mbpfan-git`.

`yaourt -S mbpfan-git`

Edit `/etc/mbpfan.conf` to fit your machine (file is well commented)

Then run `sudo systemctl enable mbpfan` and `sudo systemctl start mbpfan`. That's all, your fans are working now.

### Get WiFi and Bluetooth working

#### WiFi
To this poing I was operating network using thunderbolt ethernet adapter. It is time to turn on WiFi adapter...

I suggest to use [DKMS](https://wiki.archlinux.org/index.php/Dynamic_Kernel_Module_Support) for kernel modules, since you might like to use multiple kernels. To do so, you must install kernel headers: `sudo pacman -S linux-headers`.

When you have headers for your kernel installed, you can use previously installed yaourt to get [Broadcom wireless module](https://wiki.archlinux.org/index.php/Broadcom_wireless): `yaourt -S broadcom-wl-dkms`

It is also good idea, just for sure, to add to `/etc/modprobe.d/blacklist_broadcom.conf` this:
<pre>blacklist b43
blacklist ssb</pre>

That's it. After reboot your wifi should work.

#### Bluetooth
Bluetooth on Macbook works out of the box. Just is not enabled by default.
To enable it run following commands:
<pre>
sudo pacman -S bluez bluez-utils pulseaudio-bluetooth
sudo systemctl enable bluetooth
sudo systemctl start bluetooth
</pre>

That's it. Bluetooth is on.

### Disabling S/PIDF (strange red light from audio jack)
You probably noticed there is srange red light in your headphones connector. This light is indicating your digital stereo output is activated.

To disable it. Install [ALSA](https://wiki.archlinux.org/index.php/Advanced_Linux_Sound_Architecture) utils `sudo pacman -S pulseaudio-alsa alsa-utils`.

Then run `sudo alsamixer` and disable `S/PIDF` and `S/PDIF Default PCM` output on your `HDA Intel PCH` soundcard. See [this forum](https://bbs.archlinux.org/viewtopic.php?pid=1587806#p1587806) for more informations.

### Enable tap to click in GDM
You probably noticed, that when you turned on "tap to click" feature in `gnome-settings`, it is working only after you log in with your user.

To enable tap to click in [GDM](https://wiki.archlinux.org/index.php/GDM) (before login) execute this in terminal:
<pre>sudo su - gdm -s /bin/bash
export $(dbus-launch)
GSETTINGS_BACKEND=dconf gsettings set org.gnome.desktop.peripherals.touchpad tap-to-click true</pre>

### Solving tochpad freeze after restart/wake

**This issue was probably fixed by [kernel 4.12.4](https://cdn.kernel.org/pub/linux/kernel/v4.x/ChangeLog-4.12.4) (commit a54408d0a004757789863d74e29c2297edae0b4d) and I do not experience it anymore now. So probably no need to do these workarounds**

I don't know if this issue is just wit mine Macbook. But if you experience random touchpad/keyboard freezes after reboot or after wakeup from suspend/hibernation this could help you. Details and solution is explained in [this forum thread](https://bbs.archlinux.org/viewtopic.php?id=221386).

I just added `i8042.reset` to kernel boot parameters. So mine `/boot/loader/entries/archlinux-vanilla.conf` looked like:
<pre>title	Arch Linux (vanilla)
linux	/vmlinuz-linux
initrd	/intel-ucode.img
initrd	/initramfs-linux.img
options	root=PARTLABEL=ROOT rw i8042.reset</pre>

This solved painful issue when touchpad/keyboard was frozen and I had no other choice than turning computer off using "long power button press". This is not happening more.

But I still experience issue when sometimes after boot touchpad is not responding for approx 20 seconds. After that timeout, everinting works OK and there ale no problems during usage of computer. If someone has idea how to fix this, I'd like to know. It seems like [this bug](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1668105)

For this it helps adding `usbcore.old_scheme_first=1` to kernel parameters. Just look [this forum](https://bbs.archlinux.org/viewtopic.php?pid=1691450#p1691450) for details.

### Get suspend/hibernation to work
For some reason suspend do not work by default on Macbook. For me solution was [this](https://wiki.archlinux.org/index.php/Mac#Suspend_and_Hibernate).

So for suspend to work just add this to `/etc/udev/rules.d/90-xhc_sleep.rules`:
<pre># disable wake from S3 on XHC1
SUBSYSTEM=="pci", KERNEL=="0000:00:14.0", ATTR{power/wakeup}="disabled"</pre>

You might also enable hibernation. [See ArchWiki](https://wiki.archlinux.org/index.php/Power_management/Suspend_and_hibernate#Hibernation) how to do that.

### Installing ZEN kernel
You might like to use custom kernel. I preffer [ZEN Kernel](https://github.com/zen-kernel/zen-kernel) over standard vanilla. You can see [ArchWiki](https://wiki.archlinux.org/index.php/Kernels) for details. I think this kernel is just working better than vanilla. You might experience some other bugs/problems which I do not on ZEN Kernel.

If you want to use ZEN, just do `sudo pacman -S linux-zen linux-zen-headers`.

Then create config file for systemd-boot in `/boot/loader/entries/archlinux.conf`:
<pre>title	Arch Linux (ZEN)
linux	/vmlinuz-linux-zen
initrd	/intel-ucode.img
initrd	/initramfs-linux-zen.img
options	root=PARTLABEL=ROOT rw i8042.reset usbcore.old_scheme_first=1</pre>

If you want to boot this kernel by default, edit corresponding line in `/boot/loader/loader.conf`.

There is also [linux-macbook](https://aur.archlinux.org/packages/linux-macbook/) kernel you maybe like to try.

### Power Management
For better battery life, you can use [TLP](https://wiki.archlinux.org/index.php/TLP).

Just install and enable it.
<pre>sudo pacman -S tlp ethtool smartmontools x86_energy_perf_policy tlp-rdw
sudo systemctl enable tlp
sudo systemctl enable tlp-sleep
sudo systemctl enable NetworkManager-dispatcher.service</pre>

Then disable systemd-rfkill
<pre>sudo systemctl mask systemd-rfkill.service
sudo systemctl mask systemd-rfkill.socket</pre>

You can also change TLP configuration using file `/etc/default/tlp`.

### Getting Windows to work
As I've mentioned, I had Windows 10 installed using Bootcamp (just for few games, to be able to have some fun). Infortunately when I removed macOS, I could not boot to Windows anymore.

First step to fix this is convert your disk partition table from Hybrid GPT/MBR (which macOS is using) to only GPT. See details [here](https://superuser.com/questions/508026/windows-detects-gpt-disk-as-mbr-in-efi-boot/508454).

Do following steps:
- run `sudo gdisk /dev/YOURDISK` (for example `sudo gdisk /dev/sda`)
- in gdisk type `p` to check you are on right disk (You do not want to destroy other disk partition table)
- then type `x` for advanced menu
- type `n` to clear MBR partition table
- finally type `w` and confirm to replace old data on disk

When this is done, Windows is probably not booting yet. Now you must boot from Windows installation media (If you do not have one, you can easily create it using `dd` since [Microsoft is offering download of intaller ISO](https://www.microsoft.com/cs-cz/software-download/windows10ISO) in UEFI mode. Then use `Repair Windows` option from installer menu and select `Repair booting`.

Now you should have working Windows. Just one more thing. Now your Macbook is booting to Windows by default. But we want to start `systemd-boot` by default and select operating system here.

To fix this, just boot into your Arch Linux (by holding alt/option key when powering on the Macbook, or boot from Arch installation media). Remember that if you are booting from installation media you must again mount at least `/` and `/boot` partitions and then chroot into mounted environment. When you are in Arch Linux environment, just repeat `sudo bootctl install` procedure.

It's done. Now you can boot both Arch Linux or Windows on your Macbook.
