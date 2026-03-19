# Jetson-Flash

The following flash process allows the use of Sony IMX708 based MIPI/CSI-2 cameras with the Nvidia Jetson Orin Nano (8GB) module installed on the Jetson Dev Kit (supports 2 cameras), or the Argus Cubesat Carrierboard (supports 4 cameras).

### Create build directory
Following https://docs.nvidia.com/jetson/archives/r36.3/DeveloperGuide/IN/QuickStart.html
1. On a computer running Ubuntu 22.04, download the Jetpack 6.0 / Jetson Linux 36.3 release package and sample file system for your Jetson developer kit from https://developer.nvidia.com/embedded/jetson-linux-r363. Other versions of Ubuntu and Jetson Linux may work but have not been tested.
```
$ mkdir Jetson36.3 && cd Jetson36.3
$ wget https://developer.nvidia.com/downloads/embedded/l4t/r36_release_v3.0/release/jetson_linux_r36.3.0_aarch64.tbz2
$ wget https://developer.nvidia.com/downloads/embedded/l4t/r36_release_v3.0/release/tegra_linux_sample-root-filesystem_r36.3.0_aarch64.tbz2
```
2. Enter the following commands to untar the files and assemble the rootfs:
```
$ tar xf jetson_linux_r36.3.0_aarch64.tbz2
$ sudo tar xpf tegra_linux_sample-root-filesystem_r36.3.0_aarch64.tbz2 -C Linux_for_Tegra/rootfs/

$ export DEVDIR=<install-path>/Linux_for_Tegra
$ cd $DEVDIR
$ sudo ./tools/l4t_flash_prerequisites.sh
$ sudo ./apply_binaries.sh
```

### Create kernel sources subdirectory
Following https://docs.nvidia.com/jetson/archives/r36.3/DeveloperGuide/SD/Kernel/KernelCustomization.html#sd-kernel-kernelcustomization
1. To get the kernel source, run the source_sync.sh script:
```
$ cd $DEVDIR/source
$ ./source_sync.sh -k -t jetson_36.3
```

### Directory patching
Following https://developer.ridgerun.com/wiki/index.php/Raspberry_Pi_Camera_Module_3_IMX708_Linux_driver_for_Jetson#Get_the_driver_patches
1. Edit *Linux_for_Tegra/jetson-orin-nano-devkit.conf* to comment out the following line:
```
OVERLAY_DTB_FILE+=",tegra234-p3768-0000+p3767-0000-dynamic.dtbo";
```
2. Download patches provided by RidgeRun from their GitHub:
```
$ git clone git@github.com:RidgeRun/NVIDIA-Jetson-IMX708-RPIV3.git
```
3. Copy the folder of patches to the build directory by running:
```
$ cp -r  NVIDIA-Jetson-IMX708-RPIV3/patches_orin_nano/patches/ $DEVDIR/source
```
4. Apply the patches to the working build directory:
```
$ cd $DEVDIR/source
$ git apply patches/6.0_orin_nano_imx708_v0.1.0.patch
```

### Argus Carrierboard patches
The changes below only apply to the Argus carrierboard. If using the dev kit, skip to the next step. Undo these steps if patching for the dev kit using the same build directory later on.
1. Replace the following files with the ones in the "Argus Carrierboard" folder in this repo
```
source/hardware/nvidia/t23x/nv-public/overlay/tegra234-camera-rpicam3-imx708.dtsi
source/hardware/nvidia/t23x/nv-public/overlay/tegra234-p3768-camera-rpicam3-imx708.dtsi
```

2. Change the value for `cvb_eeprom_read_size` from 0x100 to 0x0 in file
```
$DEVDIR/bootloader/generic/BCT/tegra234-mb2-bct-misc-p3767-0000.dts
```

