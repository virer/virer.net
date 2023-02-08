# iPXE and UEFI ISO

<br>

To create a UEFI compatible ISO file, 
we need to create a small block image file formated as vfat containing the EFI "executable" image and 
add it to the ISO as an alternative way to boot.
Below I will describe how to do that.

## Required packages :
(as always the package name may change on your distro)

```bash
# yum install mtools grub2-efi grub2-efi-x86-modules dosfstools
```

Create the efi block image file:
```bash
#!/bin/sh

BOOT_IMG_DATA=$(mktemp -d)
BOOT_IMG=/tmp/efi.img

mkdir -p $(dirname $BOOT_IMG)

truncate -s 4M $BOOT_IMG
mkfs.vfat $BOOT_IMG
mkdir -p $BOOT_IMG_DATA/efi/boot

grub2-mkimage \
    -C xz \
    -O x86_64-efi \
    -p /boot/grub \
    -o $BOOT_IMG_DATA/efi/boot/bootx64.efi \
    boot linux search normal configfile \
    chain efifwsetup search_label search_fs_uuid search_fs_file \
    part_gpt btrfs fat iso9660 loopback \
    test keystatus gfxmenu regexp probe \
    efi_gop efi_uga all_video gfxterm font \
    echo read ls cat png jpeg halt reboot

mcopy -i $BOOT_IMG -s $BOOT_IMG_DATA/efi ::

# EOF
```bash

Then go inside the ISO tree (I mean at the root tree of the future ISO)

```bash
cd "~/ISO_content"
```

And create the ISO with xorriso tool:

```bash
#!/bin/bash

BOOT_IMG=/tmp/efi.img
OUTPUT=/tmp/uefi_ipxe_filename.iso

# FYI the "boot.cat" file is generated on the fly
xorriso -as mkisofs \
    -r -V "IPXE_UEFI" \
    -o $OUTPUT \
    -J -J -joliet-long -cache-inodes \
    -b isolinux/isolinux.bin \
    -c isolinux/boot.cat \
    -boot-load-size 4 -boot-info-table -no-emul-boot \
    -eltorito-alt-boot \
    -e --interval:appended_partition_2:all:: \
    -append_partition 2 0xef $BOOT_IMG \
    -no-emul-boot -isohybrid-gpt-basdat \
    .

# EOF
```

