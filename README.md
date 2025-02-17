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

