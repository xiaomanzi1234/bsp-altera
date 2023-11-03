# Enclustra Build Environment - User Documentation

## FAQ

### How to script U-Boot?

All U-Boot commands can be automated by scripting, so that it is much more convenient to deploy flash images to the hardware.

For example, QSPI deployment:

Put the following commands as plain text to a file `cmd.txt`:

```
sf probe

echo "SPL"
mw.b 0 0xFF ${size_spl}
tftpboot 0 u-boot-splx4.sfp
sf update 0 ${qspi_offset_addr_spl} ${filesize}

echo "U-Boot"
mw.b 0 0xFF ${size_u-boot}
tftpboot 0 u-boot.img
sf update 0 ${qspi_offset_addr_u-boot} ${filesize}

echo "U-Boot Script"
mw.b 0 0xFF ${size_boot-script}
tftpboot 0 boot.scr
sf update 0 ${qspi_offset_addr_boot-script} ${filesize}

echo "Devicetree"
mw.b 0 0xFF ${size_devicetree}
tftpboot 0 devicetreee.dtb
sf update 0 ${qspi_offset_addr_devicetree} ${filesize}

echo "Bitstream"
mw.b 0 0xFF ${size_bitstream}
tftpboot 0 bitstream.itb # fpga.rbf for Cyclone V devices
sf update 0 ${qspi_offset_addr_bitstream} ${filesize}

echo "Kernel"
mw.b 0 0xFF ${size_kernel}
tftpboot 0 uImage
sf update 0 ${qspi_offset_addr_kernel} ${filesize}

echo "Rootfs"
mw.b 0 0xFF ${size_rootfs}
tftpboot 0 uramdisk
sf update 0 ${qspi_offset_addr_rootfs} ${filesize}

run qspiboot
```

Then generate an image `cmd.img` and put it onto the TFTP server on the host computer like following.

```
mkimage -T script -C none -n "QSPI flash commands" -d cmd.txt cmd.img
cp cmd.img /tftpboot
```

And finally, load the file on the target platform in U-boot and execute it, like this (after step 5 Setup U-Boot connection parameters, in the user documentation):

```
tftpboot 0x10000000 cmd.img
source 0x10000000
```


### How can the flash memory be programmed from Linux?

In order to program a flash memory from Linux, a script like the following one can be used. - All required files need to be present in the current folder. They can be loaded via TFTP or from USB drive / SD card.

```
#!/bin/sh

getsize ()
{
        local  size=`ls -al $1 | awk '{ print $5 }'`
        echo "$size"
}

SPL_FILE="u-boot-splx4.sfp"
U-BOOT_FILE="u-boot.img"
SCRIPT_FILE="boot.scr"
DEVICETREE_FILE="devicetree.dtb"
BITSTREAM_FILE="bitstream.itb"
KERNEL_FILE="uImage"
ROOTFS_FILE="uramdisk"

SPL_OFFSET=0x0
U-BOOT_OFFSET=0x100000
SCRIPT_OFFSET=0x200000
DEVICETREE_OFFSET=0x280000
BITSTREAM_OFFSET=0x300000
KERNEL_OFFSET=0x1000000
ROOTFS_OFFSET=0x2000000

# erase flash device
flash_erase /dev/mtd0 0 0x4000000

# write SPL
FILESIZE=`getsize ${SPL_FILE}`
echo writing SPL file ${SPL_FILE} size ${FILESIZE}
mtd_debug write /dev/mtd0 ${SPL_OFFSET} ${FILESIZE} ${SPL_FILE}

# write U-Boot
FILESIZE=`getsize ${U-BOOT_FILE}`
echo writing U-Boot file ${U-BOOT_FILE} size ${FILESIZE}
mtd_debug write /dev/mtd0 ${U-BOOT_OFFSET} ${FILESIZE} ${U-BOOT_FILE}

# write U-Boot script
FILESIZE=`getsize ${SCRIPT_FILE}`
echo writing U-Boot script file ${SCRIPT_FILE} size ${FILESIZE}
mtd_debug write /dev/mtd0 ${SCRIPT_OFFSET} ${FILESIZE} ${SCRIPT_FILE}

# write devicetree
FILESIZE=`getsize ${DEVICETREE_FILE}`
echo writing devicetree ${DEVICETREE_FILE} size ${FILESIZE}
mtd_debug write /dev/mtd0 ${DEVICETREE_OFFSET} ${FILESIZE} ${DEVICETREE_FILE}

# write bitstream
FILESIZE=`getsize ${BITSTREAM_FILE}`
echo writing bitstream file ${BITSTREAM_FILE} size ${FILESIZE}
mtd_debug write /dev/mtd0 ${BITSTREAM_OFFSET} ${FILESIZE} ${BITSTREAM_FILE}

# write Linux kernel
FILESIZE=`getsize ${KERNEL_FILE}`
echo writing Linux kernel file ${KERNEL_FILE} size ${FILESIZE}
mtd_debug write /dev/mtd0 ${KERNEL_OFFSET} ${FILESIZE} ${KERNEL_FILE}

# write rootfs
FILESIZE=`getsize ${ROOTFS_FILE}`
echo writing rootfs file ${ROOTFS_FILE} size ${FILESIZE}
mtd_debug write /dev/mtd0 ${ROOTFS_OFFSET} ${FILESIZE} ${ROOTFS_FILE}
```

Just make the script executable and execute it like this:

```
chmod +x flash.sh
./flash.sh
```




Last Page: [Updating the binaries](./6_Binaries_Update.md)

Next Page: [Known issues](./8_Known_Issues.md)