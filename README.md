## Preface

For easier tracking and maintenence, we use a clean Android aosp-marlin-7.1.2_r17 checkout as the base for the container-based Android development.

There is also a document available on slideshare for [the brief system architecture of Android Containerization](https://www.slideshare.net/PowenCheng1/android-containerization-in-brief). 

## Downloading The Code

To initialize and sync up your local repository using the CBA trees, use a command like this:

```shell
# create a working directory, say "cba-aosp-marlin-7.1.2_r17"
$ mkdir -p cba/cba-aosp-marlin-7.1.2_r17
$ cd cba-aosp-marlin-7.1.2_r17
$ repo init -u https://github.com/clondroid/cba-aosp-marlin-7.git -b master
$ repo sync
```

### Vendor specific binaries

**!!! IT IS VERY IMPORTANT !!!**, due to license issues, the google and qcom vendor specific driver binaries are not included in CBA code repositories, you have to download and install it by yourselves, please reference CBA project repository "[android_vendor_marlin_7](https://github.com/clondroid/android_vendor_marlin_7)" for how to download and install google and qcom vendor specific driver binaries into CBA source code trees.

### lxc source

The essential lxc distribution has not been integrated into Android build process yet, please reference [lxc-for-Android-7.1.2](https://github.com/clondroid/lxc-for-Android-7.1.2) for how to build lxc by yourself.
As for the lxc packages binaries, they have been integrated into CBA Android source under “vendor/icl”. 

## Building host and container ROM images

Building rom images for host and container is quite straight forward, please follow the steps described below.

- change directory to , say "~/cba/cba-aosp-marlin-7.1.2_r17"
  - "cba/cba-aosp-marlin-7.1.2_r17" will be $CBA_ANDROID_HOME
  - Use the following steps to build host and container images

**[Steps for host]**
```shell
$ cd $CBA_ANDROID_HOME

# we need Open JDK 1.8 for building AOSP, please install it if you haven't
$ export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
$ export PATH=${JAVA_HOME}/bin:${PATH}

$ export USE_CCACHE=1
$ export CCACHE_DIR=$CBA_ANDROID_HOME/../.ccache
$ prebuilts/misc/linux-x86/ccache/ccache -M 50G

$ source build/envsetup.sh
$ lunch aosp_marlin-eng
$ make -j8

#
# ROM images for host Android can be found under $CBA_ANDROID_HOME/out/target/product/marlin/...
#
```

**[Steps for container]**
```shell
$ cd $CBA_ANDROID_HOME

# we need Open JDK 1.8 for building AOSP
$ export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
$ export PATH=${JAVA_HOME}/bin:${PATH}

$ export USE_CCACHE=1
$ export CCACHE_DIR=$CBA_ANDROID_HOME/../.ccache
$ prebuilts/misc/linux-x86/ccache/ccache -M 50G

$ source build/envsetup.sh
$ lunch aosp_marlin_con-eng
$ make -j8

#
# ROM images for host Android can be found under $CBA_ANDROID_HOME/out_con/target/product/marlin/...
#
```

## Flashing host images

Flashing CBA (Container-based Android) images on Pixel XL is quite simple, nothing different from the essential [AOSP ROM images installation](https://source.android.com/setup/running), conceptually, you can follow the following steps to flash CBS images

- [Unlocking the bootloader](https://source.android.com/setup/running#unlocking-the-bootloader)
- [Booting your device into into fastboot mode](https://source.android.com/setup/running#booting-into-fastboot-mode)
- Flashing images

```shell
$ fastboot flash boot $CBA_ANDROID_HOME/out/target/product/marlin/boot.img
$ fastboot flash system $CBA_ANDROID_HOME/out/target/product/marlin/system.img

# erase userdata and cache, dot not flash userdata which will result in data partition size problem
$ fastboot -w

# Flashing vendor partition is required especially when you've upgrade your Pixel XL
# to Android 8.X, the essential Android 7.X vendor partition is not
# compatible with the one of Android 8.X
$ fastboot flash vendor $CBA_ANDROID_HOME/out/target/product/marlin/vendor.img

#
# Reboot the device when ready and finish setup wizard provisioning 
#
$ fastboot reboot
```

**[Note] Please be noted that after rebooting the device, the container won't start, the is is because the container's partitions, like root, userdata, ... are not ready yet.**

# Preparing container's block device images

Before a container can be ready for launching, we need to prepare its required block devices images.

- Creating a directory for locating container's block device images, say "~cba/container.images"
  - ~cba/container.image will be $CONTAINER_IMAGES_HOME

```shell
# install android-tools-fsutil if you haven't
$ sudo apt-get install android-tools-fsutils

$ cd $CONTAINER_IMAGES_HOME

$ simg2img $CBA_ANDROID_HOME/out_con/target/product/marlin/system.img  $CONTAINER_IMAGES_HOME/system.img.raw

# create fresh empty userdata.img
$ dd if=/dev/zero of=$CONTAINER_IMAGES_HOME/data.img.raw bs=1 count=0 seek=8G
$ mkfs.ext4  $CONTAINER_IMAGES_HOME/data.img.raw

# create fresh empty persist data image
$ dd if=/dev/zero of=$CONTAINER_IMAGES_HOME/data-persist.img.raw bs=1 count=0 seek=512M
$ mkfs.ext4  $CONTAINER_IMAGES_HOME/data-persist.img.raw
```

**[Pushing container's block device images]**

```shell
$ adb push $CONTAINER_IMAGES_HOME/system.img.raw  /data/maru/con1/system.img.raw

# The userdata.img is 8G, please be patient when pushing it.
$ adb push $CONTAINER_IMAGES_HOME/data.img.raw  /data/maru/con1/data.img.raw

$ adb push $CONTAINER_IMAGES_HOME/data-persist.img.raw  /data/maru/con1/data-persist.img.raw

#
# Reboot the device
#
$ adb reboot
```

## Switching Between Host and Container

Please launch the prebuild "aSwitch" APP to switch between host and container.