### Set up the toolchain
This step and the next are likely not necessary and the .dtbo and .ko files provided in this repo may be used to bypass them. If using the dev-kit (or for any reason where these files need to be re-generated, perform these steps.
1. You will need to download the toolchain to compile the source:
```
$ mkdir $HOME/l4t-gcc
$ cd $HOME/l4t-gcc

$ wget https://developer.nvidia.com/downloads/embedded/l4t/r36_release_v3.0/toolchain/aarch64--glibc--stable-2022.08-1.tar.bz2
$ tar -xf aarch64--glibc--stable-2022.08-1.tar.bz2
```
2. Create aliases:
```
$ export INSTALL_MOD_PATH=$DEVDIR/rootfs/
$ export KERNEL_HEADERS=$DEVDIR/source/kernel/kernel-jammy-src

$ export CROSS_COMPILE=$HOME/l4t-gcc/aarch64--glibc--stable-2022.08-1/bin/aarch64-buildroot-linux-gnu-
```

### Compile the kernel
See note about previous step before continuing.
```
$ cd $DEVDIR/source

$ make -C kernel/kernel-jammy-src/ ARCH=arm64 O=$KERNEL_OUT LOCALVERSION=-tegra CROSS_COMPILE=${CROSS_COMPILE_AARCH64} defconfig
$ make -C kernel/kernel-jammy-src/ ARCH=arm64 O=$KERNEL_OUT LOCALVERSION=-tegra CROSS_COMPILE=${CROSS_COMPILE_AARCH64} menuconfig
```
Be sure that IMX708 is selected and deselect any other IMX driver:
```
Device Drivers  --->
  <*> Multimedia support  --->
      Media ancillary drivers  --->
          Camera sensor devices  --->
              <M> Sony IMX708 sensor support
```
```
$ make -C kernel
$ sudo -E make install -C kernel
$ make dtbs
$ cp nvidia-oot/device-tree/platform/generic-dts/dtbs/* $DEVDIR/kernel/dtb/
$ make modules
$ sudo -E make modules_install
```

### Flash Jetson
1. Put Jetson in recovery mode. The method of doing this will vary depending on the board being used.
```
$ cd $DEVDIR

$ sudo ./tools/kernel_flash/l4t_initrd_flash.sh --external-device nvme0n1p1 \
  -c tools/kernel_flash/flash_l4t_t234_nvme.xml -p "-c bootloader/generic/cfg/flash_t234_qspi.xml" \
  --showlogs --network usb0 jetson-orin-nano-devkit internal
```
Once the flash is complete, the linux desktop should appear on the monitor. Complete account creation on jetson.

### Driver installation
1. Get the Jetson IP address to your board:
```
$ ifconfig
```
2. NOTE: If the Kernel was not compiled locally, use the .dtbo and .ko files from this repo. If it was, copy them over using SCP by running:
```
$ scp $DEVDIR/kernel/dtb/tegra234-p3768-0000+p3767-0000-dynamic.dtbo nvidia@orin_ip_address:/tmp
$ scp $INSTALL_MOD_PATH/lib/modules/5.15.136-tegra/updates/drivers/media/i2c/nv_imx708.ko nvidia@orin_ip_address:/tmp
```
3. On the Jetson, mv the .dtbo and .ko files to their respective locations:
```
$ sudo cp /tmp/tegra234-p3768-0000+p3767-0000-dynamic.dtbo /boot/
$ sudo cp /tmp/nv_imx708.ko /lib/modules/5.15.136-tegra/updates/drivers/media/i2c/
```
4. Modify the extlinux.conf to accept the modified dtb and modules.
```
$ sudo vim /boot/extlinux/extlinux.conf
```
Add the following lines to the file as shown. Do not edit the full file, the PARTUUID field must not be changed
```
nv-auto-config
      FDT /boot/tegra234-p3768-0000+p3767-0000-nv.dtb
      OVERLAYS /boot/tegra234-p3768-0000+p3767-0000-dynamic.dtbo
```
EXAMPLE UPDATED FILE (DO NOT COPY)
```
TIMEOUT 30
DEFAULT primary

MENU TITLE L4T boot options

LABEL primary
      MENU LABEL primary kernel
      LINUX /boot/Image
      INITRD /boot/initrd
      APPEND ${cbootargs} root=PARTUUID=1df9c0d6-778b-44c9-9e8d-f9f0efcecced rw rootwait rootfstype=ext4 mminit_loglevel=4 console=ttyTCU0,115200 firmware_class.path=/etc/firmware fbcon=map:0 net.ifnames=0 nospectre_bhb video=efifb:off console=tty0 nv-auto-config
      FDT /boot/tegra234-p3768-0000+p3767-0000-nv.dtb
      OVERLAYS /boot/tegra234-p3768-0000+p3767-0000-dynamic.dtbo

# When testing a custom kernel, it is recommended that you create a backup of
# the original kernel and add a new entry to this file so that the device can
# fallback to the original kernel. To do this:
#
# 1, Make a backup of the original kernel
#      sudo cp /boot/Image /boot/Image.backup
#
# 2, Copy your custom kernel into /boot/Image
#
# 3, Uncomment below menu setting lines for the original kernel
#
# 4, Reboot

# LABEL backup
#    MENU LABEL backup kernel
#    LINUX /boot/Image.backup
#    INITRD /boot/initrd
#    APPEND ${cbootargs}
```
5. Connect IMX708 cameras and reboot

### Installing necessary Nvidia components
On the desktop, install Nvidia sdkmanager from its website, and run it:
```
https://developer.nvidia.com/sdk-manager
```
1. Make sure Target Hardware is set to "Jetson Orin Nano modules", and the SDK version is "Jetpack 6.0 (rev. 2)". If this version is not listed, You may have to run sdkmanager through a terminal using the following command:
```
$ sdkmanager --archived-versions
```
2. In the options presented, under Target Components, deselect "Jetson Linux", and Select "Jetson Runtime Components", "Jetson SDK Components", and "Jetson Platform Services". Accept the terms and conditions, and continue.
3. In the options presented, select Connection: Ethernet, enter the Jetson IP address found earlier, along with the username and password for the account set up on it.
4. Wait until installation completes. This may take several minutes.

### Running cameras
On the Jetson, run either of the following pipelines:
#### Display (Does not work over SSH)
```
SENSOR_ID=0 
FRAMERATE=14 
gst-launch-1.0 nvarguscamerasrc sensor-id=$SENSOR_ID ! "video/x-raw(memory:NVMM),width=4608,height=2592,framerate=$FRAMERATE/1" ! queue ! nvegltransform ! nveglglessink
```
#### JPEG snapshots
```
SENSOR_ID=0 
FRAMERATE=14
gst-launch-1.0 nvarguscamerasrc num-buffers=1 sensor_id=$SENSOR_ID ! "video/x-raw(memory:NVMM), width=4608, height=2592, framerate=$FRAMERATE/1, format=NV12" ! nvjpegenc ! filesink location=test.jpg
```

## UEFI Fix
Due to a bug in the Jetson Linux 36.3.0 UEFI, the Jetson's may randomly stop booting at power on. This issue is detailed in the following forum post
https://forums.developer.nvidia.com/t/assertion-issue-in-uefi-during-boot/315628
Specifically, this version of Jetson Linux is affected by the bug that causes Assertion 3, as patched here
https://github.com/NVIDIA/edk2-nvidia/pull/113/commits

The following steps are optional but HIGHLY RECOMMENDED. Special thanks to Nvidia Developer Forums user jameskuo for the script that has been slightly modified and included in this repo. 

Look in atf_and_optee_README.txt for more information. This may be found in nvidia-jetson-optee-source.tbz2 in public_source.tbz2. https://developer.nvidia.com/downloads/embedded/l4t/r36_release_v3.0/sources/public_sources.tbz2.

### Install Docker
Following https://github.com/NVIDIA/edk2-nvidia/wiki/Build-with-docker#install-docker
```
$ sudo apt install docker.io
$ sudo usermod -a -G docker ${USER}
# Log out, log back in
```

### Assert Issue Fix
1. Export Environment Variables:
```
$ export CROSS_COMPILE_AARCH64=$HOME/l4t-gcc/aarch64--glibc--stable-2022.08-1/bin/aarch64-buildroot-linux-gnu-
$ export CROSS_COMPILE_AARCH64=$HOME/l4t-gcc/aarch64--glibc--stable-2022.08-1/
```
2. Download and unzip assertIssueFix.zip from this repo:
```
$ cd assertIssueFix
$ ./buildOptee.sh
```
3. Once the script is completed, three new files will be created. Copy these over to your build folder:
```
$ cp patched-tos.img $DEVDIR/bootloader/tos-optee_t234.img
$ cp uefi_Jetson_RELEASE.bin $DEVDIR/bootloader/uefi_jetson.bin
$ cp uefi_StandaloneMmOptee_RELEASE.bin $DEVDIR/bootloader/standalonemm_optee_t234.bin
```
4. Put the Jetson in recovery mode and run the following command to update the UEFI bootloader. This process will not wipe the data on the Jetson:
```
$ cd $DEVDIR
$ sudo ./flash.sh -c bootloader/generic/cfg/flash_t234_qspi.xml jetson-orin-nano-devkit internal
```

Your Jetson should now be ready to use!
