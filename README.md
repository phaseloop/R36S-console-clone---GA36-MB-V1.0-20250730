# R36S console clone autopsy

GA36-MB V1.0-20250730

## Anatomy of a fake

This console is next level fake because chip markings say: `RK3326 NACLH04028 2520 P43D126` but inside is much older and shittier Allwinner A33. Yes, I could not believe it myself for a while. EmuElec boot logs show detection of Mali400 GPU which sadly is a proof.

Console runs custom build of EmuElec 4.7 modified for A33 CPU so it's not like even public EmuElec build will work (I tried).

Partition schema is weird because that was the thing back them with ancient uboot. I think similar partition schema target was/is used with Bananapi BPI-M2M also running on Allwinner A33.

TL;DR - Image and backup whole SD card because there is no current way to reflash it with any image or release that will work. Once your original SD card is gone - device is trash.

## Sunxi-kernel

This is main entry to understand WTF everything is that weird and why console uses old kernel and is not compatible with anything else.

https://linux-sunxi.org/Main_Page

While apparently modern kernels can work on A33, probably for performance reasons console vendor decided to use Sunxi Linux kernel.

## No DTB for you

Initially (kernel 3.x) there were no DTB files and Allwinner used script.bin file which was some kind of ancient DTB and was parsed by linux-sunxi kernel.

Apparently this file can be converted back to human readable from and DTB can be recreated manually for this board:

https://www.cnx-software.com/2012/05/06/editing-allwinner-a10-board-configuration-files-script-bin/

`magin.bin` file from device attached to repo.

## Autopsy

### Partition schema

```
Disk r36s-emuelec.img: 50.05 GiB, 53739520000 bytes, 104960000 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x815d9b77

Device            Boot   Start       End  Sectors  Size Id Type
r36s-emuelec.img1      4956160 104855550 99899391 47.6G  b W95 FAT32
r36s-emuelec.img2 *      73728    139263    65536   32M  6 FAT16
r36s-emuelec.img3            1   4956160  4956160  2.4G 85 Linux extended
r36s-emuelec.img5       139264    172031    32768   16M 83 Linux
r36s-emuelec.img6       172032    237567    65536   32M 83 Linux
r36s-emuelec.img7       237568   1810431  1572864  768M 83 Linux
r36s-emuelec.img8      1810432   4956159  3145728  1.5G 83 Linux

```

Partition 1 - roms and savestate files

Partition 2 - magic.bin file, boot logo, battery state images and some font files

Partition 3 - not real partition, extended partition entry for 5, 6, 7 and 8

Partition 5 - uboot config partition (raw, no filesystem)

Partition 6 - kernel image (raw, no filesystem)

Partition 7 - EMUELEC partition with SYSTEM image (squashfs with EmuElec - original EmuElec uses same file)

Partition 8 - some kind of r/w filesystem or overlay. Stores logs, swap file, emulation cores, hostname file (host name is UDT). It seems this is where console stores its config.


### Uboot config

Uboot config is stored raw on separate partition, probably to the ease on configuration. While it's human readable text it is wrapped with a binary prefix (more info on linux-sunxi page).

```
bootdelay=0
bootcmd=run setargs_mmc boot_normal
console=ttyS2,115200
nand_root=/dev/nandd
mmc_root=/dev/mmcblk0p7
init=/init
disk=/dev/mmcblk0p8
loglevel=0
setargs_nand=setenv bootargs console=${console} root=${nand_root} init=${init} d
isk=${disk} ion_cma_512m=8m ion_cma_1g=176m ion_carveout_512m=0m ion_carveout_1g
=150m coherent_pool=4m loglevel=${loglevel} partitions=${partitions}
setargs_mmc=setenv bootargs console=${console} root=${mmc_root} init=${init} dis
k=${disk} ion_cma_512m=8m ion_cma_1g=176m ion_carveout_512m=0m ion_carveout_1g=1
50m coherent_pool=4m loglevel=${loglevel} partitions=${partitions}
boot_normal=sunxi_flash read 40007800 boot;boota 40007800
boot_recovery=sunxi_flash read 40007800 recovery;boota 40007800
boot_fastboot=fastboot
recovery_key_value_max=0x13
recovery_key_value_min=0x10
fastboot_key_value_max=0x8
fastboot_key_value_min=0x2
```

### Kernel

Kernel image is stored as separate partition too - block number is passed in uboot config.

Beginning of kernel image:

```
sun8i
ARMv7 Processor
@ #!
!1C "
#normal standby wakeup src config = 0x%x. 
enable CPU0_WAKEUP_MSGBOX. 

(cut lines for readability)

%s version %s (lxl@lxl) (gcc version 4.6.3 20120201 (prerelease) (crosstool-NG l
inaro-1.13.1-2012.02-20120222 - Linaro GCC 2012.02) ) %s
Linux version 3.4.39 (lxl@lxl) (gcc version 4.6.3 20120201 (prerelease) (crossto
ol-NG linaro-1.13.1-2012.02-20120222 - Linaro GCC 2012.02) ) #493 SMP PREEMPT Mo
n Aug 18 10:47:18 CST 2025


```

### 3 root fs

root fs is located on vfat partition EMUELEC as `SYSTEM` file which is squashfs filesystem. I think theoretically `SYSTEM` can be replaced
by any EmuElec version compiled for ARM A7 instruction set. For example Allwinner H3 or 2+ processors which are supported by EmuElec.

Nothing fancy in root fs.

Most interesting part are bootloader overlays - I'm not sure if those are used.

```
share/bootloader/overlays/sun50i-a64-ir.dtbo
share/bootloader/overlays/sun50i-a64-pine64-audio-board.dtbo
share/bootloader/overlays/sun50i-a64-pine64-wifi-bt.dtbo
share/bootloader/overlays/sun50i-a64-spdif.dtbo
share/bootloader/overlays/sun50i-h5-spdif.dtbo
share/bootloader/overlays/sun50i-h5-tve.dtbo
share/bootloader/overlays/sun50i-h6-ir.dtbo
share/bootloader/overlays/sun50i-h6-spdif.dtbo
share/bootloader/overlays/sun8i-h2-plus-bpi-m2-zero-ethernet.dtbo
share/bootloader/overlays/sun8i-h2-plus-ir.dtbo
share/bootloader/overlays/sun8i-h2-plus-spdif.dtbo
share/bootloader/overlays/sun8i-h3-spdif.dtbo
share/bootloader/overlays/sun8i-h3-tve.dtbo

```

### EmuElec build

Apparently vendor took EmuElec and dependencies (CoreElec/LibreElec) and added custom build target (I suppose H3 target will be compatible).
Building A33 kernel was a different story. Kernel compilation date is modern and one of partitions has "ANDROID" string somewhere so maybe it was some A33 Android for Bananapi build with additional partitions and EmuElec initrd image uploaded on top of it.

Fastboot and recovery key entries in uboot also suggest this is a repurposed Android build of some kind. Maybe it's also the reason why vendor took the easiest route with old kernel instead of porting something to newer one (which theoretically should work).

Screenshot of os-release taken by @lcdyk0517 - discussion: https://github.com/lcdyk0517/arkos4clone/issues/147

<img width="2614" height="1000" alt="527472066-9028aee8-2d6c-4c40-9a1d-cafe06be1eb0" src="https://github.com/user-attachments/assets/50947a49-33aa-4650-bb4a-23ff55e75725" />


