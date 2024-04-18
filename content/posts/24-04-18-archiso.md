---
title: "Creating a custom Archlinux ISO"
subtitle: "Using Archiso to create a custom Archlinux ISO with custom packages and configurations"
date: "2024-04-18"
toc: true # table of contents (only h2 and h3 added to toc)
bold: true # display post title in bold in posts list
math: false # load katex
tags:
  - Linux
  - Github
next: true # show link to next post in footer
draft: false
---

## Background

At my job, I was faced with the task of creating a way to provision small-form-factor PCs with a custom Linux installation.
We had a Ubuntu VM image that we had been using with customers that had virtualisation infrastructure, and now we wanted an easy way to install it on bare metal in the field. We couldn't rely on PXE booting, as the network infrastructure was not under our control.
The technicians in the field were not very tech-savvy, so we needed a solution that was as simple as possible.
Every PC needed a slightly different configuration, so we needed something interactive.

After some research, I found tools like [goldboot](https://github.com/fossable/goldboot) that looked interesting, but at the time this was far from being production-ready.
Being an Archlinux user myself, I remembered that Archlinux has a tool called `archiso` that is used to create the official Archlinux ISOs. It can be used to spin a custom ISO with custom packages and configurations.
It's also possible to add a startup script that automatically runs when the ISO boots, so I could make it interactive. The only purpose of this script would be to ask the user for some information and then run a script that would configure the system according to the user's input. It would [dd](https://linux.die.net/man/1/dd) the included Ubuntu VM image to the disk, resize the partition, and install the bootloader. Then some configuration files would be overwritten with the user's custom configuration.

## Archiso

### Configuration

Using [Archiso](https://wiki.archlinux.org/title/archiso) is quite straightforward if you're using ArchLinux. We were not, so I had to come up with a way to build it on Github Actions. We ended up using a Docker container to build the ISO.

It's best to start from the baseline example. As of writing, the link in the documentation page is dead. You can find it [here](https://github.com/archlinux/archiso/tree/master/configs/baseline).

Copy this folder to another location and start customizing it.

Files you might want to customize:
- `packages.x86_64`: Add the packages you want to include in the ISO.
- `pacman.conf`: Add your custom repository if you have one, or change the mirror list.
- `profiledef.sh`: This is where you define the name and compression type of the ISO.

To include extra files in the ISO, you can add them to the `airootfs` folder. This is where the root filesystem of the ISO is built.

This is how our `profiledef.sh` looked like:

```bash
#!/usr/bin/env bash
# shellcheck disable=SC2034

iso_name="bootstrap"
iso_label="ARCH_$(date --date="@${SOURCE_DATE_EPOCH:-$(date +%s)}" +%Y%m)"
iso_publisher="REDACTED"
iso_application="Custom PC Bootstrap"
iso_version="$(date --date="@${SOURCE_DATE_EPOCH:-$(date +%s)}" +%Y.%m.%d)"
install_dir="arch"
buildmodes=('iso')
bootmodes=('bios.syslinux.mbr' 'bios.syslinux.eltorito'
           'uefi-ia32.grub.esp' 'uefi-x64.grub.esp'
           'uefi-ia32.grub.eltorito' 'uefi-x64.grub.eltorito')
arch="x86_64"
pacman_conf="pacman.conf"
airootfs_image_type="squashfs"
airootfs_image_tool_options=('-comp' 'xz' '-Xbcj' 'x86' '-Xdict-size' '1M' '-b' '1M')
file_permissions=(
  ["/etc/shadow"]="0:0:400"
  ["/root"]="0:0:750"
  ["/root/.automated_script.sh"]="0:0:755"
  ["/root/.gnupg"]="0:0:700"
  ["/usr/local/bin/choose-mirror"]="0:0:755"
  ["/usr/local/bin/Installation_guide"]="0:0:755"
  ["/usr/local/bin/livecd-sound"]="0:0:755"
)
```

One notable change is the compression type of the `airootfs_image_tool_options`. We changed it to `xz` as a compromise between compression ratio and speed.
You can read about all the options in the [mksquashfs manpage](https://manpages.debian.org/jessie/squashfs-tools/mksquashfs.1.en.html#Compressors_available_and_compressor_specific_options).

Archiso produces an iso that can boot in both BIOS and UEFI systems. You can customize the boot menu by editing the `syslinux` and `grub` configuration files.

This is how I customized the `grub` configuration file to hide the startup messages and immediately go into the installation script:

```bash
menuentry "Custom PC Installer (%ARCH%, ${archiso_platform})" --class arch --class gnu-linux --class gnu --class os --id 'archlinux' {
    set gfxpayload=keep
    linux /%INSTALL_DIR%/boot/%ARCH%/vmlinuz-linux archisobasedir=%INSTALL_DIR% archisodevice=UUID=${ARCHISO_UUID} script=/opt/bootstrap.sh quiet
    initrd /%INSTALL_DIR%/boot/intel-ucode.img /%INSTALL_DIR%/boot/amd-ucode.img /%INSTALL_DIR%/boot/%ARCH%/initramfs-linux.img
}
```

Syslinux was a bit more of a mess to edit, it has many files that link to each other in a strange way if you've never seen it before. You can modify the colors and background image easily, however.
This is how I modified the `archiso_sys-linux.cfg` file to achieve the same effect as with the grub modifications.

```bash
LABEL arch64
TEXT HELP
Boot the Custom PC Installer on BIOS.
ENDTEXT
MENU LABEL Custom PC Installer (x86_64, BIOS)
LINUX /%INSTALL_DIR%/boot/x86_64/vmlinuz-linux
INITRD /%INSTALL_DIR%/boot/intel-ucode.img,/%INSTALL_DIR%/boot/amd-ucode.img,/%INSTALL_DIR%/boot/x86_64/initramfs-linux.img
APPEND archisobasedir=%INSTALL_DIR% archisodevice=UUID=%ARCHISO_UUID% script=/opt/bootstrap.sh quiet
```

The `bootstrap.sh` script is the script that will be run when the ISO boots.
It's outside the scope of this post to show the contents of this script (and it's too tailored to my employer's need), but it's a simple bash script that uses `dialog` to ask the user for some information and then runs the installation script.
I might write a post about it in the future.

### Building the ISO

As we needed to build this ISO using our CI/CD pipeline, we used a Docker container to build it. The public Github Actions runners provided run Ubuntu.

This workflow file is a bit of a hack, but it works. It uses a Docker container to build the ISO and then copies the resulting ISO as an artifact. This last part is pretty slow, it would be better to upload the ISO to a cloud storage service
This workflow example shows how we included the VM images in the ISO, which you of course can completely ignore.

At the time of writing, the touch command is necessary to create a file that is missing in the Archiso build process. This is a [bug](https://gitlab.archlinux.org/archlinux/archiso/-/issues/225) in Archiso that has been probably been fixed in the latest version by now.

_.github/workflows/ci.yml_
```yaml
name: CI

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  build:
    runs-on: self-hosted
    steps:
    - uses: actions/checkout@v4
    - name: archiso
      run: |
        docker run --rm -v ${{ github.workspace }}:/workspace --privileged archlinux:base-devel \
          bash -c "\
            pacman -Syyuu --noconfirm && \
            pacman -Sy archiso qemu-img wget --noconfirm && \
            mkdir -p /vmdk && cd /vmdk && \
            wget -qO disk-1.vmdk '${{ secrets.URL_1 }}' && \
            qemu-img convert -p -f vmdk -O raw /vmdk/disk-1.vmdk /workspace/bootstrap/airootfs/images/disk-image-1.raw && \
            pacman -Rs wget qemu-img --noconfirm && \
            cd / && rm -rf /vmdk && \
            mkdir -p /tmp/x86_64/airootfs/usr/share/licenses/common/GPL2/ && touch /tmp/x86_64/airootfs/usr/share/licenses/common/GPL2/license.txt && \
            find /workspace -type f -name '.gitkeep' -delete 2> /dev/null && \
            mkarchiso -v -w /tmp -o /workspace /workspace/bootstrap && \
            find /workspace -type f ! -name '*.iso' -exec rm {} \; "
    - name: 'Archive ISO'
      uses: actions/upload-artifact@v3
      with:
        name: iso
        path: ${{ github.workspace }}/*.iso
```

## Conclusion
Archiso is a nice building block to create custom Linux bootable media. I haven't quite seen it used in production in the way I used it, but it worked well for us.

It's probably handy to create both unattended installations and custom rescue media in many situations. I hope this post has been helpful to you.
