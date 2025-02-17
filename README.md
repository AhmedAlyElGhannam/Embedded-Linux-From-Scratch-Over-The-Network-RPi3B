# Embedded Linux From Scratch Over The Network

## Introduction
In this project, I tried to utilize some of the knowledge I have gained in my journey through the intensive Embedded Linux Course taught by the brilliant [Eng. Fady Khalil](https://github.com/FadyKhalil) as a part of ITI's 9-Month Internship Program Embedded Systems Track.

I will build a Linux From Scratch image for the Raspberry Pi 3B+ but, as a twist, I will be sticking to a theme which is file transfer over the network via protocols like TFTP and NFS. 

### Target Hardware Specifications
As stated earlier, the target hardware is a Raspberry Pi 3B+ 1GB RAM with the following pin configuration. Knowing the pin configuration will be important later on in order to communicate with the Pi via a serial terminal like `gtkterm` using a USB-to-TTL device as shown below.
![](./README_Photos/pinout.jpg)
![](./README_Photos/usb2ttl.jpg)

### Build System Specifications
I have built this Yocto image on my trusty laptop **Thinkchad**: a Lenovo Thinkpad P53 with the specifications stated below. I made sure to record the time it took me to build anything for reference. Bare in mind that performance may differ due to multiple reasons such as: temperature---a laptop is more likely to throttle than a PC; disk speed---I built it entirely on an SSD; and lastly the performance of your machine---I am running a native/real Ubuntu 22.04 machine, not a VM.
![](./README_Photos/drip-info.png)


### Proposed Plan
A typical Embedded Linux From Scratch image is built as follows:
1. Create a cross toolchain using Crosstool-NG &rarr; This provides a cross toolchain + Sysroot libraries for the target.
1. Cross-build U-boot &rarr; U-boot then acts as the bootloader needed to load the kernel, DTB, pass bootargs,... etc.
1. Cross-build Linux &rarr; This gives a Linux Image for the target + Device Tree Binary.
1. Cross-build rootfs binaries and init process &rarr; This along with creating the right directory structure (and populating it with sysroot libraries) gives a fully functional root filesystemwith an init process to initialize the system.

Since I am trying to stick to a theme here, I will be modifying some of the steps in this plan. You will see. :)

## Creating a Cross Toolchain
### Cloning & Installation

1. Clone `crosstool-ng` repository.

    ```bash
    git clone https://github.com/crosstool-ng/crosstool-ng
    ```

1. Navigate into the cloned repository folder, then checkout the release tag `1.26.0`.

    ```bash
    git checkout crosstool-ng-1.26.0
    ```

1. Before diving in, install all the following packages to avoid any unnecessary errors during installation. Modify the command to use your package manager if you do not use a Debian-based distro. 

    ```bash
    sudo apt install autoconf automake bison bzip2 cmake flex g++ gawk gcc gettext git gperf help2man libncurses5-dev libstdc++6 libtool libtool-bin make patch python3-dev rsync texinfo unzip wget xz-utils -y
    ```

