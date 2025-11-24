---
date: '2025-09-23T09:26:39-03:00'
draft: false
title: 'IMAGEWTY Firmware Format: Structure and Analysis'
description: 'Technical exploration of the IMAGEWTY firmware format, its structure, and underlying components.'
tags: ["Firmware", "Allwinner", "Analysis"]
cover:
  image: "/images/imagewty-firmware-format-structure-and-analysis.png"
  alt: "IMAGEWTY Illustration"
  relative: true
---

The **IMAGEWTY** format is used by the **LiveSuit**, **PhoenixSuit**, **PhoenixUSB**, and **PhoenixCard** tools, all of which are **closed-source**, as are their protocols and standards.

---

## Extraction and Repacking Tools

* **[imgRePacker](https://xdaforums.com/t/tool-imgrepacker-livesuits-phoenixsuits-firmware-images-unpacker-packer.1753473/) (Windows)**
  Worked only on Windows, despite having a Linux version. Failed to process images with header `0x403`.

* **[imagewty-tool](https://github.com/uictorius/imagewty-tool) (Linux)**
  Functional and still **under development**. Successfully tested with images having headers `0x300` and `0x403`.

* **[AWUtils](https://github.com/Ithamar/awutils) (Linux)**
  **Outdated**, and extraction did **not work** as expected (bugs).

---

# Basic Structure

The basic layout of an **IMAGEWTY** format image can be represented as follows:

> This structure allows creating a valid image even without the random (**garbage**) data present in traditional images. In a hexadecimal view, additional **data of unknown** purpose can be observed.

```
+------------------------------+
| Global Header (96 bytes)     |
+------------------------------+
| Garbage / Unused (928 bytes) |
+------------------------------+
| File Header #1 (1024 bytes)  |
+------------------------------+
| File Header #2 (1024 bytes)  |
+------------------------------+
| File Header #3 (1024 bytes)  |
+------------------------------+
| ...                          |
+------------------------------+
| File Header #48 (1024 bytes) |
+------------------------------+
|                              |
|   File Data #1               |
|   (size varies)              |
|                              |
+------------------------------+
|                              |
|   File Data #2               |
|   (size varies)              |
|                              |
+------------------------------+
| ...                          |
+------------------------------+
|                              |
|   File Data #48              |
|   (size varies)              |
|                              |
+------------------------------+
```

The image consists of a **global header**, followed by **individual file headers**, and finally the **files themselves**.

> In the **example**, the illustration contains **48 files**, each with its respective **header**.

---

## Global Header

Although the header is **96 bytes**, only **68 bytes** are actually used. The structure is as follows:

```
+---------+-----------+----------------------------------------+
| Offset  | Size       | Field / Description                   |
+---------+-----------+----------------------------------------+
| 0x00    | 8 bytes    | Magic / Signature ("IMAGEWTY")        |
| 0x08    | 4 bytes    | Header Version                        |
| 0x0C    | 4 bytes    | Header Size                           |
| 0x10    | 4 bytes    | Base RAM                              |
| 0x14    | 4 bytes    | Format Version                        |
| 0x18    | 4 bytes    | Total Image Size (compressed)         |
| 0x1C    | 4 bytes    | Header Size Including Alignment       |
| 0x20    | 4 bytes    | File Header Length                    |
| 0x24    | 4 bytes    | USB Product ID                        |
| 0x28    | 4 bytes    | USB Vendor ID                         |
| 0x2C    | 4 bytes    | Hardware ID                           |
| 0x30    | 4 bytes    | Firmware ID                           |
| 0x34    | 4 bytes    | Unknown Field #1                      |
| 0x38    | 4 bytes    | Unknown Field #2                      |
| 0x3C    | 4 bytes    | Number of Files in Archive            |
| 0x40    | 4 bytes    | Unknown Field #3                      |
| 0x44…   | padding    | Zeros to align to Header Size (96 B)  |
+---------+-----------+----------------------------------------+
```

The header starts with the 8-byte signature **"IMAGEWTY"**.
The remaining fields are stored in **4-byte blocks**, totaling **68 bytes** of actual data.
The **28 extra bytes** and the **928-byte garbage area** can be filled with zeros.

![96 bytes](/images/hexdump-hy300.png)

**Offset:** from `0x000` to `0x3FF` — includes **96 bytes of header** and **928 bytes of garbage**, totaling **1024 bytes**.
After this, the file headers begin at offset `0x400`.

---

### Expected Values

Values present in the global header of the example image:

```
Magic / Signature    = "IMAGEWTY"
Header Version       = 0x300      (v3)
Header Size          = 0x60       (96 Bytes)
Base RAM             = 0x4D00000
Format Version       = 0x100234
Total Size           = 0xA34B3800 (2,612.70 MB)
Header + Alignment   = 0x0        (0)
File Header Length   = 0x400      (1024 Bytes)
USB Product ID       = 0x1234
USB Vendor ID        = 0x8743
Hardware ID          = 0x100
Firmware ID          = 0x100
Unknown Field #1     = 0x1        (1)
Unknown Field #2     = 0x400      (1024)
Number of Files      = 0x30       (48)
Unknown Field #3     = 0x400      (1024)
```

Since the file is **little-endian**, the data may appear **differently** when viewed in **hexadecimal**.

* **Header Size** → size of the global header
* **File Header Length** → size of each file header
* **Number of Files** → total number of files in the image

---

# File Headers

The first file header starts at the **fixed offset `0x400`**, immediately after the 1024-byte global header including garbage.

All files follow the same **header structure**, with most values changing only **minimally**:

```
+------------+--------+----------------------+
| Offset     | Size   | Field                |
+------------+--------+----------------------+
| 0x400      | 4 B    | Filename Length      |
| 0x404      | 4 B    | Header Size          |
| 0x408      | 8 B    | Maintype             |
| 0x410      | 16 B   | Subtype              |
| 0x420      | 4 B    | Unknown0             |
| 0x424      | 256 B  | Filename             |
| 0x524      | 4 B    | Stored Length        |
| 0x528      | 4 B    | Pad #1               |
| 0x52C      | 4 B    | Original Length      |
| 0x530      | 4 B    | Pad #2               |
| 0x534      | 4 B    | File Offset          |
| 0x538-0x7FF| 456 B  | Garbage / Padding    |
+------------+--------+----------------------+
```

* **Filename Length** → length of the filename
* **Header Size** → size of this file header (typically 1024 bytes)
* **Maintype** → main type (e.g., `"COMMON"`)
* **Subtype** → subtype (e.g., `"BOARD_CONFIG_BIN"`)
* **Filename** → file name, null-padded (e.g., `boot.fex`)
* **Stored Length** → stored file size (with padding)
* **Original Length** → actual file size (without padding)
* **File Offset** → starting position of the file data

---

### Expected Values (example: `sys_config.fex`)

```
Filename Length     = 0x100 (256)
File Header Size    = 0x400 (1024)
Maintype            = "COMMON  "
Subtype             = "SYS_CONFIG100000"
Unknown             = 0x0 (0)
Filename            = "sys_config.fex"
Stored Length       = 0x800 (2048)
Pad #1              = 0x0 (0)
File Length         = 0x7BF (1983)
Pad #2              = 0x0 (0)
File Offset         = 0xD800 (55296)
```

* **Stored Length** → file size including padding
* **File Length** → actual file size without padding
* **File Offset** → starting position of the file inside the image

*Garbage-filled file header*

-----

# Overview

In the tests I conducted, images generated with **[imagewty-tool](https://github.com/uictorius/imagewty-tool)** and flashed with **PhoenixSuit** did not show any errors.

```
+------------------------------+
| Global Header (96 bytes)     |
+------------------------------+
| Garbage / Unused (928 bytes) |
+------------------------------+
| File Header #1 (1024 bytes)  |  <- starts at offset 0x400
+------------------------------+
| File Header #2 (1024 bytes)  |
+------------------------------+
| File Header #3 (1024 bytes)  |
+------------------------------+
| ...                          |
+------------------------------+
| File Header #48 (1024 bytes) |
+------------------------------+
|                              |
|   File Data #1               |
|   (size varies)              |
|                              |
+------------------------------+
|                              |
|   File Data #2               |
|   (size varies)              |
|                              |
+------------------------------+
| ...                          |
+------------------------------+
|                              |
|   File Data #48              |
|   (size varies)              |
|                              |
+------------------------------+
```

For now, I recommend reading:

  * **Structure of the Android firmware image:** [http://nskhuman.ru/allwinner/firmware.php?np=1](http://nskhuman.ru/allwinner/firmware.php?np=1)
    *imagewty-tool and these posts are based on the “Structure of the Android firmware image”.*

May also be useful:

  * **Purpose of files in the firmware image:** [http://nskhuman.ru/allwinner/firmware.php?np=2](http://nskhuman.ru/allwinner/firmware.php?np=2)
  * **Partitions: How the sections are used:** [http://nskhuman.ru/allwinner/firmware.php?np=4](http://nskhuman.ru/allwinner/firmware.php?np=4)
  * **GPT Partition Table:** [http://nskhuman.ru/allwinner/firmware.php?np=3](http://nskhuman.ru/allwinner/firmware.php?np=3)

-----

## Next post

In the next post, I will cover the files **extracted** from the image and how to modify them, such as **boot.fex** and **super.fex**.

```
$ lsarisc.fex          dlinfo.fex         sys_config.fex     Vboot-resource.fex
aultls32.fex       dtbo.fex           sys_partition.fex  Vdtbo.fex
aultools.fex       env.fex            toc0.fex           vendor_boot.fex
board.fex          fes1.fex           toc1.fex           Venv.fex
boot0_nand.fex     misc.fex           u-boot-crash.fex   Vmisc.fex
boot0_sdcard.fex   Reserve0.fex       u-boot.fex         vmlinux.fex
boot.fex           split_xxxx.fex     usbtool_crash.fex  VReserve0.fex
boot_package.fex   sunxi.fex          usbtool.fex        Vsuper.fex
boot-resource.fex  sunxi_gpt.fex      vbmeta.fex         Vvbmeta.fex
cardscript.fex     sunxi_mbr.fex      vbmeta_system.fex  Vvbmeta_system.fex
cardtool.fex       sunxi_version.fex  vbmeta_vendor.fex  Vvbmeta_vendor.fex
config.fex         super.fex          Vboot.fex          Vvendor_boot.fex
```