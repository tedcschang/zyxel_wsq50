# Zyxel WSQ50 (Multy X) OpenWrt Installation Guide

This document provides step-by-step instructions for backing up factory partitions, unlocking the bootloader, flashing OpenWrt U-Boot, and installing OpenWrt via TFTP and Sysupgrade on the **Zyxel WSQ50 (Multy X)**.

> \[!WARNING\]
> Flashing bootloaders and modifying partition tables can **brick** your device. Proceed at your own risk. Ensure stable power and verify commands before execution.

## Prerequisites

* **Serial Console**: 3.3V TTL USB-to-UART adapter connected to the WSQ50 serial pins.

* **TFTP Server**: Running on your PC (e.g., `192.168.1.99`).

* **Required Files**:

  * `aten_tool.sh` (Seed password calculator for Zyxel zloader)

  * `openwrt-ipq40xx-u-boot-stripped.wsq50.elf`

  * `openwrt-*-ipq40xx-generic-zyxel_wsq50-initramfs-uImage.itb`

  * `openwrt-*-ipq40xx-generic-zyxel_wsq50-squashfs-sysupgrade.bin`

* **USB Flash Drive**: Formatted to FAT32 (for storing backups).

## 1. Stock Firmware Information & Partition Backup

1. Connect to the serial console (115200 8N1).

2. Boot into the stock firmware and log in:

   * **Username**: `root`

   * **Password**: `1234`

3. Check and record factory system information (important!! keep it in safe place):

   ```
   atsh
   
   ```

4. View the MTD layout:

   ```
   cat /proc/mtd
   
   ```

   *Typical output:*

   ```
   dev:    size   erasesize  name
   mtd0: 00060000 00010000 "0:QSEE"
   mtd1: 00080000 00010000 "u-boot"
   mtd2: 00010000 00010000 "env"
   mtd3: 00010000 00010000 "0:ART"
   mtd4: 00010000 00010000 "dualflag"
   mtd5: 00010000 00010000 "CRT"
   mtd6: 00260000 00010000 "reserved"
   
   ```

5. **Backup ART and CRT partitions** (Critical for wireless calibration, important!! keep it in safe place):

   ```
   dd if=/dev/mtd3 of=/tmp/ART.bin
   dd if=/dev/mtd5 of=/tmp/CRT.bin
   
   ```

6. Insert your USB flash drive, mount it, copy `ART.bin` and `CRT.bin` to the drive, and store them safely.

## 2. Unlock Zyxel Bootloader (zloader)

1. Reboot the router:

   ```
   reboot
   
   ```

2. When prompted on the serial console, press ESC repeatedly to stop the boot sequence and enter the `zloader` prompt:

   ```
   WSQ50>
   
   ```

3. Retrieve the seed code:

   ```
   WSQ50> ATSE WSQ50
   012345678901
   
   ```

4. On your PC, generate the unlock code using `aten_tool.sh`:

   ```
   $ ./aten_tool.sh 012345678901
   ATEN 1,879C711
   
   ```

5. Enter the generated command in the console to unlock extended commands:

   ```
   WSQ50> ATEN 1,879C711
   
   ```

6. Return to the master U-Boot console:

   ```
   WSQ50> ATGU
   WSQ50#
   
   ```

7. Verify the SPI flash layout:

   ```
   WSQ50# smeminfo
   
   ```

## 3. Flash OpenWrt U-Boot into SPI NOR Flash

> \[!CAUTION\]
> Double-check the load address and size. Any interruption or error in this step may cause a permanent brick.

1. Ensure `openwrt-ipq40xx-u-boot-stripped.wsq50.elf` is placed in your TFTP server root directory (`192.168.1.99`).

2. Load and write the new U-Boot:

   ```
   WSQ50# tftpboot openwrt-ipq40xx-u-boot-stripped.wsq50.elf
   WSQ50# sf probe
   WSQ50# sf erase 0xe0000 0x80000
   WSQ50# sf write ${loadaddr} 0xe0000 ${filesize}
   
   ```

3. Reboot into the new OpenWrt U-Boot:

   ```
   WSQ50# reset
   
   ```

4. Press ESC to halt boot and stay at the U-Boot prompt.

5. Configure the default boot command:

   ```
   WSQ50# setenv bootcmd bootipq
   WSQ50# saveenv
   
   ```

## 4. Boot Initramfs Recovery Image

1. Load the OpenWrt initramfs recovery ITB image via TFTP:

   ```
   WSQ50# tftpboot openwrt-snapshot-r36407-14651b9683-ipq40xx-generic-zyxel_wsq50-initramfs-uImage.itb
   WSQ50# bootm
   
   ```

2. Wait for OpenWrt initramfs to boot to the shell (`root@OpenWrt:~#`).

## 5. Partition and Format eMMC Storage

Re-partition the internal eMMC disk (`/dev/mmcblk0`) with GPT layout.

1. Create partitions using `parted`:

   ```
   parted -s -a optimal /dev/mmcblk0 \
       mklabel gpt \
       mkpart primary 4MiB 20MiB \
       name 1 "0:HLOS" \
       mkpart primary 20MiB 36MiB \
       name 2 "0:HLOS_1" \
       mkpart primary 36MiB 164MiB \
       name 3 "rootfs" \
       mkpart primary 164MiB 292MiB \
       name 4 "rootfs_1" \
       mkpart primary ext4 292MiB 100% \
       name 5 "rootfs_data"
   
   ```

2. Format the `rootfs_data` partition (`/dev/mmcblk0p5`) as ext4:

   ```
   mkfs.ext4 -F -L rootfs_data /dev/mmcblk0p5
   
   ```

3. Reboot the device:

   ```
   reboot
   
   ```

## 6. Flash Permanent Sysupgrade Firmware

1. Interrupt boot again into U-Boot and load the initramfs image one more time:

   ```
   WSQ50# tftpboot openwrt-snapshot-r36407-14651b9683-ipq40xx-generic-zyxel_wsq50-initramfs-uImage.itb
   WSQ50# bootm
   
   ```

2. Once booted, connect your PC to the LAN port and access LuCI WebGUI:

   * **URL**: `http://192.168.1.1`

   * **Username**: `root` (no password by default)

3. Navigate to **System** $\rightarrow$ **Backup / Flash Firmware**.

4. In the **Flash new firmware image** section, select and flash:
   `openwrt-*-ipq40xx-generic-zyxel_wsq50-squashfs-sysupgrade.bin`

5. Keep settings unchecked (clean installation) and proceed.

6. The router will write the firmware to the eMMC and reboot into your permanent OpenWrt system.
