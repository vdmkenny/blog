---
title: "Updating older device BIOS without burning a CD"
subtitle: "Using ventoy to boot CD-only ISO files from USB"
date: "2024-06-05"
toc: true # table of contents (only h2 and h3 added to toc)
bold: true # display post title in bold in posts list
math: false # load katex
tags:
  - Linux
  - Thinkpad
next: true # show link to next post in footer
draft: false
---

## Background

This is a quick post to document a solution that I only stumbled upon by accident. I hope I can save you a couple of hours of dissecting an old CD iso with this simple solution.

I have an old Lenovo [Thinkpad X230](https://psref.lenovo.com/syspool/Sys/PDF/withdrawnbook/ThinkPad_X230.pdf) that had fallen out of daily use a while ago.
However, even though this is now a 12 year old model, being a sturdy 12.5" device with some upgradeability, it seems like a good candidate to travel with me to Japan this year.

It has modest specs, Intel i5-3320M, only 8GB of DDR3 RAM, a good looking 720p screen, and all the nice Thinkpad features of the past.

Chinesium 94Wh batteries are still readily availably, so it seems optimal for a traveling machine.

I had a look at the bios version using `dmidecode`, and it shows it's on version 2.02 released in 2012.

```bash
➜  ~ sudo dmidecode | grep 'BIOS Revision'
	BIOS Revision: 2.02
```

A quick look at the [Lenovo website](https://pcsupport.lenovo.com/us/en/products/laptops-and-netbooks/thinkpad-x-series-laptops/thinkpad-x230/downloads/ds029187) shows that the most recent version is 2.77, released in 2019! Quite a jump!

This device predates [fwupd](https://fwupd.org/) by quite some years. This tool has largely made the problem here a thing of the past, for compatible devices. (Which modern Thinkpads are!)

The files offered by Lenovo are either:
- Windows .exe file
- Bootable CD ISO

Since I have no interest in running Windows, this is option is out.
So...

## CD's, what are those?

As you may remember, USB booting is a _relatively_ new thing. ISO files that can be written to USB drives are called Hybrid ISOs, since they have 2 bootloaders. A normal CD ISO can normally only boot from a burned CD.

Let's have a look what we are dealing with:
```bash
➜  file g2uj33us.iso
g2uj33us.iso: ISO 9660 CD-ROM filesystem data 'G2ETB7WWUS' (bootable)
```

Nope, this isn't a hybrid iso. This file was made to boot off a CD.
Not ideal, since it's 2024, and this laptop doesn't even have a CD drive!

I'll save you the trouble: writing this iso to a USB drive with `dd` does not result in a bootable drive. I tried it just to be sure.
Maybe other machines with different BIOSes would, but not this one.

## What about turning it in to a hybrid iso?

The program `isohybrid` (Found in the pacman package `core/syslinux`) claims to do just that. But if it was possible in this case, I had no idea how.
When I tried running this tool against this iso file, it pretty much had a stroke:
```bash
➜  isohybrid g2uj33us.iso
isohybrid: g2uj33us.iso: unexpected boot catalogue parameters
```

This error led me to some 10 year old forum posts that ultimately were a dead end.

## WWVD? (What would ventoy do?)

While I've personally never used it, [Ventoy](https://www.ventoy.net/en/index.html) is a name I heard a few times some years ago. 

It claimed to be a tool which you can use to put multiple ISO files on a drive. This way you can carry around your 17 linux distros just in case you need to switch from OpenSUSE TumbleWeed to Pop!OS in a pinch. Or something.

I've never really been a distrohopper, so I had never tried it before. 
Lo and behold, though, it claims it can boot any kind of ISO.

On their website, they have a page where they list all [tested ISOs](https://www.ventoy.net/en/isolist.html) and they claim to have tested "Lenovo BIOS Update"!

I installed [aur/ventoy-bin](https://aur.archlinux.org/packages/ventoy-bin) and flashed it to a USB drive.

```bios
# As always, make sure you confirm the correct device name
# before doing a destructive action like this.
➜  ~ sudo ventoy -i /dev/sdb
```

After this, I was left with a usb device with 2 partitions:
```bash
➜  ~ lsblk /dev/sdb
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sdb      8:16   1 14.5G  0 disk
├─sdb1   8:17   1 14.4G  0 part
└─sdb2   8:18   1   32M  0 part
```

I mounted the largest partition, copied the iso file, and unmounted it safely:
```bash
➜  ~ sudo mount /dev/sdb1 /mnt
➜  ~ sudo cp g2uj33us.iso /mnt
➜  ~ sync
➜  ~ sudo umount /mnt
```

After this, I rebooted the machine and booted from the USB drive.
If you're playing along from home, always make sure you go into a firmware upgrade like this with a full battery and the AC adapter connected securely.

![At this point I was quite sure something would go wrong and the laptop would be bricked...](/blog/thinkpadbios/biosupgrade.jpg)

It worked! I have to admit I was kind of surprised.

After a reboot, I could gloat about a command now outputting a slightly bigger number. Yay!

```bash
➜  ~ sudo dmidecode | grep 'BIOS Revision'
	BIOS Revision: 2.77
```

## Conclusion

With this, I've finished yet another procrastination task instead of doing actual preparation for my Japan trip later this year.

This has proven a good way to update a legacy system without having to resort to an old Windows version. I hope this helps!
