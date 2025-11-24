---
date: '2025-09-30T21:05:36-03:00'
draft: false
title: 'Anatomy of a Rockchip Android ROM'
description: 'This article details the process of unpacking and analyzing a firmware image for devices based on Rockchip SoCs (System on a Chip), specifically the RK322x. The process covers everything from the initial factory image to the extraction of individual Android system partitions.'
tags: ["Android", "Firmware", "Rockchip", "Analysis"]
cover:
  image: "/images/anatomy-of-a-rockchip-android-rom.png"
  alt: "Anatomy of a Rockchip Android ROM"
  relative: true
---

## 1\. Firmware Container Analysis (RKFW Format)

The starting point is a firmware image file, usually with an `.img` extension. The first step is to identify its format. Using `hexdump`, we can inspect the first few bytes of the file.

```bash
$ hexdump -C rom_rk322x.img | head -n2
00000000  52 4b 46 57 66 00 00 00  00 08 00 00 06 01 e2 07  |RKFWf...........|
00000010  0c 11 16 21 32 41 32 32  33 66 00 00 00 4e a9 02  |...!2A223f...N..|
```

The header analysis reveals the `RKFW` signature. This is a container format used by Rockchip, primarily in factory update tools like the *Rockchip Factory Tool*.

## 2\. Extracting the Main Image (RKFW → RKAF)

