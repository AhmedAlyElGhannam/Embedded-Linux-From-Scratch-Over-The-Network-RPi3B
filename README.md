# Embedded Linux From Scratch Over The Network

## Introduction
In this project, I tried to utilize some of the knowledge I have gained in my journey through the intensive Embedded Linux Course taught by the brilliant [Eng. Fady Khalil] as a part of ITI's 9-Month Internship Program Embedded Systems Track.

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
1. Create a cross toolchain using Crosstool-NG &rarr This provides a cross toolchain + Sysroot libraries for the target.
1. Cross-build U-boot &rarr U-boot then acts as the bootloader needed to load the kernel, DTB, pass bootargs,... etc.
1. Cross-build Linux &rarr This gives a Linux Image for the target + Device Tree Binary.
1. Cross-build rootfs binaries and init process &rarr This along with creating the right directory structure (and populating it with sysroot libraries) gives a fully functional root filesystemwith an init process to initialize the system.

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

1. The build process took about 30 minutes.

![]() 

1. Add the newly-created toolchain path to `.bashrc` and source it.
    ```bash
    echo 'PATH="$HOME/x-tools/aarch64-OverTheNetwork/bin:$PATH"' >> $HOME/.bashrc
    ```

## Building & Customizing U-Boot