1. It might be a good idea to visit [Crosstool-NG's Official Documentation](https://crosstool-ng.github.io/docs/) if you want to use an installation method other than the one used here: I used the `Hacker's Way`.

1. The first thing that needs to be run is `bootstrap` script. This script makes sure that all required packages are installed on your machine before building `crosstool-ng`.

    ```bash
    ./bootstrap
    ```

1. Run the following command to configure the installation of `crosstool-ng` to be local; aka in the repository directory to avoid path/permission shenanigans.

    ```bash
    ./configure --enable-local
    ```

1. Finally, run `make` to build it.

    ```bash
    make 
    ```

### Configuration & Build Process

1. The first step is picking a base configuration for a specific architecture and tweaking it some more. You can view all available one via:

    ```bash
    ./ct-ng list-samples
    ```

1. Alternatively, you can use `| grep` to only view the architectures you want. In this case, I want the one used for Raspberry Pi 3B+.

    ```bash
    ./ct-ng list-samples | grep rpi
    ```

1. Set the configuration for Raspberry Pi 3B+.

    ```bash
    ./ct-ng aarch64-rpi3-linux-gnu
    ```

1. Now, for the fun part: open the TUI configuration menu and configure the following settings by yourself:

    ```bash
    ./ct-ng menuconfig
    ```
    1. `Paths and Misc Options` -> **Disable** Render toolchain read-only.
    1. `Toolchain Options` -> **Change** Tuple's vendor string "OverTheNetwork" (make sure to not use dashes since it upsets crosstool-ng).
    1. `C-Library` -> **Change** C-Library to glibc.

1. Once you are done, press `esc` twice, save and exit.

1. Before building, run the following command to view the number of logical processors on your machine. 

    ```bash
    nproc
    ```
1. To build, simply run `./ct-ng build`.
    ```bash
    time ./ct-ng build -j$(nproc)
    ```

1. The build process took about 20 minutes.

![](./README_Photos/ctng-time.png) 

1. Add the newly-created toolchain path to `.bashrc` and source it.
    ```bash
    echo 'PATH="$HOME/x-tools/aarch64-OverTheNetwork-linux-gnu/bin:$PATH"' >> $HOME/.bashrc
    source $HOME/.bashrc
    ```

## Creating a Bootloader Using U-Boot
### Cloning U-Boot
1. Clone `u-boot` repository.

    ```bash
    git clone https://github.com/u-boot/u-boot.git
    ```

1. Checkout a **working tag.** The one that worked for me was `v2025.01`.

    ```bash
    git checkout v2025.01
    ```
### Configuring & Building U-Boot
1. . I will be using a Raspberry Pi 3B+---which uses the SOC `BCM2837` based on ARM CORTEX-A53, I will be using its default configuration provided by `u-boot` under the name `rpi_3_b_plus_defconfig`. 

    ```bash
    ls configs/ | grep rpi
    # copy the config name named rpi_3_b_plus_defconfig
    ```

1. To apply this configuration, simply:

    ```bash
    make rpi_3_b_plus_defconfig
    ```

1. Enter the TUI configuration menu and apply the following changes:

    ```bash
    make menuconfig
    ```
    1. `Boot Options` -> `Autoboot Options` -> **Change** `delay in seconds before automatically booting` to `10` seconds.
    1. `Command Line Interface` -> **Change** `Shell Prompt` to `Our-Boot>`.

1. Once you are done, press `esc` twice, save and exit.

1. To build `u-boot`, I defined the used toolchain prefix and architecture, in addition to the number of cores assigned to the build process. I added the shell command `time` at the beginning to calculate the time to build it: it took about 30 seconds.

    ```bash
    time make ARCH=arm CROSS_COMPILE=aarch64-OverTheNetwork-linux-gnu- -j12
    ``` 

## Cross-compiling Linux
### Cloning Linux

1. The first step is to clone Linux repository from Github.
    ```bash 
    git clone https://github.com/torvalds/linux.git
    ```

1. Alternatively, if you are building it to a Raspberry Pi, it is more recommended to clone their fork of Linux since it contains their default configuration to ease the building/configuration process.
    ```bash
    git clone https://github.com/raspberrypi/linux.git
    ```

1. Go to the repository's directory and checkout the latest stable branch.
    ```bash
    cd linux
    git checkout rpi-6.6.y
    ```

### Configuring & Building Linux
1. Default configurations are located under `arch/BOARD_ARCH/configs/`. Since I am building the kernel for Raspberry Pi 3B+, my `BOARD_ARCH` is `arm64` and the configuration file is named `bcmrpi3_defconfig`. Apply this configuration by running the following command. It is a good idea to export the environment variables `ARCH` and `CROSS_COMPILE` beforehand with values matching your board. Or, just pass them with every command like I do.
    ```bash
    make bcmrpi3_defconfig ARCH=arm64 CROSS_COMPILE=aarch64-OverTheNetwork-linux-gnu-
    ```

1. You can go into the TUI configuration menu and tweak a few settings. I left it as it is.
    ```bash
    make menuconfig
    ```

1. All that remains is to build the kernel and all board-specific files like overlays and dtb. I specified the kernel image format as `Image` since I want to produce a 64-bit kernel. Building them took about 5 minutes.
    ```bash
    time make -j12 Image dtbs ARCH=arm64 CROSS_COMPILE=aarch64-OverTheNetwork-linux-gnu-
    ```
    ![](./README_Photos/linux-time.png)

1. The kernel can be found under `linux/arch/arm64/boot`, the device tree binary can be found in `linux/arch/arm64/boot/dts/broadcom` under the name `bcm2837-rpi-3-b-plus.dtb`. 


## Rootfs Binaries & Init Process Using Busybox
### Cloning Busybox
1. Start things off by cloning busybox's repository.
    ```bash
    git clone git://busybox.net/busybox.git
    ```
1. Change directory into the cloned busybox and checkout the latest stable branch.
    ```bash
    cd busybox
    git checkout remotes/origin/1_36_stable
    ``` 
### Configuring & Building Binaries
1. Wipe out any previous configuration files that may have made their way here and set the default configuration.
    ```bash
    make distclean
    make defconfig
    ```
1. Enter the TUI configuration menu and adjust the following:
    ```bash
    make menuconfig
    ```
    1. `Settings` -> **Enable** Build static binary (no shared libraries).
    1. `Settings` -> **Set** Cross compiler prefix to `aarch64-OverTheNetwork-linux-gnu-`.
    1. `Networking utilities` -> **Disable** `tc`. **[If You Encounter An Error Related to Networking/tc Package While Building]** 

1. Building busybox is a pretty straight forward process: all you have to do is run
    ```bash
    make -j12
    ```
1. All that is left is to install the built binaries into a directory. By default, the installation folder is called `_install` and located in busybox's directory.
    ```bash
    make install
    ```

## Constructing `boot` && `rootfs`
I created a new directory in `$HOME`: it contains a `rootfs` directory and a `TFTP` directory.

### `boot` Partition
1. Format your sd card into one partition as `fat16`.
1. Move `uboot.bin`, `bcm2837-rpi-3-b-plus.dtb`, and `overlays` folder to it.
1. Move the kernel `Image` file to the TFTP directory created.

### `rootfs` Partition
1. Since I will be using NFS, the rootfs I am building will be just a directory on my laptop. I created a directory with the hierarchy shown below. Populate the directories with the sysroot of the produced toolchain and busybox's output.
    ```bash
    ├── bin
    ├── etc
    │   └── init.d
    ├── lib
    ├── lib64 -> lib
    ├── sbin
    ├── usr
    │   ├── bin
    │   ├── include
    │   │   ├── arpa
    │   │   ├── asm
    │   │   ├── asm-generic
    │   │   ├── bits
    │   │   │   └── types
    │   │   ├── drm
    │   │   ├── finclude
    │   │   ├── gdb
    │   │   ├── gnu
    │   │   ├── linux
    │   │   │   ├── android
    │   │   │   ├── byteorder
    │   │   │   ├── caif
    │   │   │   ├── can
    │   │   │   ├── cifs
    │   │   │   ├── dvb
    │   │   │   ├── genwqe
    │   │   │   ├── hdlc
    │   │   │   ├── hsi
    │   │   │   ├── iio
    │   │   │   ├── isdn
    │   │   │   ├── misc
    │   │   │   ├── mmc
    │   │   │   ├── netfilter
    │   │   │   │   └── ipset
    │   │   │   ├── netfilter_arp
    │   │   │   ├── netfilter_bridge
    │   │   │   ├── netfilter_ipv4
    │   │   │   ├── netfilter_ipv6
    │   │   │   ├── nfsd
    │   │   │   ├── raid
    │   │   │   ├── sched
    │   │   │   ├── spi
    │   │   │   ├── sunrpc
    │   │   │   ├── surface_aggregator
    │   │   │   ├── tc_act
    │   │   │   ├── tc_ematch
    │   │   │   └── usb
    │   │   ├── misc
    │   │   │   └── uacce
    │   │   ├── mtd
    │   │   ├── net
    │   │   ├── netash
    │   │   ├── netatalk
    │   │   ├── netax25
    │   │   ├── neteconet
    │   │   ├── netinet
    │   │   ├── netipx
    │   │   ├── netiucv
    │   │   ├── netpacket
    │   │   ├── netrom
    │   │   ├── netrose
    │   │   ├── nfs
    │   │   ├── protocols
    │   │   ├── rdma
    │   │   │   └── hfi
    │   │   ├── rpc
    │   │   ├── scsi
    │   │   │   └── fc
    │   │   ├── sound
    │   │   │   ├── intel
    │   │   │   │   └── avs
    │   │   │   └── sof
    │   │   ├── sys
    │   │   ├── video
    │   │   └── xen
    │   ├── lib
    │   │   ├── audit
    │   │   └── gconv
    │   │       └── gconv-modules.d
    │   ├── lib64 -> lib
    │   ├── libexec
    │   │   └── getconf
    │   ├── sbin
    │   └── share
    │       ├── i18n
    │       │   ├── charmaps
    │       │   └── locales
    │       └── locale
    │           ├── be
    │           │   └── LC_MESSAGES
    │           ├── bg
    │           │   └── LC_MESSAGES
    │           ├── ca
    │           │   └── LC_MESSAGES
    │           ├── cs
    │           │   └── LC_MESSAGES
    │           ├── da
    │           │   └── LC_MESSAGES
    │           ├── de
    │           │   └── LC_MESSAGES
    │           ├── el
    │           │   └── LC_MESSAGES
    │           ├── en_GB
    │           │   └── LC_MESSAGES
    │           ├── eo
    │           │   └── LC_MESSAGES
    │           ├── es
    │           │   └── LC_MESSAGES
    │           ├── fi
    │           │   └── LC_MESSAGES
    │           ├── fr
    │           │   └── LC_MESSAGES
    │           ├── gl
    │           │   └── LC_MESSAGES
    │           ├── hr
    │           │   └── LC_MESSAGES
    │           ├── hu
    │           │   └── LC_MESSAGES
    │           ├── ia
    │           │   └── LC_MESSAGES
    │           ├── id
    │           │   └── LC_MESSAGES
    │           ├── it
    │           │   └── LC_MESSAGES
    │           ├── ja
    │           │   └── LC_MESSAGES
    │           ├── ko
    │           │   └── LC_MESSAGES
    │           ├── lt
    │           │   └── LC_MESSAGES
    │           ├── nb
    │           │   └── LC_MESSAGES
    │           ├── nl
    │           │   └── LC_MESSAGES
    │           ├── pl
    │           │   └── LC_MESSAGES
    │           ├── pt
    │           │   └── LC_MESSAGES
    │           ├── pt_BR
    │           │   └── LC_MESSAGES
    │           ├── ru
    │           │   └── LC_MESSAGES
    │           ├── rw
    │           │   └── LC_MESSAGES
    │           ├── sk
    │           │   └── LC_MESSAGES
    │           ├── sl
    │           │   └── LC_MESSAGES
    │           ├── sr
    │           │   └── LC_MESSAGES
    │           ├── sv
    │           │   └── LC_MESSAGES
    │           ├── tr
    │           │   └── LC_MESSAGES
    │           ├── uk
    │           │   └── LC_MESSAGES
    │           ├── vi
    │           │   └── LC_MESSAGES
    │           ├── zh_CN
    │           │   └── LC_MESSAGES
    │           └── zh_TW
    │               └── LC_MESSAGES
    └── var
        └── db
    ```

### Server Setup
#### TFTP Server
1. Install `tftpd-hpa` 
    ```
    sudo apt install tftpd-hpa -y
    ```

1. Set an IP address for your ethernet port.
    ```
    sudo ip a add 192.168.1.3/24 dev enp0s31f6
    ```

1. Edit the configuration file of `tftpd-hpa` located in `/etc/default/` under the name `tftpd-hpa`.
    ```bash
    sudo vim /etc/default/tftpd-hpa
    #write inside the file
    tftf_option = “--secure --create”
    # change the server directory
    TFTP_DIRECTORY="/home/nemesis/OverTheNetwork/TFTP"
    ```

1. Restart `tftp` service and check its status.
    ```bash
    systemctl restart tftpd-hpa
    systemctl status tftpd-hpa
    ```
#### NFS Server
1. Install `nfs-kernel-server`
    ```bash
    sudo apt install nfs-kernel-server -y
    ```

1. Edit the configuration file:
    ```bash
    sudo vim /etc/exports
    # Add the following line (rootfs location + RPi IP + additional args)
    /home/nemesis/OverTheNetwork/rootfs 192.168.1.2(rw,no_root_squash,no_subtree_check)
    # save and exit
    ```

1. Apply the settings:
    ```bash
    sudo exportfs -r
    ```

### Final Touches
Last thing that remains is to insert the sd card into the Pi and connect to it via a USB-To-TTL. Then, the fun part begins!


## Booting Into The Pi Step-By-Step
1. Upon powering on the Pi, it will boot into U-Boot. Interrupt autoboot by pressing any key.
1. Set the Pi's IP.
    ```
    setenv ipaddr 192.168.1.2
    ```
1. Set the tftp server IP.
    ```
    setenv serverip 192.168.1.3
    ```
1. Load Image && DTB to DRAM via tftp.
    ```
    tftpboot $kernel_addr_r Image
    tftpboot $fdt_addr_r bcm2837-rpi-3-b-plus.dtb
    ```
1. Set bootargs.
    ```
    setenv bootargs "8250.nr_uarts=1 console=ttyS0,115200n8 root=/dev/nfs ip=192.168.1.2:::::eth0 nfsroot=192.168.1.3:/home/nemesis/OverTheNetwork/rootfs,nfsvers=3,tcp rw init=/linuxrc"
    ```
1. Boot into kernel.
    ```
    booti $kernel_addr_r - $fdt_addr_r
    ```

It should load successfully after that!
