# Synology DSM 7 - Compiling DVB Linux Kernel Modules
I'm writing this since it could be helpful to everyone trying to make the DVBSky T982 DVB-C card work on a V1000 Synology running DSM 7 and above.

Specs:

- Model: Synology DS1821+ 
- Arch: x86_64
- Core Name: v1000
- OS: DSM 7.2.2-72806 Update 3
- Linux Kernel 4.4.302+
- The PCI-DVB device I'm using is a DVBSky T982 PCIe card 
  - This device is reported to be supported inside the Linux Kernel from 3.19, but the required modules are not provided by DSM kernel. So, we'll have to compile the kernel modules ourselves.

This guide will use the `media_build` repo from [**Linuxtv**](https://git.linuxtv.org/media_build.git) to compile the kernel modules. This repo backports patches to use more recent devices in legacy kernels. This repo is EOL, but it should work for a myriad of devices, and there are no other alternatives to my knowledge.


## Acknowledgements

This is a fork of the [**Embedded-MediaDrivers**](https://github.com/b-rad-NDi/Embedded-MediaDrivers) repo by [**@b-rad-NDi**](https://github.com/b-rad-NDi). I simply added support for the V1000 architecture.

I combined it with the information by [**@belgio99**](https://github.com/belgio99) in his [**dsm7-dvb-drivers**](https://github.com/belgio99/dsm7-dvb-drivers) repository.

Further, I want to thank [**@th0ma7**](https://github.com/th0ma7) for the work on [**his Synology repo**](https://github.com/th0ma7/synology).

They all made my life so much easier for compiling and installing the kernel modules with the Synology toolchain. I used their work as a base for this guide.

## Warning
I cannot, and will not, be held responsible for any damage to your Synology. 

Moreover, this guide is provided as-is: I can't guarantee that it will work for other architectures, in fact it's known that **there are** architectures that have problems with compiling the kernel modules.

I'm just sharing what worked for me and my v1000 architecture.

## Prerequisites
- A Synology v1000 architecture NAS with DSM 7 installed
- A DVB device supported by the Linux Kernel (check [**here**](https://www.linuxtv.org/wiki/index.php/Category:Hardware) for reference)

# Step 1: Prepare the build environment
First of all, we need to prepare the build environment. 
I chose to do everything in a Docker container, to keep my system clean and tidy.

So I spun up an Ubuntu 22.04 container:

```console
root@NAS# docker run -it --platform linux/amd64 --name dsm7-dvb-build ubuntu:latest bash
```
   
Now update and install the required packages:

```console
root@BUILD_ENV# apt update
root@BUILD_ENV# apt install build-essential ncurses-dev bc libssl-dev libc6-i386 curl libproc-processtable-perl git wget kmod
```

Then, create a folder with all the tools needed for the compilation:
   
```console
root@BUILD_ENV# mkdir /compile
root@BUILD_ENV# cd /compile
root@BUILD_ENV# git clone https://github.com/Snafu/Embedded-MediaDrivers
root@BUILD_ENV# cd Embedded-MediaDrivers
```

This will clone this Embedded-MediaDrivers repo, which contains the tool you'll use to compile the kernel modules.

# Step 2: Configure the download of the toolchain and the GPL sources
The toolchain and GPL sources are downloded automatically when configuring for V1000 archtitecture. You simply need to change the variables `KERNEL_URI` and `TOOLCHAIN_URI` in the beginning of `configs/SYNO-V1000.conf`.

## Synology Toolchain
Look for the latest toolchain (filname starting with `v1000-gcc`) in the subfolder of the DSM version you want to buid for: https://archive.synology.com/download/ToolChain/toolchain
Copy the link to the file and set the variable `TOOLCHAIN_URI` to that value.

## GPL Sources
Look for the GPL sources (filename starting with `linux-`) in the subfolder of the DSM version you want to buid for: https://archive.synology.com/download/ToolChain/Synology%20NAS%20GPL%20Source 
To find the required version run `uname -r` first. In my case I needed `linux-4.4.x.txz`.
Copy the link to the file and set the variable `KERNEL_URI` to that value.

# Step 3: Build the Synology Linux kernel
## Compile the kernel and media modules
Run the following command:
```console
root@BUILD_ENV# ./md_builder.sh -g -b -d SYNO-V1000
```
to compile the Synology kernel and media modules. This takes a while. The script automatically checks out the latest working commit and applies a required patch.

# Step 4: Installing the kernel modules
To get the modules onto your NAS, exit the shell of the docker container to return to your NAS shell and run the following:
```console
root@BUILD_ENV# exit
root@NAS# [ ! -e /lib/modules/$(uname -r) ] && ln -s /lib/modules /lib/modules/$(uname -r)
root@NAS# [ ! -e /sbin/depmod ] && ln -s /usr/bin/kmod /sbin/depmod
root@NAS# cd /lib/modules/ && docker exec -i dsm7-dvb-build /bin/bash -c "cd /lib/modules/$(uname -r)/; tar -cf - kernel" | tar -xvf -
root@NAS# cd /lib/modules && depmod -a

```
This first creates a symlink `/lib/modules/$(uname -r)` (in my case `/lib/modules/4.4.302+`) to `/lib/modlues`, which is required so depmod can find the modules, in case it doesn't exist yet.
Then it copies the modules from the build environment to `/lib/modules/kernel` using tar as a middleman. 
Finally it runs depmod to create the dependency files required by modprobe.

# Step 5: Device firmware loading
Create the file `/usr/local/etc/rc.d/load-modules.sh` to load the kernel modules at startup, then restart your NAS.
```bash
#!/bin/sh

case $1 in
start)
        echo "Loading DVB core module"
        modprobe dvb-core
        modprobe cx23885
        ;;
        stop)
        # nothing
        ;;
        *)
        echo "Usage: $0 {start|stop}"
        exit 1
        ;;
esac

exit 0
```
In my case, I also needed to load the firmware for my device. In fact, `dmesg` was reporting the following error:
```console
root@NAS# dmesg
...
[164482.377077] si2157 9-0060: found a 'Silicon Labs Si2157-A30 ROM 0x50'
[164482.377134] si2157 10-0063: found a 'Silicon Labs Si2157-A30 ROM 0x50'
[164482.377154] si2157 10-0063: Direct firmware load for dvb_driver_si2157_rom50.fw failed with error -2
[164482.377155] si2157 10-0063: Falling back to user helper
[164482.380751] si2157 10-0063: error -11 when loading firmware
...
```

This was due to missing firmware: I downloaded the correct firmwares from the [**CoreELEC repo**](https://github.com/CoreELEC/dvb-firmware/tree/master/firmware) 
and put it in the `/lib/firmware` folder of my NAS.
For my device, I needed the firmware file `dvb-tuner-si2157-a30-01.fw`, which I had to rename to `dvb_driver_si2157_rom50.fw` to make it work.


# Step 6: Enjoy!
To me, this "unsupported" device is now working flawlessly. I'm using it with a TVHeadend docker container from [linuxserver.io](https://docs.linuxserver.io/images/docker-tvheadend/). I had to alter the container config file manually to add the `/dev/dvb/adapterX/frontent0` et al device files to the container, after creating it using the ContainerManager Synology app. There is also a [SynoCommunity package](https://synocommunity.com/package/tvheadend), however, I never tested that.
