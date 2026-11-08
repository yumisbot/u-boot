.. SPDX-License-Identifier: GPL-2.0+

GA36-MB V1.2 (Allwinner A23 "R36S" clone)
=========================================

Handheld sold as "R36S" with an Allwinner A23 (sun8i), AXP223 PMIC and 512 MiB DDR3.
The board boots goodluckOS (a mainline-based Linux image) from the SD card.

Status: boots goodluckOS to login on hardware with the open SPL (no vendor boot0).
Not supported: display (the A23 MIPI-DSI panel has no U-Boot driver, so no boot logo),
USB, network. Console is UART2 (PB0/PB1), 115200n8.

Build
-----

.. code-block:: bash

    make ga36mb_a23_defconfig
    make CROSS_COMPILE=arm-none-eabi-

Outputs used below: ``spl/sunxi-spl.bin`` (24 KiB, eGON header) and ``u-boot.img``
(legacy uImage, linked at 0x4a000500).

Card layout
-----------

The SPL goes to 8 KiB (sector 16) and ``u-boot.img`` to 40 KiB (sector 80), the sunxi
defaults. Keep ``CONFIG_SYS_MMCSD_RAW_MODE_U_BOOT_SECTOR`` at its default: mainline adds
``U_BOOT_DATA_PART_OFFSET`` (0x10) to it, so an override such as 0x9530 reads the wrong
sector and the SPL jumps into junk.

The kernel is not loaded from a filesystem. ``CONFIG_BOOTCOMMAND`` reads the goodluckOS
Android boot image (kernel with appended DTB) from raw sector 172032 (0x2a000, plus 4 for the
header) and runs ``bootz`` with the DTB at 0x425f8708::

    mmc dev 0; mmc read 0x42000000 0x2a004 0x2fec; bootz 0x42000000 - 0x425f8708

``CONFIG_ARMV7_NONSEC`` is off because the goodluckOS DTB uses the A23 secure-mode SMP
method (no PSCI), so the kernel must be entered in secure SVC.

Flash
-----

Build a 1 MiB image: MBR (keep the one on your card) at 0, ``sunxi-spl.bin`` at
0x2000, ``u-boot.img`` at 0xA000, zeros elsewhere (u-boot.img ends near sector 996), then
write it to the start of the card, with the card unmounted::

    # whole first MiB, including the MBR of the image
    sudo dd if=ga36mb-open-boot-1MiB.img of=/dev/sdX bs=1M count=1 conv=fsync

    # or keep the card's own MBR/partition table
    sudo dd if=ga36mb-open-boot-1MiB.img of=/dev/sdX bs=512 skip=1 seek=1 conv=fsync

Or write the two binaries directly::

    sudo dd if=spl/sunxi-spl.bin of=/dev/sdX bs=1k seek=8 conv=fsync
    sudo dd if=u-boot.img        of=/dev/sdX bs=1k seek=40 conv=fsync

This replaces the vendor boot0 on the card. The partitions and the raw boot image at sector
172032 are not touched.

Expected console output::

    U-Boot SPL ... DRAM: 512 MiB
    Trying to boot from MMC1
    U-Boot ... Model: GA36-MB V1.2 (Allwinner A23)
    ...
    Starting kernel ...
