# Zybo Z7 bringup

## Goal and disclaimer
The aim of this repository is to publish the information we gathered about bringing up the zybo-z7 board during our final school project.

As usual, it is provided without any warranties that the information provided here is useful,
or even accurate, and it might brick your board (although this is unlikely). You can consider
everything there published under the MIT license, uless other licenses apply.

The repository is just intended as a place to dump raw information for now, it is not necessarily proofread, or anything alike.
Please also note that the commit history will probably not be kept clean, and force pushes are an option.

## Board
The board the work is intended for is a Zybo Z7-10 from Digilent. It will probably work on a Z7-20 as well, but that wasn't checked.

## Acknowledgements and sources
The following ressources were used as sources of information during the bringup, and might be useful to you:
* https://github.com/Xilinx/u-boot-xlnx
* https://github.com/DigilentInc/Linux-Digilent-Dev
* https://github.com/cryptotronix/u-boot-xlnx
* https://github.com/musashino-build/openwrt/commit/56348c95e95bac5af77ee2825fe595c8383fd834
* http://mariolizanac.com/posts/linaro-for-the-zybo-board/ (Note: this applies to the previous generation zybo)
* https://github.com/MarioLizanaC/Ready_for_Work_Zybo_Base_System (ditto)
* https://reference.digilentinc.com/reference/programmable-logic/zybo-z7/migration-guide
* http://www.wiki.xilinx.com/Build+Kernel

## Receipe for bring up
As we received the boards late in the development process, and those were changed from zedboards to the current models,
bringup was done as quickly as possible as the deadline was closing, in a "move fast and break things" fashion.
What was done here is not necessarily advisable, but worked for us after countless tries.

The basic process was:

### BOOT.BIN generation

* Create a new project with vivado
* Add the boards packages from https://github.com/Digilent/vivado-boards/ to vivado
* Change the current board as explained in https://reference.digilentinc.com/reference/programmable-logic/zybo-z7/migration-guide
* Rebuild everything, including the bitstream
* Launch the SDK from vivado
* Select tool-> create boot image as explained in http://mariolizanac.com/posts/linaro-for-the-zybo-board/
* Add to partition list:
    * As a bootloader, the fsbl from https://github.com/MarioLizanaC/Ready_for_Work_Zybo_Base_System (even if for the wrong board)
    * As data, the bitstream generated from vivado
    * As data, the uboot elf image from the same repository
* Create and export the BOOT.bin image. Reserve for later (this is a receipe, after all)

### Zimage

Any kernel with support for the zedboard should work in theory, in practice we had more luck with the
zImage from https://github.com/MarioLizanaC/Ready_for_Work_Zybo_Base_System than with our own kernels
(the same can be said about u-boot, and it remains to be seen why this is the case).

### Device tree

For any kernel to work, we need a device tree. The one from https://github.com/cryptotronix/u-boot-xlnx was used.
The device tree was added to xilinx's Linux kernel tree, in https://github.com/MayeulC/bringup-zybo-z7-linux-xlnx
Clone this repository and run:

```
    export ARCH=arm
    export CROSS_COMPILE=arm-none-eabi-
    export echo PATH=/path-to-uboot-source-tree/tools:$PATH
    make xilinx_zynq_defconfig
    make zynq-zybo-z7.dtb
```

Note: you probably want to clone something like https://github.com/MayeulC/bringup-zybo-z7-u-boot-xlnx or install the
u-boot-tools for your distribution.

Usually, at this point, you might want to try to build your kernel with `make UIMAGE_LOADADDR=0x00008000 uImage modules -j`,
but we had no luck with the generated image.

Grab the generated `zynq-zybo-z7.dtb` and reserve it for later.

### Putting it all together

You should have three files by now. Take your SD card, format it with fdisk (something like this, this wasn't double-checked):

```
    fdisk /dev/sdx
    o # This will create a new partition table
    n #new partition
    p # primary
    1 # part id
    8192 # first sector
    +150M # Arbitrary, but take something big enough
    t #change type
    c # hex code for fat
    l # list partitions, take note of the last sector
    n [enter] [enter] # new primary partition with default id (2)
    <the last sector you noted +1>
    [enter]
    w # write the partition table

    mkfs.vfat /dev/sdX1
    mkfs.ext4 /dev/sdX2
```

Now, copy your files on the first partition, you might want to add a `boot.scr` file with the following as well:

```
    fatload mmc 0 0x3000000 zImage
    fatload mmc 0 0x2A00000 zynq-zybo-z7.dtb
    setenv bootargs console=ttyPS0,115200 root=/dev/mmcblk0p2 rw earlyprintk rootfstype=ext4 rootwait
    bootz 0x3000000 - 0x2A00000;
```
Those commands will be executed by uboot during boot, but could be typed as well.

### Userspace and root partition

This is the last missing piece, and one of the easiest. Grab a root filesystem for one of the existing
ARM distributions, and stick it on the second partition.

You can use the contents from the following archive, for example, to have a linaro installation:
https://releases.linaro.org/archive/15.06/ubuntu/vivid-images/nano/linaro-vivid-nano-20150618-705.tar.gz

