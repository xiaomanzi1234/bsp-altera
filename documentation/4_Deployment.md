# Enclustra Build Environment - User Documentation

## Deployment

This chapter describes how to prepare the hardware to boot from different boot media, using the binaries generated from the build environment.

All the guides in this section require the user to build the required files for the chosen device, with the build environment, as described in the previous section. Once the files are built, they can be deployed to the hardware as described in the following sub sections.

> **_Note:_**  Default target output directories are named according to the following directory naming scheme: `out_<timestamp>_<module>_<board>_<bootmode>`

As a general note on U-Boot used in all the following guides: U-Boot is using variables from the default environment. Moreover, the boot scripts used by U-Boot also rely on those variables. If the environment was changed and saved earlier, U-Boot will always use these saved environment variables on a fresh boot, even after changing the U-Boot environment. To restore the default environment, run the following command in the U-Boot command line:

```
env default -a
```

This will not overwrite the stored environment but will only restore the default one in the current run. To permanently restore the default environment, the `saveenv` command has to be invoked.

> **_Note:_**  A `*** Warning - bad CRC, using default environment` warning message that appears when booting into U-Boot indicates that the default environment will be loaded.

Boot storage | Offset | Size
--- | --- | --- | ---
MMC | partition 1 (FAT) | 0x80000
eMMC | partition 1 (FAT) | 0x80000
QSPI | 0x180000 | 0x80000


### SD Card Partitioning

Run fdisk tool:

```
fdisk /dev/sdX
# where X is the letter of the device
```

Within fdisk run the following commands:

```
# delete the partition table
o
# create a new partition
n
# choose primary
p
# set number to 2
2
# leave default start sector

# set the size to 2 MiB
+2M
# change the partition type
t
# set type to Intel Boot Partition
a2
# create a new partition
n
p
1
# leave default start sector

# set the size to 100 MiB
+100M
# set the 1st partition type to FAT32
t
1
c
# create a new partition
n
p
3
# leave default start and end sector


# set the 3rd partition type to Linux
t
3
83
# write changes and exit
w
q
```

Format the 1st and 3rd partitions:

```
mkfs.vfat -F 32 -n BOOT /dev/sdX1
mkfs.ext2 -L rootfs /dev/sdX3
# where X is the letter of the device
```


### Storage Multiplexing

Mercury+ AA1 provides QSPI flash, SD card and eMMC flash, but all these memories are connected to the same IO pins of the SoC device. Mercury SA1 shares SD card and eMMC flash with the same IO pins. The active memory is selected according to the configured boot mode. `altera_set_storage` U-Boot command provides a mechanism to switch the memory device in U-Boot temporarily.

Examples:

```
# Switch to SD card
=> altera_set_storage MMC
=> mmc rescan
=> mmc info
Device: SOCFPGA DWMMC
Manufacturer ID: 3
OEM: 5344
Name: SL08G
Tran Speed: 50000000
Rd Block Len: 512
SD version 3.0
High Capacity: Yes
Capacity: 7.4 GiB
Bus Width: 4-bit

# Switch to eMMC
=> altera_set_storage EMMC
=> mmc rescan
=> mmc info
Device: SOCFPGA DWMMC
Manufacturer ID: 70
OEM: 100
Name: W5251
Tran Speed: 52000000
Rd Block Len: 512
MMC version 5.0
High Capacity: Yes
Capacity: 14.3 GiB
Bus Width: 8-bit

# Switch to QSPI
=> altera_set_storage QSPI
=> mmc rescan
dwmci_send_cmd: Timeout on data busy
dwmci_send_cmd: Timeout on data busy
dwmci_send_cmd: Timeout on data busy
dwmci_send_cmd: Timeout on data busy
Card did not respond to voltage select!
=> sf probe
SF: Detected S25FL512S_256K with page size 512 Bytes, erase size 256 KiB, total 64 MiB
```


### SD Card

In order to deploy images to an SD card, perform the following steps as root:

1. Partition your SD Card according to section: [SD Card Partitioning](sd-card-partitioning)

2. Copy all required files to the FAT partition on the SD card:

```
mount /dev/sdX1 /mnt
cp boot.scr fpga.rbf uImage devicetree.dtb /mnt # For Cyclone V devices
cp boot.scr u-boot.img bitstream.itb uImage devicetree.dtb /mnt # For Arria 10 devices
umount /mnt
# where X is the letter of the device
```

3. Unpack the rootfs to the EXT4 partition on the SD card

```
mount /dev/sdX3 /mnt
tar -xf rootfs.tar -C /mnt
umount /mnt
# where X is the letter of the device
```

4. Copy the SPL binary to the boot partition on the SD card

