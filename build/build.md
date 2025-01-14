---
---
- TOC
{:toc}

# Build trust|me

The steps to build trust\|me are very similar for each flavour of the platform such as core, IDS, etc. The difference between the flavours are the containers installed to trust\|me. To build the trust\|me flavour you're interested in, select the appropriate containers in [the corresponding build step](#include-containers-to-trustme-image).
If you just want to try out trust\|me the core flavour is the best option for you.

> Some build steps are architecture / device dependent. These steps are only necessary for that specific architecture / device.

## Prerequisites

The following prerequisites are necessary for all trust\|me flavours. Please make sure your build host meets these requirements:
   * Build host configuration as described in section [Setup Host]({{ "/" | absolute_url }}setup_host)
   * Sufficient hard disk space, at least 100 GB
   * Sufficient RAM. We tested the build on a VM having 4 GB RAM. However, a build host with less RAM should also work.

## Create a workspace directory
```
mkdir ws-yocto
cd ws-yocto
```

## Initialize workspace
Workspace initialization is done using the **repo** tool and the manifest file for the architecture you want to build for.
Please refer to the following list to select the correct manifest file:

|Manifest file | Description |
|--------------|---------------------------|
|**yocto-arm64-zcu104.xml**|The Xilinx ZCU104 Evaluation Board
|**yocto-x86-trustx-corei7-64.xml**|Any x86 based plattform supporting UEFI

```
repo init -u https://github.com/trustm3/trustme_main.git -b master -m <manifest file>
repo sync -j8
```

## Setup yocto environment
In this step the necessary Yocto layers for your build are added and the bitbake command becomes available. 
Therefore, you have to specify the target architecture and machine for your build.
The following architectures and devices are supported currently:

|Architecture|Machine|
|----|---------------|
|x86| trustx-corei7-64|
|arm64|zcu104-zynqmp|

```
source init_ws.sh out-yocto <architecture> <machine>
```
> This automatically switches to out-yocto

## Optional: Use own PKI
If you want to use an own PKI, place the necessary files in the directory `ws-yocto/out-yocto/test_certificates`.
For information on which files are needed, please refer to the [PKI section](/pki/pki).

## Build PMU firmware
> Xilinx ZCU104 specific

The ZCU104 board needs a fimware file for it's PMU. To generate this file, run the following command
```
bitbake multiconfig:pmu:pmu-firmware
```

## Include containers to trust\|me image
These commands install guest operating systems and containers (e.g. the IDS container) to your trust\|me platform.
To make trust\|me work out of the box, use exactly one of the following commands.
Experienced users may choose to include all containers. However this will require manual configuration of the platform.

Build and include the minimal core container
```
bitbake multiconfig:container:trustx-core
```

Build and include the IDS Trusted Connector container
```
bitbake multiconfig:container:ids
```

Build and include the debian/os installer image (see [example: Using GuestOs debos](../operate.md#example-using-guestos-debos))
```
bitbake multiconfig:container:deb
```

> **experimental**:
Build and include the docker-converter image
(see [example: Using docker-convertos](../operate.md#example-using-docker-convertos))
```
bitbake multiconfig:container:docker-convert
```

## Build trust\|me image
This step builds all necessary packages needed for trust\|me and generates a bootable image that can be deployed to the boot medium of your platform.
In order to do so please refer to the [Deploy section](/deploy/x86)

```
bitbake trustx-cml
```
## Build installer image
If required, a bootable installer image can be created. This image can be used to boot the target platform and install trust\|me to the internal disk as described in the [Deploy section](/deploy/x86)
> Currently, the installer medium is only available for x86 platforms

```
bitbake trustx-installer
```


## Build keytool image for UEFI Secure Boot configuration
> x86 UEFI specific

Create a bootable image containing the KeyTool and the trust\|me secure boot keys.
This image can be used to configure secure boot on your platform as described in section [Deploy](/deploy/x86)
```
bitbake trustx-keytool
```


# Build FAQ
## How to change kernel config
Temporarily
```
bitbake -f -c menuconfig virtual/kernel
bitbake -f virtual/kernel
bitbake -f trustx-cml-initramfs
```

Persistently
* Add file to ws-yocto/meta-trustx/recipes-kernel/linux/files
* Register new file in .bbappend files inside ws-yocto/meta-trustx/recipes-kernel/linux/

## How to sign kernel+initramfs binary manually
> x86 UEFI specific

```
sbsign --key test_certificates/ssig_subca.key \
      --cert test_certificates/ssig_subca.cert \
      --output linux.sigend.efi \
      ws-yocto/out-yocto/tmp/deploy/images/intel-corei7-64/bzImage-initramfs-intel-corei7-64.bin
```

   To boot the signed kernel using UEFI, it should be placed as /EFI/BOOT/BOOTX64.EFI on the UEFI system partition.
