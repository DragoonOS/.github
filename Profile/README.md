# DragoonOS
A fork of GrapheneOS, designed for power users who want more control and possible play integrity API bypass.

## What's changed?
* Optional root access via KernelSU
* Spoofed build fingerprint to pass cts profile check
* Give user the ability to block any user app from discovering another app (a.k.a. AppFilter)
* Give user the ability to screenshot/screenrecord any app regardless of app's settings (a.k.a. disable_flag_secure)

## Why make this?
I switched from iPhone to Pixel half a year ago, only to be extremely disappointed by the built-in Google bullshit. The Play Store, GMail, Google Maps, Pixel Launcher, and Chrome, all reek of Google's ad bullshit and filth (like news in Chrome's new page). Google has fallen from grace years ago, so I switched to GrapheneOS to find sanctuary, only to be F IN THE A by Google's play integrity API again. Usually I would just install KernelSU and use TrickyStore to bypass it, but GrapheneOS has done something to verify the kernel integrity so simply patching the init_boot.img won't work. Also LSPosed is broken on GrapheneOS so I can't block applist detection or screenshot any app which should be the right of an end user. So I made this fork just to please myself with a clean yet highly customizable OS.

## How to use it?
Currently this is my hobby project and given the highly experimental nature, no binary files will be provided for now. If you would like this to change, please open an issue and once it's popular enough, I'll start to release binaries.

If you're a power user who would like to try it now, a detailed build instruction is available as follows:
### Build environment setup
Install these dependencies on a x86_64 Debian Linux distro
```
apt install repo yarnpkg zip rsync
```