```
dd if=u-boot-splx4.sfp of=/dev/sdX2 # For Arria 10 devices
dd if=u-boot-with-spl.sfp of=/dev/sdX2 # For Cyclone V devices
# where X is the letter of the device
```

If one wants to manually trigger booting from a SD Card, the following command has to be invoked from the U-Boot command line:

```
run mmcboot
```

> **_Note:_**  If `saveenv` command is used in U-boot to save the U-Boot environment, a `uboot.env` file will appear on the FAT partition of the SD card.


### eMMC Flash

On the Mercury SA1-R3 and Mercury AA1+ modules the MMC bus lines are shared between eMMC flash and SD card. It is not possible to simultaneously access the SD card and eMMC memory. In the following approach an SD card is used to transfer the data including all the partitions to the eMMC memory.

1. Prepare 2 bootable SD cards (See section [SD Card Partitioning](sd-card-partitioning) for the steps required to prepare an SD card):
    - One with a default SD card image, which is only used to boot until U-Boot console.
    - The second SD card contains the image to be written to the eMMC flash. Make sure that the image to be written to the eMMC is small enough to fit into the DDR memory. E.g. set a rootfs size of 300Mbyte. The rootfs partition size can be increased in a later step.


2. Boot from the first SD card until U-Boot console

3. Replace the SD card with the one containing the eMMC image

4. Copy the SD card content into the DDR memory (assuming the total size is smaller than 512Mbyte)

```
mmc dev 0
mmc read 0 0 0x100000   # copy 512Mbyte of data (block size = 512bytes)
```

5. Switch to the eMMC memory

```
altera_set_storage EMMC
```

6. Copy the data from the DDR memory to the eMMC memory

```
mmc rescan
mmc write 0 0 0x100000
```

7. Optional: If a bigger rootfs partition is required, it can be increased after booting from eMMC memory into Linux. The data on the disk will be preserved while the partition table is modified.

Run fdisk tool:

```
fdisk /dev/mmcblk0
```

Within fdisk run the following commands:

```
# delete rootfs partition
d
3
# create a new partition
n
p
3
# leave default start and end sector


# set the 3rd partition type to Linux
t
3
83
# write changes and exit
w
q
```


> **_Note:_**  If `saveenv` command is used in U-boot to save the U-Boot environment, a `uboot.env` file will appear on the FAT partition of the eMMC flash.


### QSPI Flash

The QSPI flash can be programmed via JTAG with the vendor tools. An alternative is described following. It requires booting from SD card to update the QSPI flash in U-Boot.

1. Prepare an SD card according to section [SD Card](sd-card)

2. Create a directory on the SD card and copy the required files for QSPI boot to the SD card into this newly created directory. The directory name is assumed `qspi` in the following steps.

3. Boot from SD card until U-Boot console

4. Copy the files from the SD card to the DDR memory and write the data into the QSPI flash

```
sf probe
mmc dev 0
fatload mmc 0:1 0 qspi/u-boot-splx4.sfp
sf update 0 $qspi_offset_addr_spl $filesize

fatload mmc 0:1 0 qspi/u-boot.img
sf update 0 $qspi_offset_addr_u-boot $filesize

fatload mmc 0:1 0 qspi/boot.scr
sf update 0 $qspi_offset_addr_boot-script $filesize

fatload mmc 0:1 0 qspi/devicetreee.dtb
sf update 0 $qspi_offset_addr_devicetree $filesize

fatload mmc 0:1 0 qspi/bitstream.itb    # fpga.rbf for Cyclone V devices
sf update 0 $qspi_offset_addr_bitstream $filesize

fatload mmc 0:1 0 qspi/uImage
sf update 0 $qspi_offset_addr_kernel $filesize

fatload mmc 0:1 0 qspi/uramdisk
sf update 0 $qspi_offset_addr_rootfs $filesize
```


#### QSPI Flash Layouts

Partition | Filename | Offset | Size
--- | --- | --- | ---
U-Boot SPL | u-boot-splx4.sfp | 0x0 | 0x80000
U-Boot | u-boot.img | 0x100000 | 0x80000
U-Boot environment | - | 0x180000 | 0x80000
U-Boot script | boot.scr | 0x200000 | 0x80000
Linux devicetree | devicetree.dtb | 0x280000 | 0x80000
FPGA bitstream | bitstream.itb / fpga.rbf | 0x300000 | 0xd00000
Linux kernel| uImage | 0x1000000 | 0x1000000
Rootfs | uramdisk | 0x2000000 | 0x2000000


Last Page: [Command Line Interface CLI](./3_CLI.md)

Next Page: [Project mode](./5_Project_Mode.md)