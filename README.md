# Porting Kali NetHunter on Samsung Galaxy S4 (jflte, GT-I9505) Lineage OS 18.1
NOTE: This guide is a work in progress! It is by no means complete!

This is an updated guide on installing the pentesting platform Kali Nethunter (Full version) on Samsung Galaxy S4 (model GT-I9505), with Lineage OS 18.1.

NOTE: As always, a full backup of your device is recommended before proceeding any further. 

Why not only Kali NetHunter?
Kali Nethunter is not a custom ROM in itself so we are using Lineage OS custom ROM as a foundation. A custom kernel for Lineage OS will be created to allow for extra pentesting functionality.

Prerequisites:
- Your Samsung Galaxy S4 device should be in developer mode.*
- A full backup has been made of the device.
- A relatively recent 64-bit computer running Linux, we will be running Debian-based commands.
- CCache installed on your machine (to speed up subsequent builds if something goes wrong)
- A reasonable amount of RAM (16 GB to build up to lineage-17.1, 32 GB or more for lineage-18.1 and up). The less RAM you have, the longer the build will take. Enabling ZRAM can be helpful.
- A reasonable amount of Storage (200 GB to build up to lineage-17.1, 300 GB for lineage-18.1 and up). You might require more free space for enabling ccache or building for multiple devices. Using SSDs results in considerably faster build times than traditional hard drives.
- A decent internet connection and reliable electricity. :)
- A folder that you can work in (this guide uses "jfltexx-los18")

*This is done by navigating to Settings -> About and tapping on the Build number field 7 times until you receive the notification that developer mode has been enabled. Go back to the main settings page and you will have a new section titled Developer options. Tap on the new Developer options section and enable both the Advanced Reboot and Android Debugging options.

Resources:
LineageOS guide for Samsung Galaxy S4 (GT-i9505): https://wiki.lineageos.org/devices/jfltexx/

Building NetHunter: https://www.kali.org/docs/nethunter/building-nethunter/

Porting NetHunter to new device: https://www.kali.org/docs/nethunter/porting-nethunter/ 

Disclaimer:
I am not very knowledgeable in this field, but I want to save other people's time and potential frustration if they need this done. Therefore I am undertaking this task which leaves room for errors and misunderstandings on my part. More knowledgeable users may very well improve this guide if they find any incorrect information.

## Environment consideration

This guide assumes you are operating in a Debian-based Linux environment.

Open Terminal in your working folder, this is where we will store source code, patches etc.

## Build the custom kernel

