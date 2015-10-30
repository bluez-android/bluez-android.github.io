---
layout: default
---
# BlueZ for Android

[AOSP](https://source.android.com/) with [BlueZ 5](http://www.bluez.org/) integrated as replacement for default Bluedroid Bluetooth stack.

This project is an example on how BlueZ 5 for Android can be integrated with AOSP project.

## Short summary

Supported Android versions:

* 5.0 (`lollipop` branch)
* 4.4.4 (`kitkat` branch)

## Supported devices

* Nexus 4 (`mako` target)
* Nexus 7 2013 (`flo` target)
* Nexus 7 2013 LTE (`deb` target)
* Nexus 5 (`hammerhead` target) 

## Getting Started

> Provided instructions are for Android 5.0 (Lollipop).
> For Android 4.4 (KitKat) use `kitkat` branch for repo and `android-msm-<target>-3.4-kitkat-mr2` for kernel.

### Downloading source
```bash
repo init -u https://github.com/bluez-android/aosp_platform_manifest.git -b lollipop
repo sync
```

### Downloading binary drivers
Additional binary drivers are required to boot AOSP on Nexus devices. Those require accepting EULA and can be downloaded from https://developers.google.com/android/nexus/drivers. Follow instructions for your supported device and Android version of choice and after unpacking place them in vendor folder of downloaded repo.

### Initializing build environment
> Replace `<target>` with appropriate value depending on your device, see [above](#supported-devices).

```bash
source build/envsetup.sh
lunch aosp_<target>-userdebug
```

### Building
```bash
make -j8
```

### Flashing
> You can skip `-w` to keep userdata and cache partitions content, i.e. in case you want just to upgrade software without erasing any data.

```bash
adb reboot bootloader
fastboot flashall -w
```

## Building own kernel

**BlueZ for Android** provides prebuilt kernels for all supported devices. When building own kernel from MSM tree, following configuration is recommended in order for all features to work properly:

* enabled `Enable loadable modules support` (`CONFIG_MODULES`)
* enabled `Bluetooth subsystem support` (`CONFIG_BT`) as module
* enabled `802.1d Ethernet Bridging` (`CONFIG_BRIDGE`)
* backported `hid-generic` driver
* backported `aes-cmac` crypto hash
* Bluetooth subsystem built from backports (with RFCOMM and BNEP protocols support)
  recommended latest development release available from https://www.kernel.org/pub/linux/kernel/projects/backports/
* HCI SMD driver built with backports

To make things easier, we provide required modifications as patches available in [misc](https://github.com/bluez-android/misc) repository and step-by-step instruction below to have own kernel binaries built.

```bash
### download misc repository
git clone https://github.com/bluez-android/misc.git

### download kernel
git clone https://android.googlesource.com/kernel/msm kernel-msm
cd kernel-msm

### prepare kernel sources
git checkout android-msm-"target"-3.4-lollipop-release
git am ../misc/patches-kernel/0001-hid-Backport-hid-generic-driver.patch
git am ../misc/patches-kernel/0002-crypto-add-CMAC-support-to-CryptoAPI.patch
git am ../misc/patches-kernel/0003-crypto-af_alg-properly-label-AF_ALG-socket.patch

### configure kernel
make ARCH=arm CROSS_COMPILE=arm-eabi- <target>_defconfig
scripts/config --enable CONFIG_MODULES
scripts/config --module CONFIG_BT
scripts/config --enable CONFIG_BRIDGE
scripts/config --enable CONFIG_CRYPTO_CMAC
scripts/config --enable CONFIG_CRYPTO_USER_API
scripts/config --enable CONFIG_CRYPTO_USER_API_HASH
scripts/config --enable CONFIG_CRYPTO_USER_API_SKCIPHER
make ARCH=arm CROSS_COMPILE=arm-eabi- oldnoconfig

### build kernel
make ARCH=arm CROSS_COMPILE=arm-eabi- -j8

cd ..

### unpack backports
tar xf backports-<yyyymmdd>.tar.xz
cd backports-<yyyymmdd>

### prepare backports sources
patch -p1 < ../misc/patches-backports/0001-Add-defconfig-bluetooth.patch
# (optional)
patch -p1 < ../misc/patches-backports/0002-Enable-6LOWPAN.patch
# (for Nexus 4 and Nexus 7)
patch -p1 < ../misc/patches-backports/0003-Add-HCI-SMD.patch
# (for Nexus 5)
patch -p1 < ../misc/patches-backports/0004-Add-HCI-H4.patch
patch -p1 < ../misc/patches-backports/0005-Workaround-for-msm-kernel-tree.patch

### configure backprots
make ARCH=arm CROSS_COMPILE=arm-eabi- KLIB_BUILD=../kernel-msm defconfig-bluetooth

### build backports
make ARCH=arm CROSS_COMPILE=arm-eabi- KLIB_BUILD=../kernel-msm -j8
```

Finally, existing kernel should be replaced with `kernel-msm/arch/arm/boot/zImage` and `modules/*.ko` with corresponding files from `backports-<yyyymmdd>/`.

## Other useful links

Google documentation on how to download and build AOSP: http://source.android.com/source/building.html

BlueZ documentation for BlueZ for Android: http://git.kernel.org/cgit/bluetooth/bluez.git/tree/android/README