Setup build folder, you might get rate limited by Google when downloading some parts of AOSP code, see [this](https://source.android.com/docs/setup/download/troubleshoot-sync) to setup authenticated access.
```
mkdir grapheneos-16
cd grapheneos-16
repo init -u https://github.com/DragoonOS/platform_manifest.git -b 16
repo sync -j8
```
### Build a custom kernel with root access
If you want root access with KernelSU and is currently using a pixel model other than "tokay", "caiman" or "komodo", you'll need to build your own kernel. Read [this](https://grapheneos.org/build#kernel-6th-through-9th-generation-pixels) for reference.

Acquire the official kernel version string and build time from your stock OS
```
adb shell
uname -r
uname -v
exit
```
Then modify your kernel build script like [this commit](https://github.com/DragoonOS/kernel_common-6.1/commit/00cf81985fce217400c0190c92857e19fb11e940), but with your own version string

Then apply the KernelSU patch by running the following line in your kernel folder
```
cd aosp
curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" | bash -
cd ..
```
Then continue to follow the official GrapheneOS kernel build instructions and copy the build outputs to your kernel prebuilt folder, remember don't overwrite the entire folder but merge instead, as you will still need the kernel-headers folder.

### Acquire device specific vendor files
You do this once to get device specific vendor files for your pixel, replace PIXEL_CODENAME with Pixel device codename (such as komodo for Pixel 9 Pro XL)
```
yarnpkg install --cwd vendor/adevtool/
source build/envsetup.sh
lunch sdk_phone64_x86_64-cur-user
m arsclib
adevtool generate-all -d PIXEL_CODENAME
```

### Acquire the BUILD_NUMBER
For spoofing reasons first acquire a valid BUILD NUMBER, this has to match the version of vendor files you've downloaded in the previous step

You can either acquire this from Google's [site](https://developers.google.com/android/images)
Find the section for your device model and the correct version matching the vendor files you've downloaded previously, then copy the flash link and extract the BUILD NUMBER from it.

For example, the version of vendor files I've downloaded for my device is BP2A.250805.005, I got this flash link ```https://flash.android.com/build/13691446?target=tegu-user&signed``` and the BUILD NUMBER is 13691446.

Alternatively if you're still using the stock pixel OS, you can extract this BUILD NUMBER and also BUILD DATETIME from your system, just make sure your're running the same version of OS as the vendor files you've downloaded previously.

Connect your phone to PC, then run
```
adb shell
getprop | grep "ro.build"
exit
```

The BUILD NUMBER would be in ```ro.build.version.incremental```, and the BUILD DATETIME would be in ```ro.build.date.utc```, it isn't necessary to spoof BUILD DATETIME, but it's good to do it nevertheless. Again make sure you have the same ro.build.id as in your downloaded vendor files.

Now to start building stuff (remember to replace the BUILD_DATETIME and BUILD_NUMBER with your own value, if you acquired BUILD_NUMBER from Google's site just skip the BUILD_DATETIME line)
```
export OFFICIAL_BUILD=false
export BUILD_NUMBER=13691446
export BUILD_DATETIME=1750803747
source build/envsetup.sh
```
### Build the OS
Replace komodo with your own pixel model
```
lunch komodo-cur-user
m vendorbootimage vendorkernelbootimage target-files-package
```
For the Pixel 6, Pixel 6 Pro and Pixel 6a you need this instead
```
m vendorbootimage target-files-package
```
### Sign the OS
After a long long build time, sign your build with your own key, don't forget to replace komodo with your own pixel model
```
mkdir -p keys/komodo
cd keys/komodo
ssh-keygen -t ed25519 -f ./id_ed25519
CN=GrapheneOS
../../development/tools/make_key releasekey "/CN=$CN/"
../../development/tools/make_key platform "/CN=$CN/"
../../development/tools/make_key shared "/CN=$CN/"
../../development/tools/make_key media "/CN=$CN/"
../../development/tools/make_key networkstack "/CN=$CN/"
../../development/tools/make_key sdk_sandbox "/CN=$CN/"
../../development/tools/make_key bluetooth "/CN=$CN/"
openssl genrsa 4096 | openssl pkcs8 -topk8 -scrypt -out avb.pem
../../external/avb/avbtool.py extract_public_key --key avb.pem --output avb_pkmd.bin
cd ../..
```

Now finialize your build, don't forget to replace komodo and BUILD NUMBER with your own
```
m otatools-package
script/finalize.sh
script/generate-release.sh komodo 13691446
```

Now find your factory/ota image in the releases/BUILD_NUMBER/ANOTHER_FOLDER folder, and flash it on your own terms, I recommend re-lock the bootloader after flashing.
If you have any questions, read the [official GrapheneOS build instruction](https://grapheneos.org/build) for reference.

## How to fuck play integrity up?
* First find a valid keybox.xml on your own

* Install [KernelSU manager](https://github.com/tiann/KernelSU/releases/)

* Install any root file manager, I recommend [this](https://f-droid.org/en/packages/me.zhanghai.android.files/)

* Install the TrickyStore module, yeah I know it's closed source now, but nothing bad is in it. For security related details, read [here](TS_SECURITY.md)

* Move your keybox.xml to /data/adb/tricky_store

* Install [PIF Fork](https://github.com/osm0sis/PlayIntegrityFork), create this file ```/data/adb/modules/playintegrityfix/scripts-only-mode``` (and the missing folders) first, then flash the module. This helps by setting the ```ro.boot.verifiedbootstate``` system property to "green" instead of "yellow". You can also make your own module to do this.

* Enjoy strong integrity! Alas, google pay and RCS message use a different integrity check (called droidguard) which won't work and is much harder to fix on GrapheneOS.

* To make Revolut work with ksu manager, use your root file manager to edit ```/data/system/ApplicationFilterUser.txt``` and add the following line, reboot to apply changes
```
com.revolut.revolut me.weishu.kernelsu
```


* You can also block play store from auto updating youtube revanced by adding this line
```
com.android.vending com.google.android.youtube
```

## Credits

* [GrapheneOS](https://grapheneos.org)

* [KernelSU](https://kernelsu.org)

* [Hide my applist](https://github.com/Dr-TSNG/Hide-My-Applist)

* [DisableFlagSecure](https://github.com/LSPosed/DisableFlagSecure)
