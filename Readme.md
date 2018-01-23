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

## A quick note on compilation
You can safely skip this part if you are experienced with (cross-)compilation.

During compilation, a few environment variables are kept in the terminal they were defined in (with the `export`
keyword). Those export are necessary to compile the code for the right architecture. Hence, it probably is a good
idea to perform all the steps in the same terminal window.

The explanations here assume that you already have a cross-compilation toolchain on your computer. You might want
to install the following candidates: `make`, `git`, and a cross-compiler such as `arm-none-eabi-gcc`. Consult
your distribution's documentation for the right package names.

In the following steps, we generally employ `-j 5`. `-jN` (the space can be ommited) tells make to use
`N` parallel threads to perform the job, which should speed up things quite a bit. It is generally
regarded as a good idea to put the number of CPU cores you have +1, so that they are fully used.
Adjust as needed.

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

We should have been able to compile our own u-boot and use this one, but it didn't work for reasons that remain unknown as of now.
Nevertheless, you probably want U-Boot tools on your computer for the next steps. Either **Install them from your distribution's depots
or do the following**:

Clone https://github.com/MayeulC/bringup-zybo-z7-u-boot-xlnx , and run in the repository:
```
    export ARCH=arm
    export CROSS_COMPILE=arm-none-eabi-
    export PATH=$PWD/tools/:$PATH
    make zynq_zybo_z7_config
    make -j 5
```
Note that if you install `uboot-tools` or equivalent from your distribution, you will still need to perform the first two `export`s
to compile the remaining files.

### zImage

Any kernel with support for the zedboard should work in theory, while in practice we compiled it ourselves. A zImage
is a self-extractible compressed kernel image.

To compile your own, clone the https://github.com/MayeulC/bringup-zybo-z7-linux-xlnx repository, and **assuming you
have kept the same terminal as the one you ran the previous command in** (if that's not the case, cd back and do the
exports again), run:

```
    make xilinx_zynq_defconfig
    make UIMAGE_LOADADDR=0x00008000 uImage modules -j5
```
And grab the generated zImage.

### Device tree

For any kernel to work, we need a device tree. The one from https://github.com/cryptotronix/u-boot-xlnx was used.
In the same repository, and preferably with the same command prompt to keep the environment variables, just run:

```
    make zynq-zybo-z7.dtb
```

And grab the generated `zynq-zybo-z7.dtb` (it should have told you the path).

### Putting it all together

You should have three files by now. Take your SD card, format it with fdisk (something like this, but keep in mind
that this wasn't double-checked):

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

Alternatively, you can also do the following to get a Arch Linux running:
```
    wget http://os.archlinuxarm.org/os/ArchLinuxARM-zedboard-latest.tar.gz
    mkdir tmp && mount /dev/sdX1 tmp
    bsdtar -xpf ArchLinuxARM-zedboard-latest.tar.gz -C mnt
    sync # it is always a good idea to sync after I/O intensive tasks, esp. before removing SD cards
    umount tmp
```

## Final word

Pre compiled, ready-to-use (at least for the zybo z7-10) boot image files are present in the boot/
directory.
You can find the Linux kernel source code at https://github.com/MayeulC/bringup-zybo-z7-linux-xlnx under the GPLv2 license.
