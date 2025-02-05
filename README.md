# Jetson-Flash
## Jetson Orin Nano (8GB + w/ SD Card slot) on Dev Kit board

### Create build directory
Following https://docs.nvidia.com/jetson/archives/r36.3/DeveloperGuide/IN/QuickStart.html
1. Download the Jetpack 6.0 / Jetson Linux 36.3 release package and sample file system for your Jetson developer kit from https://developer.nvidia.com/linux-tegra
2. Enter the following commands to untar the files and assemble the rootfs:
```
$ tar xf ${L4T_RELEASE_PACKAGE}
$ sudo tar xpf ${SAMPLE_FS_PACKAGE} -C Linux_for_Tegra/rootfs/
$ cd Linux_for_Tegra/
$ sudo ./tools/l4t_flash_prerequisites.sh
$ sudo ./apply_binaries.sh
```

### Create kernel sources subdirectory
Following https://docs.nvidia.com/jetson/archives/r36.3/DeveloperGuide/SD/Kernel/KernelCustomization.html#sd-kernel-kernelcustomization
1. To get the kernel source, run the source_sync.sh script:
```
$ cd <install-path>/Linux_for_Tegra/source
$ ./source_sync.sh -k -t jetson_36.3
```

### Directory patching
Following https://developer.ridgerun.com/wiki/index.php/Raspberry_Pi_Camera_Module_3_IMX708_Linux_driver_for_Jetson#Get_the_driver_patches
1. Edit *Linux_for_Tegra/jetson-orin-nano-devkit.conf* to comment out the following line
```
OVERLAY_DTB_FILE+=",tegra234-p3768-0000+p3767-0000-dynamic.dtbo";
```
2. Download contents provided in RidgeRun GitHub by running the command:
```
export DEVDIR=<install-path>/Linux_for_Tegra
git clone git@github.com:RidgeRun/NVIDIA-Jetson-IMX708-RPIV3.git
cd NVIDIA-Jetson-IMX708-RPIV3/patches_orin_nano
```
3. Then you will need the patch in the directory Linux_for_Tegra/source by running:
```
cp -r patches/ $DEVDIR/Linux_for_Tegra/source
```
4. Next, you can then apply the patch and go back to the Linux_for_Tegra directory.
```
cd $DEVDIR/source
git apply patches/6.0_orin_nano_imx708_v0.1.0.patch
cd ..
```
5. Replace the following files with the ones in this repo
```
source/hardware/nvidia/t23x/nv-public/overlay/tegra234-camera-rpicam3-imx708.dtsi
source/hardware/nvidia/t23x/nv-public/overlay/tegra234-p3768-camera-rpicam3-imx708.dtsi
```

### Set up the toolchain
1. You will need to download the toolchain to compile the source:
```
mkdir $HOME/l4t-gcc
cd $HOME/l4t-gcc

wget -O toolchain.tar.xz https://developer.nvidia.com/downloads/embedded/l4t/r36_release_v3.0/toolchain/aarch64--glibc--stable-2022.08-1.tar.bz2
tar -xjf aarch64--glibc--stable-2022.08-1
```
2. Create output directories and aliases:
```
KERNEL_OUT=$DEVDIR/source/kernel_out
MODULES_OUT=$DEVDIR/source/modules_out
mkdir -p $KERNEL_OUT
mkdir -p $MODULES_OUT

export INSTALL_MOD_PATH=$DEVDIR/rootfs/
export KERNEL_HEADERS=$DEVDIR/source/kernel/kernel-jammy-src

export CROSS_COMPILE=$HOME/l4t-gcc/aarch64--glibc--stable-2022.08-1/bin/aarch64-buildroot-linux-gnu-
```

### Flash Jetson
This step and the next (Compile the kernel) can be done in parallel to save time.
1. Put Jetson in recovery mode
```
cd $DEVDIR

sudo ./tools/kernel_flash/l4t_initrd_flash.sh --external-device nvme0n1p1 \
  -c tools/kernel_flash/flash_l4t_t234_nvme.xml -p "-c bootloader/generic/cfg/flash_t234_qspi.xml" \
  --showlogs --network usb0 jetson-orin-nano-devkit internal
```
Once the flash is complete, the linux desktop should appear on the monitor. Complete account creation on jetson.

### Compile the Kernel
```
cd $DEVDIR/source

make -C kernel/kernel-jammy-src/ ARCH=arm64 O=$KERNEL_OUT LOCALVERSION=-tegra CROSS_COMPILE=${CROSS_COMPILE_AARCH64} defconfig
make -C kernel/kernel-jammy-src/ ARCH=arm64 O=$KERNEL_OUT LOCALVERSION=-tegra CROSS_COMPILE=${CROSS_COMPILE_AARCH64} menuconfig
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
make -C kernel
sudo -E make install -C kernel
make dtbs
cp nvidia-oot/device-tree/platform/generic-dts/dtbs/* $DEVDIR/kernel/dtb/
make modules
sudo -E make modules_install
```

### Driver installation
1. Get the Jetson IP address going to your board, connect it by ethernet, run the command shown below and copy the ip address showed in eth0 (inet):

```
ifconfig
```
2. In your computer, copy the files via SSH by running
```
scp $DEVDIR/kernel/dtb/tegra234-p3768-0000+p3767-0000-dynamic.dtbo nvidia@orin_ip_address:/tmp
scp $INSTALL_MOD_PATH/lib/modules/5.15.136-tegra/updates/drivers/media/i2c/nv_imx708.ko nvidia@orin_ip_address:/tmp
```
3. Open a terminal on your Jetson and run:
```
sudo cp /tmp/tegra234-p3768-0000+p3767-0000-dynamic.dtbo /boot/
sudo cp /tmp/nv_imx708.ko /lib/modules/5.15.136-tegra/updates/drivers/media/i2c/
```
4. Modify the extlinux.conf to accept the modified dtb and modules.
```
sudo vim /boot/extlinux/extlinux.conf
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
5. Connect 2x IMX708 cameras and Reboot

### Install nvarguscamerasrc and stream from cameras
1. On the desktop, install sdkmanager. Make sure target hardware is correct. SDK version: Jetpack 6.0 (rev 2).
2. Deselect: Jetson Linux. Select: Jetson Runtime Components. Install.
3. On the Jetson, run the following pipelines:
#### Display
```
SENSOR_ID=0 
FRAMERATE=14 
gst-launch-1.0 nvarguscamerasrc sensor-id=$SENSOR_ID ! "video/x-raw(memory:NVMM),width=4608,height=2592,framerate=$FRAMERATE/1" ! queue ! nvegltransform ! nveglglessink
```
#### JPEG snapshots
```
SENSOR_ID=0 
FRAMERATE=14
gst-launch-1.0 nvarguscamerasrc num-buffers=1 sensor_id=$SENSOR_ID ! 'video/x-raw(memory:NVMM), width=4608, height=2592, framerate=$FRAMERATE/1, format=NV12' ! nvjpegenc ! filesink location=RidgeRun_test.jpg
```
