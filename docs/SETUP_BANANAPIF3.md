# Setting up Banana Pi F3 board (Spacemit K1) for KernelCI/LAVA

Before we get to run tests on banana pi f3 for uboot and linux kernel we need to have the setup ready. Getting the setup ready means that board should be in a state to recieve U-Boot proper, OpenSBI and Linux kernel binaries everytime the board starts. 

For this purpose, the source code of the U-Boot is modified so that FSBL polls for U-Boot proper and OpenSBI binaries to be provided through UART (sz) on start. The modified source code is available in this [repository](https://github.com/alitariq4589/pi-u-boot/tree/v2022.10-k1-v2.0#).

The Bootflow of the banana pi f3 board is `On chip firmware -> U-boot SPL -> OpenSBI -> U-Boot Proper -> Linux Kernel`.

This way the board first switches from M mode to S mode in opensbi and then starts U-Boot which then starts Linux kernel. But for this purpose, you also need a binary of the u-boot which has OpenSBI available inside it. You cannot use OpenSBI separately because it contains the jump address to U-Boot binary in the memory so it has to be a single binary with OpenSBI and U-Boot combined.

This tutorial will first explain how to get the desired files and then it will explain how to flash them.

## Getting FSBL (First stage bootloader) for Banana pi f3

First stage bootloader is also called U-Boot SPL in case of Banana Pi F3. If you build the upstream u-boot, you may not get this.

You can either:

1. Download the `FSBL.bin` file from github releases
OR
2. Build the `FSBL.bin` from source (which is a bit time consuming)

### 1. Downloading the FSBL.bin file

You can download FSBL and all the other related files from [this](https://github.com/alitariq4589/lava-webserver-riscv/releases/tag/manual_release) link.

### 2.  Building the supported U-Boot

Clone the U-boot with support for loading binaries through UART.

```
git clone https://github.com/alitariq4589/pi-u-boot.git
```

Checkout the proper branch

```
git checkout v2022.10-k1-v2.0
```

Download the spacemit toolchain from [this](https://archive.spacemit.com/toolchain/spacemit-toolchain-linux-glibc-x86_64-v1.1.2.tar.xz) link. Extract the toolchain and add the toolchain prefix to the `CROSS_COMPILE` variable (e.g.export CROSS_COMPILE="/home/user0/custom_installed/spacemit-toolchain-linux-glibc-x86_64-v1.0.5/bin/riscv64-unknown-linux-gnu-"). It is prefered to add this variable definition in the ~/.bashrc (or ~/.zshrc) so you dont have to define it everytime.

Build the source code

```
make k1_defconfig ARCH=riscv
make -j$(nproc) ARCH=riscv
```

Note: In case you get OpenSBI availability error, you can get the source code from [Bianbu repository for OpenSBI](https://gitee.com/bianbu-linux/opensbi) on gitee, build and then set the variable `OPENSBI` (e.g. export OPENSBI="/home/user0//opensbi/build/platform/generic/firmware/fw_dynamic.bin")

## Flashing the SD card

You can flash the sd card manually but it is recommended that you first format the sd card to GPT and FAT32 format and then flash the [Bianbu OS](https://archive.spacemit.com/toolchain/) in it (this may seem ambiguous now but it helps a lot because Bianbu OS has all the partitions already named and you only have to flash your binaries on appropriate addresses in the SD card).

After you flash the OS, the structure of the sd card becomes as follows:

![Bianbu_OS_Partitions](/assets/banana-pi-f3-boot.drawio.png)

Here you only have to flash the `FSBL.bin` which you acquired in the previous step. You have flash it exactly starting from 131072 bytes. Following command can be used for this.

```
sudo dd if=./FSBL.bin of=/dev/sda bs=1K seek=128
```

Once this is flashed, you can delete other partitions (useful tip: You can use `gparted` in linux because it also shows the partition names).


At this point the banana pi f3 is ready for kernelci with the flashed SD Card.