To extract the contents of the `RKFW` container, we will use the `img_unpack` tool from the **rk2918\_tools** suite.

  - **Tool Repository:** [https://github.com/dayongxie/rk2918\_tools](https://github.com/dayongxie/rk2918_tools)

After compiling the tools, `img_unpack` can be used as follows:

```
usage: ./img_unpack <source> <destination>
```

Executing the extraction:

```bash
$ ./rk2918_tools/img_unpack rom_rk322x.img rom.img
rom version: 8.0.0
build time: 2018-12-17 22:33:50
chip: 33323241
checking md5sum....OK
```

The `RKFW` container stores a main image in the `RKAF` (Rockchip Android Firmware) format. Let's verify the header of the extracted file:

```bash
$ hexdump -C rom.img | head -n 2
00000000  52 4b 41 46 00 f0 c8 4c  52 4b 33 32 32 78 00 00  |RKAF...LRK322x..|
00000010  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
```

As expected, the `RKAF` signature confirms a successful extraction.

## 3\. Unpacking the RKAF Archive into Individual Partitions

The `RKAF` file is a package that groups all the firmware partitions (`boot.img`, `system.img`, etc.). To unpack it, we use the `afptool`, also part of `rk2918_tools`.

```
USAGE:
    afptool <-pack|-unpack> <Src> <Dest>
Example:
    afptool -pack xxx update.img    Pack files
    afptool -unpack update.img xxx  Unpack files
```

Executing the unpack process:

```bash
$ ./rk2918_tools/afptool -unpack rom.img partitions
Check file...OK
------- UNPACK -------
package-file    0x00000800  0x000002D4
Image/MiniLoaderAll.bin 0x00001000  0x0002A94E
Image/parameter.txt 0x0002C000  0x00000400
Image/trust.img 0x0002C800  0x00400000
Image/uboot.img 0x0042C800  0x00400000
Image/misc.img  0x0082C800  0x0000C000
Image/baseparameter.img 0x00838800  0x00100000
Image/resource.img  0x00938800  0x00BE7A00
Image/kernel.img    0x01520800  0x007AED94
Image/boot.img  0x01CCF800  0x00189138
Image/recovery.img  0x01E59000  0x006E8F1C
Image/system.img    0x02542000  0x44214B84
Image/vendor.img    0x46757000  0x0650E51C
Image/oem.img   0x4CC65800  0x0002907C
RESERVED    0x00000000  0x00000000
UnPack OK!
```

The command created the `partitions` directory with the following structure:

```
partitions/
├── Image
│   ├── baseparameter.img
│   ├── boot.img
│   ├── kernel.img
│   ├── MiniLoaderAll.bin
│   ├── misc.img
│   ├── oem.img
│   ├── parameter.txt
│   ├── recovery.img
│   ├── resource.img
│   ├── system.img
│   ├── trust.img
│   ├── uboot.img
│   └── vendor.img
├── package-file
└── RESERVED
```

## 4\. Analysis of Metadata Files

Before analyzing the partitions, let's examine the generated metadata files.

### `package-file`

This file acts as a manifest, mapping partition names to their respective image files.

```
# NAME		Relative path
#
#HWDEF		HWDEF
package-file	package-file
bootloader	Image/MiniLoaderAll.bin
parameter	Image/parameter.txt
trust       Image/trust.img
uboot       Image/uboot.img
misc		Image/misc.img
baseparameter		Image/baseparameter.img
resource	Image/resource.img
kernel		Image/kernel.img
boot        Image/boot.img
recovery	Image/recovery.img
system		Image/system.img
vendor		Image/vendor.img
oem		Image/oem.img
# Ҫд��backup�������ļ�����������update.img��
# SELF �ǹؼ��֣���ʾ�����ļ���update.img������
# �����������ļ�ʱ��������SELF�ļ������ݣ�����ͷ����Ϣ���м�¼
# �ڽ�������ļ�ʱ�������SELF�ļ������ݡ�
backup		RESERVED
#update-script	update-script
#recover-script	recover-script
```

Commented lines with garbled characters are remnants of text in other languages and can be ignored.

### `parameter.txt`

This text file is crucial, as it defines boot parameters and the partition layout on the flash memory.

```
FIRMWARE_VER:8.0
MACHINE_MODEL:RK322x
MACHINE_ID:007
MANUFACTURER:Rockchip
MAGIC: 0x5041524B
ATAG: 0x00200800
MACHINE: 322x
CHECK_MASK: 0x80
PWR_HLD: 0,0,A,0,1
CMDLINE: console=ttyFIQ0 androidboot.baseband=N/A androidboot.selinux=permissive androidboot.wificountrycode=US androidboot.veritymode=/dev/block/platform/30020000.dwmmc/by-name/metadata androidboot.hardware=rk30board androidboot.console=ttyFIQ0 firmware_class.path=/vendor/etc/firmware init=/init initrd=0x62000000,0x00800000 mtdparts=rk29xxnand:0x00002000@0x00002000(uboot),0x00002000@0x00004000(trust),0x00002000@0x00006000(misc),0x00000800@0x00008000(baseparameter),0x00008000@0x00008800(resource),0x00010000@0x00010800(kernel),0x00010000@0x00020800(boot),0x00020000@0x00030800(recovery),0x00008000@0x00050800(backup),0x00080000@0x00058800(cache),0x00300000@0x000d8800(system),0x00008000@0x003d8800(metadata),0x00080000@0x003e0800(vendor),0x00042000@0x00460800(oem),0x00000400@0x004a2800(frp),0x00001000@0x004a2c00(security),-@0x004a3c00(userdata)
```

Key Components:

  - **Metadata:** Firmware version, device model, and manufacturer.
  - **`CMDLINE`:** The command line passed to the Linux kernel during boot. It contains essential boot arguments.
  - **`mtdparts`:** Defines the flash memory (NAND/eMMC) layout, specifying the size, offset, and name of each partition.

## 5\. Individual Partition Analysis

Now, let's analyze each extracted image file.

### `MiniLoaderAll.bin` (Bootloader)

This is the first-stage bootloader. Its `BOOTf` signature confirms its identity.

```bash
$ hexdump -C Image/MiniLoaderAll.bin | head -n 1
00000000  42 4f 4f 54 66 00 38 02  00 00 00 00 03 01 e2 07  |BOOTf.8.........|
```

Its function is to initialize basic hardware and load the next boot stage, `uboot.img`.

### `uboot.img` (U-Boot)

The second-stage bootloader, based on the popular Das U-Boot, with Rockchip customizations. Its signature is ` LOADER   `.

```bash
$ hexdump -C Image/uboot.img | head -n 1
00000000  4c 4f 41 44 45 52 20 20  00 00 00 00 00 00 00 00  |LOADER  ........|
```

It contains uncompressed ELF binaries (the executable format for Linux/Unix) and RSA signatures for integrity verification.

### `trust.img` (TrustZone)

This file contains the firmware for the TEE (Trusted Execution Environment), known as TrustZone. The `TOS` (Trusted Operating System) signature identifies it.

```bash
$ hexdump -C Image/trust.img | head -n 1
00000000  54 4f 53 20 20 20 20 20  00 00 00 00 00 00 00 00  |TOS     ........|
```

It is responsible for security-sensitive operations, such as cryptographic key management and secure boot.

### `kernel.img` (Linux Kernel)

This contains the operating system's kernel. Its structure is:

  - **Offset `0x0` – `0xF`:** `KRNL` header (16 bytes).
  - **Starting from a specific offset:** The LZO-compressed kernel.

To extract and decompress the kernel:

```bash
# 1. Extract the LZO stream (adjust 'skip' and 'count' based on the image)
$ dd if=Image/kernel.img of=kernel.lzo bs=1 skip=$((0x3C1C)) count=8040763

# 2. Decompress with lzop
$ lzop -d kernel.lzo
```

The result is a binary file containing the raw Linux kernel for the ARM architecture.

### `boot.img` and `recovery.img` (Ramdisk)

In the Rockchip ecosystem, `boot.img` and `recovery.img` usually do not contain the kernel but rather the **ramdisk** (initramfs) for normal and recovery modes, respectively. Both have a similar structure:

  - **Offset `0x0` – `0x7`:** `KRNL` header (8 bytes).
  - **Starting from `0x8`:** A `gzip` file containing the ramdisk in CPIO format.
  - **End of file:** A few bytes of padding.

The `1f 8b 08` signature at offset `0x8` confirms the `gzip` format.

```bash
$ hexdump -C Image/boot.img | head -n 1
00000000  4b 52 4e 4c 2c 91 18 00  1f 8b 08 00 00 00 00 00  |KRNL,...........|
```

Extraction process (example with `boot.img`):

```bash
# 1. Extract the gzip file (adjust 'count' according to its size)
$ dd if=Image/boot.img of=ramdisk.gz bs=1 skip=8 count=1610027

# 2. Decompress the ramdisk
$ gunzip ramdisk.gz

# 3. Extract the contents of the CPIO archive
$ mkdir ramdisk_contents
$ cd ramdisk_contents
$ cpio -idmv < ../ramdisk
```

The result is the initial root file structure for Android.

### `system.img`, `vendor.img`, `oem.img` (System Partitions)

These images contain the main Android partitions. They are in the **Android sparse image** format, an optimization to save space by not storing empty blocks.

The magic number `0xED26FF3A` (read as `3A FF 26 ED` in little-endian) identifies this format.

```bash
$ hexdump -C Image/system.img | head -n 1
00000000  3a ff 26 ed 01 00 00 00  1c 00 0c 00 00 10 00 00  |:.&.............|
```

To access them, first convert them to a raw image and then mount them:

```bash
# 1. Convert the sparse image to a raw image
$ simg2img Image/system.img system_raw.img

# 2. Create a mount point and mount the image
$ mkdir sys_mount
$ sudo mount -o loop system_raw.img sys_mount

# 3. List the contents
$ ls sys_mount/
app           etc        lib           media       usr
bin           fake-libs  lib64         preinstall  vendor
build.prop    fonts      lost+found    priv-app    xbin
...
```

The same process applies to `vendor.img` and `oem.img`.

### `misc.img`

This is a small partition used to pass commands between the system and recovery mode. It does not have a formatted filesystem; data is written to and read from specific offsets.

```bash
$ hexdump -C Image/misc.img | tail
00004000  62 6f 6f 74 2d 72 65 63  6f 76 65 72 79 00 00 00  |boot-recovery...|
00004010  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00004040  72 65 63 6f 76 65 72 79  0a 2d 2d 77 69 70 65 5f  |recovery.--wipe_|
00004050  61 6c 6c 00 00 00 00 00  00 00 00 00 00 00 00 00  |all.............|
```

In the example, we see flags like `boot-recovery` and `recovery --wipe_all`.

### Other Files

  - **`baseparameter.img`:** A binary file with hardware parameters, such as video output settings (HDMI).
  - **`resource.img`:** A resource package (`RSCE`) that can contain the Device Tree Blob (DTB) and other hardware configuration data.

## 6\. Putting It All Together: The Filesystem Table (`fstab`)

To understand how the system mounts these partitions during boot, we analyze the `fstab` (filesystem table) file located inside the ramdisk extracted from `boot.img`.

**File:** `ramdisk_contents/fstab.rk30board`

```
# Android fstab file
#<src>                                          <mnt_point>         <type>    <mnt_flags and options>                       <fs_mgr_flags>
# The filesystem that contains the filesystem checker binary (typically /system) cannot
# specify MF_CHECK, and must come before any filesystems that do specify MF_CHECK

/dev/block/by-name/oem            /oem                ext4      ro,noatime,nodiratime,noauto_da_alloc                                  wait,check
/dev/block/by-name/cache          /cache              ext4      noatime,nodiratime,nosuid,nodev,noauto_da_alloc,discard                wait,check
/dev/block/by-name/metadata       /metadata           ext4      noatime,nodiratime,nosuid,nodev,noauto_da_alloc,discard                wait
/dev/block/by-name/misc           /misc               emmc      defaults                                                              defaults

/devices/platform/*usb*           auto                vfat      defaults                                                              voldmanaged=usb:auto

/dev/block/zram0                  none                swap      defaults                                                              wait,zramsize=50%,notrim
# For sdmmc
/devices/platform/30000000.dwmmc/mmc_host*  auto       auto      defaults                                                              voldmanaged=sdcard1:auto,encryptable=userdata
# Full disk encryption has less effect on rk322x, so default to enable this.
#/dev/block/by-name/userdata       /data               f2fs      noatime,nodiratime,nosuid,nodev,discard,inline_xattr                   wait,check,notrim,encryptable=/metadata/key_file,quota
/dev/block/by-name/userdata       /data               f2fs      noatime,nodiratime,nosuid,nodev,discard,inline_xattr                   wait,check,notrim
```

The following table summarizes the mount order and types:

| Source Partition (`by-name`) | Mount Point | Filesystem (FS) |
| :--------------------------- | :---------- | :-------------- |
| `oem`                        | `/oem`      | `ext4`          |
| `cache`                      | `/cache`    | `ext4`          |
| `metadata`                   | `/metadata` | `ext4`          |
| `misc`                       | `/misc`     | `emmc` (direct access) |
| `userdata`                   | `/data`     | `f2fs`          |