The custom kernel is the first step in the installation process. It is necessary if you want the Bluetooth arsenal (https://www.kali.org/docs/nethunter/nethunter-btarsenal/), Wireless attacks like packet injection (with compatible USB network dongle, read more: https://www.kali.org/docs/nethunter/wireless-cards/), and HID attack functionality (https://www.kali.org/docs/nethunter/nethunter-hid-attacks/), for example.



### Finding kernel sources
Let's find the Lineage OS 18.1 source code and clone the repository into our working folder: 
```bash
git clone -b lineage-18.1 https://github.com/LineageOS/android_kernel_samsung_jf.git
```
This download can take a while.

Troubleshooting: This error can happen during the cloning process:
```error: 15234 bytes of body are still expected.10 MiB | 20.21 MiB/s
fetch-pack: unexpected disconnect while reading sideband packet
fatal: early EOF
fatal: index-pack failed
```
I think this is due to my VPN connection using the UDP protocol. Turning off the VPN solved this problem, and perhaps switching to the TCP protocol in your VPN client can help solve this problem too.

### Choosing the compiler/toolchain
The Samsung Galaxy S4 has an ARM 32-bit architecture (ARM32). In order to compile the new kernel from an x86 architecture we need the right tools for the job, a toolchain. We will choose Google's cross-compiler. An ARM64 toolchain will not work for this task.

We want to clone the cross-compiler tools into a folder called "toolchain" in our working directory:
```bash
# clone the repo, it will be empty
git clone https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.9 toolchain
# go to the repo folder
cd toolchain
# use the hash of the commit to restore the commit which actually contains the toolchain
git checkout e9b2ab0932573a0ca90cad11ab75d9619f19c458
```
NOTE: For some reason the compiler has been removed from the repo, resulting in only an "OWNERS"-file in the cloned repo folder. This is why we are using git checkout and the version hash.
Source: https://stackoverflow.com/questions/71281064/google-removed-cross-compilers-for-android-kernel-where-are-they-now
### Set-up the environment
Now we will set up some variables for the compiler to use:
```bash
export ARCH=arm
export SUBARCH=arm
export CROSS_COMPILE=$(pwd)/toolchain/bin/arm-linux-androideabi-
export USE_CCACHE=1
export CCACHE_EXEC=/usr/bin/ccache
ccache -M 30G
```
Awesome! Architecture is set to ARM, and we have pointed to where the cross-compiler is located.

You can view our variables by using export and piping it into grep:
```bash
export -p | grep -E -w -i "ARCH|SUBARCH|CROSS_COMPILE"
```

Hopefully you will see the variables there!

### Patch the kernel source
Change directory into your working folder (i. e. the one with "toolchain", "android_kernel_samsung_jf", and "nethunter-gt-i9505-jfltexx-los18"). 

Move the patches from this git repo into the android kernel folder, example:
```bash
cd nethunter-gt-i9505-jfltexx-los18
mv patch ../android_kernel_samsung_jf
cd ../android_kernel_samsung_jf
```

All good? Let's get on with it!

Resource for patches and how they work (if something goes wrong): https://www.howtogeek.com/415442/how-to-apply-a-patch-to-a-file-and-create-patches-in-linux/

#### Injection patch
This patch enables the mac80211 injection. It works on the LOS16 kernel and is based on the aircrack patch (http://patches.aircrack-ng.org/mac80211.compat08082009.wl_frag+ack_v1.patch).
```bash
patch -p1 < patch/injection/mac80211.compat13102019.wl_frag+ack_v1.patch
```

#### hid patch
This patch enable the hid functionalities. It works on the LOS16 kernel and is based on the Android Keyboard Gadget (https://github.com/pelya/android-keyboard-gadget).
```bash
patch -p1 < patch/hid/hid.patch
```
Possible errors:

"Hunk #9 FAILED at 531": replace "drivers/usb/gadget/f_hid.c" with the file version in the "patch" folder.
NOTE: I don't understand how to properly edit patch files (beyond removing obsolete sections), so we will be replacing files instead.

### Configure the kernel
Before to start the build process, we need to configure the kernel for the device and enable some other functionalities. Basically we need to create the `.config` file used by `make` than we can follow the Kali NetHunter guide (https://gitlab.com/kalilinux/nethunter/build-scripts/kali-nethunter-project/wikis/Modifying-the-Kernel).

#### The hard way
To start the testual menu based on the config for the european version run this:
```bash
cd android_kernel_samsung_jf
mkdir -p ../kernel # the output directory
make O=../kernel clean # as a precaution
make O=../kernel ARCH=arm VARIANT_DEFCONFIG=jf_eur_defconfig lineage_jf_defconfig menuconfig
```
Where:
 * The `ARCH=arm` argument is optional if we have set-up the enviroment.
 * The `VARIANT_DEFCONFIG=jf_eur_defconfig` is specific for the EU version of the Samsung S4
 * The `lineage_jf_defconfig` is the config base for LOS
 * The `manuconfig` start the graphical configurator
When you have finished the work, save the config to `.config`.

#### The easy way
I've already done this work and you can put my configuration as a base for yours specific mod (or future mod) if needed.
```bash
cd android_kernel_samsung_jf
cp patch/defconfig/nethunter_jf_defconfig ./arch/arm/configs/
mkdir -p ../kernel # the output directory
make O=../kernel clean # as a precaution
make O=../kernel ARCH=arm VARIANT_DEFCONFIG=jf_eur_defconfig nethunter_jf_defconfig
```

### Build the new kernel
The last step: building the kernel!

There are two little thing to consider: some warning suppression and the kernel name.

#### Bluetooth patch
Unfortunately the USB bluetooth module doens't compile because there are some declaration error in the source code...
In order to make our kernel compile, we need to apply another patch I've write to fix this issue.
```bash
cd android_kernel_samsung_jf
patch -p1 < patch/bluetooth/bluetooth_compile_fix.patch
```
This patch come from this stackoverflow thread: https://stackoverflow.com/questions/37535720/kernel-compiling-bluetooth-error

#### Warning suppression
The source code have some implicit function declarations that trigger the compiler with some warning. In order to avoid this warning, edit the `Makefile` and comment out or remove the following line:
```bash
cd android_kernel_samsung_jf
nano Makefile
-Werror-implicit-function-declaration \
```
I've already done this work for you (I know, yet another patch):
```bash
cd android_kernel_samsung_jf
patch -p1 < patch/makefile/makefile_warning.patch
```

#### Kernel name
The string that represent the kernel name (e.g. "Linux kali 3.4.112-gb6ef1a5f759-dirty #4 SMP PREEMPT Mon Sep 30 22:44:34 CEST 2019 armv7l") is generated automatically, based on some enviromental variables and must be *unique*. In particular, the alphanumeric string followed by ***-dirty*** is based on the git status of our repository.

To edit this string and avoid the ***-dirty*** we need to commit our changes:
```bash
git add -A
git commit -m "NetHunter Kernel"
```

#### Compiling!
Finally, the compiling step!
```bash
cd android_kernel_samsung_jf
make O=../kernel ARCH=arm -j$(nproc --all) LOCALVERSION="-NetHunter"
```
Where `-j` define the numer of thread to use. the `$(nproc --all)` print the number of processing units available on the machine.

If something goes wrong, install:
bc bison build-essential ccache curl flex g++-multilib gcc-multilib git git-lfs gnupg gperf imagemagick lib32ncurses5-dev lib32readline-dev lib32z1-dev libelf-dev liblz4-tool libncurses5 libncurses5-dev libsdl1.2-dev libssl-dev libxml2 libxml2-utils lzop pngcrush rsync schedtool squashfs-tools xsltproc zip zlib1g-dev

After few minute (depends on your CPU) the shell return and we can see:
```bash
  Kernel: arch/arm/boot/zImage is ready
```
Our new custom kernel for LineageOS 16 that enable Kali NetHunter functionalities is stored in ***kernel/arch/arm/boot/zImage*** and can be flashed on our Samsung Galaxy S4 (GT-I9505 jflte).

Now, because from all system-on-chip (SoC) kernel must support loadable kernel modules and we have some loadable module and we have compile the kernel with `CONFIG_MODULES=y` parameter, we also need to compile it.
```bash
cd android_kernel_samsung_jf
make O=../kernel INSTALL_MOD_PATH="." INSTALL_MOD_STRIP=1 modules_install
```
If everything goes well, we have to remove some unusefull symlink.
```bash
rm ../kernel/lib/modules/$(ls ../kernel/lib/modules)/build
rm ../kernel/lib/modules/$(ls ../kernel/lib/modules)/source
```
Our modules are in `../kernel/lib/modules`.

## Kali NetHunter installer
As described above, Kali NetHunter isn't a ROM but it can be flashed from a custom recovery as zip file.

In order to generate the NetHunter installer for the Samsung Galaxy S4 (GT-I9505 jflte) we need to add the new device and new kernel to the repo.

### Donwload NetHunter project
Before adding new device, we need to dowload the project than initialize it with the devices repository. For this we use the latest release (at this time), the **2019.4**
```bash
git clone -b 2019.4 https://gitlab.com/kalilinux/nethunter/build-scripts/kali-nethunter-project.git
cd kali-nethunter-project/nethunter-installer
./bootstrap.sh
```

This start the bootstrap process that initialize teh devices repo. Use the default answer for all the questions (N = No). This take few minute (based on your connection) to download the devices repository.

### Adding new/unsupported device
As described [here](https://gitlab.com/kalilinux/nethunter/build-scripts/kali-nethunter-devices) the devices descriptions and informations for the builder are stored in `devices.cfg` in the following form:
```bash
# Full device name for LineageOS (or some other ROM)
[codename]
author = "Your Name"
version = "1.0"
kernelstring = "NetHunter kernel or any name to call your kernel"
arch = arm64
ramdisk = gzip
block = /dev/block/WHATEVER/by-name/boot
devicenames = codename codename2_if_it_has_one
```
Where the only important things are:
 * devicenames: the name of the device
So, for our purpose, we need to add the following:
```bash
# Galaxy S4 for LineageOS 16
[jfltexx-los]
author = "Fabio Cogno"
version = "1.0"
kernelstring = "NetHunter kernel for jflte"
devicenames = jflte GT-I9505 i9505 jfltexx
```
As usaul, I've already done this work for you and you can simply apply a patch!
```bash
cd kali-nethunter-project/nethunter-installer/devices
patch -p1 < ../../../patch/devices/devices.cfg.patch
```
Now we need to add our custom kernel. To do this we need to copy the `zImage` in the right place.

The LineageOS 16 is based on Android 9 Pie, so we need to put our kernel image in `pie/jfltexx-los/`.
```bash
cd kali-nethunter-project/nethunter-installer/devices
mkdir -p pie/jfltexx-los
cp ../../../kernel/arch/arm/boot/zImage pie/jfltexx-los/
```
And our modules:
```bash
cp -r ../../../kernel/lib/modules/ pie/jfltexx-los/
```

### Build the installer
Now we can see our new listed device in the Kali NetHunter build utility under the `-d` or `--device` sectione of the help:
```bash
cd kali-nethunter-project/nethunter-installer/
python build.py -h
```
We can see our `jfltexx-los`. Yeah!

At this point we can create a simple installer for test our kernel:
```bash
cd kali-nethunter-project/nethunter-installer/
python build.py -d jfltexx-los --pie -k
```
This command generate a flashable zip like `kernel-nethunter-jfltexx-los-pie-AAAAMMDD_hhmmss.zip`.

Otherwise we can create the full Kali NetHunte installer (***NOTE: Actually it doesn't work due to a bootloop to recovery***):
```bash
cd kali-nethunter-project/nethunter-installer/
python build.py -d jfltexx-los --pie --rootfs full
```
This generate another flashable zip taht contain the Kali NetHunter overlay for Android Pie and our custom kernel (e.g. `nethunter-jfltexx-los-pie-kalifs-full-AAAAMMDD_hhmmss.zip`).
> The first time can take a long time because the script must download the app (the NetHunter terminal emulator, the store, the VNC client and the NetHunter app) and the full rootfs (about 1,3Gb)

## Installing Kali NetHunter
To install the new Kali NetHunter follow this step:
1. Download the LineageOS 16 installer from [lineageos.org](https://download.lineageos.org/jfltexx)
2. Download the Magisk installer from [magiskmanager.com](https://github.com/topjohnwu/Magisk/releases/download/v19.3/Magisk-v19.3.zip)
3. [opt] Download the pico Gapps for ARM Android 9 from [opengapps.org](https://netcologne.dl.sourceforge.net/project/opengapps/arm/20191013/open_gapps-arm-9.0-pico-20191013.zip)
4. Copy LineageOS 16 installer, Magisk installer, Gapps, and Kali NetHunter on an external SD Card.
5. Download and install latest TWRP from [twrp.me](https://dl.twrp.me/jfltexx/) with Odin or Heimdall
6. Reboot into recovery: with the device powered off, hold `Home` + `Volume Up` + `Power`.
7. Tap wipe than advanced wipe and select
  * Dalvik / ART Cache
  * System
  * Data (you loss all your data)
  * Internal Storage (**warning** you loss you system and you can reintall it only from sd card)
  * Cache
8. Flash LineagesOS 16, than flash Gapps, than flash Magisk
9. Reboot into the system and complete the installation (even wi-fi connection)
10. Reboot into recovery, wipe cache, flash NetHunter
