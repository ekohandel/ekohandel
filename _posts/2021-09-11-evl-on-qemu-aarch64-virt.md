---
layout: post
title: "EVL on a QEMU AArch64 virt machine"
toc: true
author:
- Abe Kohandel
comments: true
---

## What is EVL

EVL is the latest version of the Xenomai project also called Xenomai 4. Linux is a General Purpose Operating System (GPOS) and as a result makes no guarantees about meeting real-time requirements. The EVL project attempts to convert the Linux kernel to a _dual kernel_ via a technique called interrupt pipelining which allows a real-time core (EVL Core) to reside alongside the Linux kernel and guarantee real-time performance. The EVL project does this while making the user-space application still remain a normal thread for ease of us. Of course the user-space application must limit its interactions to the EVL Core (via `libevl`) while expecting to maintain real-time guarantees.

The [EVL project overview](https://evlproject.org/overview) provides a great introduction to the project.

## Building EVL

### Background

There are two components to the EVL project:
1. a modified Linux kernel that has the Dovetail interface as well as the EVL Core
2. a library called `libevl` that allows user-space interactions with the EVL Core

The EVL project instructions for [building EVL](https://evlproject.org/core/build-steps/) suggest downloading the EVL patched kernel and building it. This is not a very simple task and can be further complicated if we want to cross-compile the kernel and boot into a useful user-space. There are a few challenges:
1. build a cross-compilation toolchain which is a complicated task
2. optionally create a device tree binary depending on the system we are compiling for
3. construct a root filesystem (rootfs) so we have a useful system

Thankfully there are many projects such as [crosstool-ng](https://crosstool-ng.github.io), [yocto](https://www.yoctoproject.org), and [Buildroot](https://buildroot.org/) that can help us with this. For the purposes of this post I will be using `Buildroot`.

### The Build

#### Download
Let's acquire the latest version of `Buildroot` which is `2022.08` at the time of writing:
```
ekohandel@ekohandel-server:~/sandbox$ mkdir evl
ekohandel@ekohandel-server:~/sandbox$ cd evl
ekohandel@ekohandel-server:~/sandbox/evl$ git clone -b 2022.08 https://git.buildroot.net/buildroot
Cloning into 'buildroot'...
remote: Enumerating objects: 34346, done.
remote: Counting objects: 100% (34346/34346), done.
remote: Compressing objects: 100% (16518/16518), done.
remote: Total 478912 (delta 20181), reused 30438 (delta 17725), pack-reused 444566
Receiving objects: 100% (478912/478912), 104.74 MiB | 31.31 MiB/s, done.
Resolving deltas: 100% (332909/332909), done.
```

We will also need the latest version of `libevl` which is `r38` at the time of writing:
```
ekohandel@ekohandel-server:~/sandbox/evl$ git clone -b r38 https://git.xenomai.org/xenomai4/libevl.git
Cloning into 'libevl'...
warning: redirecting to https://source.denx.de/Xenomai/xenomai4/libevl.git/
remote: Enumerating objects: 3041, done.
remote: Counting objects: 100% (23/23), done.
remote: Compressing objects: 100% (23/23), done.
remote: Total 3041 (delta 6), reused 0 (delta 0), pack-reused 3018
Receiving objects: 100% (3041/3041), 550.41 KiB | 1.47 MiB/s, done.
Resolving deltas: 100% (2126/2126), done.git clone -b r38 https://git.xenomai.org/xenomai4/libevl.git
```

#### Configuring Buildroot
Luckily `Buildroot` comes with a predefined configuration for the QEMU AArch64 virt machine we are interested in so we can use that as a basis for our configuration. You can see all the predefined `Buildroot` configurations in the `buildroot/configs` directory.

In order to create a working configuration based on the default configuration run:
```
ekohandel@ekohandel-server:~/sandbox/evl/buildroot$ make BR2_DEFCONFIG=$(pwd)/configs/qemu_aarch64_virt_defconfig defconfig
```
or the shorthand equivalent:
```
ekohandel@ekohandel-server:~/sandbox/evl/buildroot$ make qemu_aarch64_virt_defconfig
```

You should see a message like below appear at the end of the output indicating that a `.config` file is written to the `Buildroot` base directory:
```
#
# configuration written to /home/ekohandel/sandbox/evl/buildroot/.config
#
```

We have created a configuration for `Buildroot` to build _a kernel_ for the QEMU AArch64 virt machine. It is whatever default kernel `Buildroot` decided to use and not our EVL patched kernel that we want. So we need to modify the `Buildroot` configuration to use the EVL patched kernel.

We need to make some other modifications to the `Buildroot` configuration as well, so we will do these all at the same time:
* point to `https://git.xenomai.org/xenomai4/linux-evl.git` version `v5.15.y-evl-rebase` as the kernel
* use `glibc` instead of the default `uClibc-ng` as the C library
  * `libevl` requires [`<sys/auxv.h>`](https://www.gnu.org/software/gnulib/manual/gnulib.html#Glibc-sys_002fauxv_002eh) for compilation and `uClibc-ng` does not provide support for this
* enable C++ support for the toolchain
  * `libevl` needs a C++ compiler so we need to enable C++ support for our toolchain
* include the `libevl` artifacts into the root filesystem
  * `Buildroot` allows for including root filesystem overlay directories which are copied into the final root filesystem created

{% include asciinema.html asciicast-id="4RE8ztwtRNodPIsIzfOZULmln" idleTimeLimit=2 %}

#### Configuring Linux
We have configured `Buildroot` appropriately now but we are still using a vanilla Linux kernel configuration. We need to modify that configuration to enable the EVL Core.

Just like the `Buildroot` configuration we need to do a few changes to the Linux kernel configuration so we will do them all at the same time:
* Enable EVL real-time core
  * Note that this automatically enabled the Dovetail feature as well
* Enable EVL quota-base scheduling and temporal partitioning policy
  * Not necessary to do this but some EVL checks will fail if we don't
* Enable EVL debug support
* Enable `.config` support and access to the configuration via `/proc/config.gz`
  * EVL can check the kernel configuration for us to ensure no known issues are detected

The following is recorded after running `menu linux-nconfig` for the first time. `Buildroot` does a lot of work the first time you run this command and I have skipped over that as it is not very valuable for the demonstration purposes.

What is important to mention is that `Buildroot` builds the toolchain prior to launching the Linux kernel configuration menu. If you go back after this step and make any changes to the toolchain configuration in `Buildroot` you will have to do a [full rebuild](https://buildroot.org/downloads/manual/manual.html#full-rebuild). Note the this will blow away your Linux kernel configuration and you will have to repeat this step.

{% include asciinema.html asciicast-id="nNIKpJFqMfLEebtBJR2dbmAZt" idleTimeLimit=2 %}

#### Building and Staging `libevl`
Before we build the kernel and create the root filesystem, we need to build the `libevl` library so it can be included in the root filesystem:

Let's create a build directory to work in:
```
ekohandel@ekohandel-server:~/sandbox/evl/libevl$ mkdir build
ekohandel@ekohandel-server:~/sandbox/evl/libevl$ cd build
```

`libevl` uses `meson` as a build system and to cross-compile the library for our target, we need to provide it a `cross-file`. `Buildroot` has already built the cross-compile toolchain for us and placed it inside of the `buildroot/output/host/bin` directory. There are sample `cross-file`s in the `libevl/meson` directory and luckily the `aarch64-none-linux-gnu` file is almost exactly what we need. We need to modify the `vendor` part of the target triples from `none` to `buildroot` which can be done by `sed`:
```
ekohandel@ekohandel-server:~/sandbox/evl/libevl/build$ sed 's/-none-/-buildroot-/g' ../meson/aarch64-none-linux-gnu > aarch64-buildroot-linux-gnu
```

Now we are ready to cross-compile `libevl` and install it to our staging `sysroot` directory. This is the same directory we already told `Buildroot` to find a filesystem overlay in. This means when `Buildroot` creates the root filesystem it will look here and include these files in the root filesystem it creates. We need to also make it so `meson` can find our cross-compile toolchain and we can do that by augmenting the `PATH` environment variable when we call `meson`.

Setup the project:
```
ekohandel@ekohandel-server:~/sandbox/evl/libevl/build$ PATH="$(pwd)/../../buildroot/output/host/bin:$PATH" meson setup --cross-file ./aarch64-buildroot-linux-gnu -Dbuildtype=debug -Dprefix=/usr -Duapi=$(pwd)/../../buildroot/dl/linux/git . ..
The Meson build system
Version: 0.61.2
-------------------------------------->8--------------------------------------
Build targets in project: 100

libevl 0.38

  User defined options
    Cross files: ./aarch64-buildroot-linux-gnu
    buildtype  : debug
    prefix     : /usr
    uapi       : /home/ekohandel/sandbox/evl/libevl/build/../../buildroot/dl/linux/git

Found ninja-1.10.1 at /usr/bin/ninja
Cleaning... 0 files.
```
Compile `libevl`:
```
ekohandel@ekohandel-server:~/sandbox/evl/libevl/build$ PATH="$(pwd)/../../buildroot/output/host/bin:$PATH" meson compile
[227/227] Linking target tidbits/oob-net-icmp
```
Install `libevl`:
```
ekohandel@ekohandel-server:~/sandbox/evl/libevl/build$ DESTDIR=$(pwd)/sysroot meson install
ninja: Entering directory `/home/ekohandel/sandbox/evl/libevl/build'
[2/19] Generating eshi/git_stamp.h with a custom command
Installing lib/libevl.so.1.1.0 to /home/ekohandel/sandbox/evl/libevl/build/sysroot/usr/lib
Installing symlink pointing to libevl.so.1.1.0 to /home/ekohandel/sandbox/evl/libevl/build/sysroot/usr/lib/libevl.so.1
-------------------------------------->8--------------------------------------
Installing /home/ekohandel/sandbox/evl/libevl/utils/trace.timer to /home/ekohandel/sandbox/evl/libevl/build/sysroot/usr/libexec/evl
Installing /home/ekohandel/sandbox/evl/libevl/utils/kconf-checklist.evl to /home/ekohandel/sandbox/evl/libevl/build/sysroot/usr/libexec/evl
Running custom install script '/bin/sh /home/ekohandel/sandbox/evl/libevl/meson/post-install.sh'
Running custom install script '/bin/sh /home/ekohandel/sandbox/evl/libevl/meson/copy-uapi.sh /home/ekohandel/sandbox/evl/libevl/build/include/uapi /usr/include'
```

#### Building the final image

We are finally ready to build the Linux kernel and create the root filesystem to boot into:
```
ekohandel@ekohandel-server:~/sandbox/evl/buildroot$ make
-------------------------------------->8--------------------------------------
Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Copying files into the device: done
Writing superblocks and filesystem accounting information: done

ln -sf rootfs.ext2 /home/ekohandel/sandbox/evl/buildroot/output/images/rootfs.ext4
ln -snf /home/ekohandel/sandbox/evl/buildroot/output/host/aarch64-buildroot-linux-gnu/sysroot /home/ekohandel/sandbox/evl/buildroot/output/staging
>>>   Executing post-image script board/qemu/post-image.sh
```

## Booting EVL

If you have gotten this far, it means we have built a Linux kernel and a root filesystem. You can find both of them here:
```
ekohandel@ekohandel-server:~/sandbox/evl/buildroot/output/images$ ls -l
total 23744
-rw-r--r-- 1 ekohandel ekohandel 11037184 Sep 12 00:10 Image
-rw-r--r-- 1 ekohandel ekohandel 62914560 Sep 12 00:10 rootfs.ext2
lrwxrwxrwx 1 ekohandel ekohandel       11 Sep 12 00:10 rootfs.ext4 -> rootfs.ext2
-rwxr-xr-x 1 ekohandel ekohandel      515 Sep 12 00:10 start-qemu.sh
```

There is also a `start-qemu.sh` helper script generated that simplifies launching QEMU with the built Linux kernel and root filesystem. We do not want a GUI so we call the script with the `serial-only` option:

```
ekohandel@ekohandel-server:~/sandbox/evl/buildroot/output/images$ ./start-qemu.sh serial-only
Booting Linux on physical CPU 0x0000000000 [0x410fd034]
Linux version 5.15.64 (ekohandel@ekohandel-server) (aarch64-buildroot-linux-gnu-gcc.br_real (Buildroot 2022.08) 11.3.0, GNU ld (GNU Binutils) 2.37) #1 SMP IRQPIPE Mon Sep 12 00:07:46 UTC 2022
Machine model: linux,dummy-virt
-------------------------------------->8--------------------------------------
EVL: core started [DEBUG]
-------------------------------------->8--------------------------------------

Welcome to Buildroot
buildroot login: root
#
```

You can see in the kernel boot log above that the EVL core has started: `EVL: core started [DEBUG]`.

## Testing EVL Linux

We can run the EVL tests to ensure everything is configured correctly:

```
# evl test
basic-xbuf: OK
clock-timer-periodic: OK
clone-fork-exec: OK
detach-self: OK
duplicate-element: OK
element-visibility: OK
fault: OK
fpu-preload: OK
fpu-stress: OK
heap-torture: OK
mapfd: OK
monitor-deadlock: OK
monitor-event: OK
monitor-flags: OK
monitor-pi: OK
monitor-pi-deadlock: OK
monitor-pp-dynamic: OK
monitor-pp-lower: OK
monitor-pp-nested: OK
monitor-pp-pi: OK
monitor-pp-raise: OK
monitor-pp-tryenter: OK
monitor-pp-weak: OK
monitor-steal: OK
monitor-trylock: OK
monitor-wait-multiple: OK
observable-hm: OK
observable-inband: OK
observable-onchange: OK
observable-oob: OK
observable-race: OK
observable-thread: OK
observable-unicast: OK
poll-close: OK
poll-flags: OK
poll-many: OK
poll-multiple: OK
poll-nested: OK
poll-observable-inband: OK
poll-observable-oob: OK
poll-sem: OK
poll-xbuf: OK
proxy-echo: OK
proxy-eventfd: OK
proxy-pipe: OK
proxy-poll: OK
ring-spray: OK
** sched-quota-accuracy: BROKEN
```
We do have one failure but that is ok, afterall we are running on a virtual system, we can worry about this when we try to run this on a real system.

We can also run the EVL check utility which verifies the Linux kernel is indeed configured correctly:
```
# evl check
# echo $?
0
```
The utility is awfully quiet so we can make sure it did indeed exit successfully

## Next Steps
Build EVL for a Beagle Bone Black and do some latency measurements!
