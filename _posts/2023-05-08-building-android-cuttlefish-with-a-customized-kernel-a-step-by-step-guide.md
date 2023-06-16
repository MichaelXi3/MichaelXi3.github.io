---
layout: post
title: "Building Android Cuttlefish with a Customized Kernel: A Step by Step Guide"
subtitle: "Android Cuttlefish Virtual Machine"
date: 2023-05-08
author: "Michael Xi"
header-img: "img/Android.jpeg"
tags: [Android, System, Android Cuttlefish]
---

# Cuttlefish Introduction

### What is Cuttlefish?

Android Cuttlefish is an open-source Android Virtual Device (AVD) developed by Google. Its purpose is to run Android as a virtual machine on Linux-based systems for testing and development purposes. It leverages QEMU, a popular open-source emulator, and KVM, a hardware-assisted virtualization technology, to enable efficient virtualization and emulation of Android devices. This allows developers, researchers, and testers to work with AOSP without having access to a physical device.

![22.png](https://s2.loli.net/2023/05/09/m4tcKaVC5UbXBWd.png)

**Github Link**: [https://github.com/google/android-cuttlefish](https://github.com/google/android-cuttlefish)

**Cuttlefish Introduction Slide**: [https://2net.co.uk/slides/aosp-aaos-meetup/2022-march-cuttlefish.pdf](https://2net.co.uk/slides/aosp-aaos-meetup/2022-march-cuttlefish.pdf)

# Cuttlefish Setup Guide

### Step_1: Make sure virtualization with KVM is available

```bash
grep -c -w "vmx\|svm" /proc/cpuinfo
```

This should return a non-zero value. If running on a cloud machine, this may take cloud-vendor-specific steps to enable.

### Step_2: Download, build, and install the host debian packages

```bash
sudo apt install -y git devscripts config-package-dev debhelper-compat golang curl
git clone https://github.com/google/android-cuttlefish
cd android-cuttlefish
for dir in base frontend; do
  cd $dir
  debuild -i -us -uc -b -d
  cd ..
done
sudo dpkg -i ./cuttlefish-base_*_*64.deb || sudo apt-get install -f
sudo dpkg -i ./cuttlefish-user_*_*64.deb || sudo apt-get install -f
sudo usermod -aG kvm,cvdnetwork,render $USER
sudo reboot
```

This script installs the Android Cuttlefish environment on a Linux-based system.

### Step_3: Download OTA images and host package for cuttlefish

✍🏻 **OTA** (Over-The-Air) image contains the Android system image and the necessary files to run Cuttlefish on a virtual machine.

✍🏻 **Host package** includes the necessary tools, libraries, and configuration files for running the Cuttlefish virtual machine on your host system.

1. Go to [http://ci.android.com/](http://ci.android.com/)
2. Enter a branch name. Start with `aosp-master` if you don‘t know what you’re looking for
3. Navigate to `aosp_cf_x86_64_phone` and click on `userdebug` for the latest build
4. Click on `Artifacts`
5. Scroll down to the **OTA** images. These packages look like `aosp_cf_x86_64_phone-img-xxxxxx.zip` -- it will always have `img` in the name. Download this file
6. Scroll down to `cvd-host_package.tar.gz`. You should always download a **host package** from the same build as your images
7. On your local system, combine the packages with following code
    
    ```bash
    mkdir cf
    cd cf
    tar xvf /path/to/cvd-host_package.tar.gz
    unzip /path/to/aosp_cf_x86_64_phone-img-xxxxxx.zip
    ```
    

### Step_4: Launch cuttlefish and other useful cuttlefish commands

1. Launch cuttlefish virtual machine
    
    ```bash
    $ HOME=$PWD ./bin/launch_cvd
    ```
    
2. Enable cuttlefish root
    
    ```bash
    ./bin/adb root
    ```
    
3. Launch cuttlefish shell
    
    ```bash
    ./bin/adb shell
    ```
    
4. Stop cuttlefish virtual machine
    
    ```bash
    HOME=$PWD ./bin/stop_cvd
    ```
    

# Build Customized Android Kernel and Swap into Cuttlefish

### Build Customized Android Kernel

1. Install `Repo` Command for downloading Android Kernel source code
    
    ```bash
    # Debian/Ubuntu.
    $ sudo apt-get install repo
    
    # Gentoo.
    $ sudo emerge dev-vcs/repo
    ```
    
    Or install manually by the following code:
    
    ```bash
    $ mkdir -p ~/.bin
    $ PATH="${HOME}/.bin:${PATH}"
    $ curl https://storage.googleapis.com/git-repo-downloads/repo > ~/.bin/repo
    $ chmod a+rx ~/.bin/repo
    ```
    
2. Download Android Kernel source code
    
    ```bash
    # Branch common-android11-5.4 as an example
    repo init -u https://android.googlesource.com/kernel/manifest -b common-android11-5.4
    repo sync -j12
    ```
    
    The option `j12` means that the sync operation will use 12 parallel threads or jobs. If an error occurs during the sync process, you can try reducing the number of threads by lowering the number that follows the `j` option first.
    
3. Apply the kernel patches to Android Kernel source code for customization
    
    > In this example, we are applying a Linux system provenance kernel patch to the kernel.


    1. **Step_1**: Download Camflow Kernel Patch from Camflow Repo
        - Camflow is a major LSM for Linux kernel Provenance
        - The link of Camflow patch releases is: [https://github.com/CamFlow/camflow-dev/releases](https://github.com/CamFlow/camflow-dev/releases)
        - Download one of the release of Camflow from the link above, in this example, we use Camflow v0.8.0 and Android Kernel branch Android13-5.15-LTS
    2. **Step_2**: Install Camflow Kernel Patch to Kernel source code before building it
        - Apply the two Camflow patches to the `common` kernel source code directory
            
            ```bash
            cd android-kernel-main-camflow/common
            git apply ../../Downloads/0001-information-flow.patch
            git apply ../../Downloads/0002-camflow.patch
            ```
            
4. Build the Android Kernel
    
    > Since Android 10, Android has introduced a new **[Generic Kernel Image(GKI)](https://source.android.com/devices/architecture/kernel/generic-kernel-image)** in kernel 4.19 and above. This means that the kernel building process has been divided into two parts: `Generic Kernel` and `GKI Module`. We have to build these two parts separately.
    > 
    > 
    > ![33.png](https://s2.loli.net/2023/05/09/4iUwsP7QLefFHTx.png)
    
    - **Build Generic Kernel**
        
        ```bash
        # bazel build
        tools/bazel build //common:kernel_x86_64_dist
        ```

        ```bash
        # create output distribution
        tools/bazel run //common:kernel_x86_64_dist -- --dist_dir=/home/username/android-kernel-5.15/vendor-build-output-x86
        ```
        
    - **Build GKI Modules**
        
        ```bash
        tools/bazel build //common-modules/virtual-device:virtual_device_x86_64_dist
        ```

        ```bash
        # create output distribution
        tools/bazel run //common-modules/virtual-device:virtual_device_x86_64_dist -- --dist_dir=/home/username/android-kernel-5.15/vendor-build-output-x86
        ```
        
        Now, both `initramfs.img` and `bzImage` should be in `/home/username/android-kernel-5.15/vendor-build-output-x86`. You can now try swapping this kernel into your Cuttlefish Android Virtual Device.
        
        ✍🏻 **`initramfs.img`**: The initial RAM filesystem is a temporary in-memory file system that contains essential files and utilities required for mounting the root file system. It is compressed and stored as an image file, such as `initramfs.img`.
                
        ✍🏻 **`bzImage`**: The `bzImage` is a compressed Linux kernel image created during the kernel compilation process. It is designed to be small enough to fit within limited memory space during the boot process. The `bzImage` is loaded by the bootloader, decompressed, and executed to initialize the system hardware and kernel before transitioning to the root file system.
                

### Swap the Cutomized Android Kernel into Cuttlefish

1. Set the `DIST_FOLDER` as the folder that contains `initramfs.img` and `bzImage`
    
    ```bash
    DIST_FOLDER=$(readlink -f /home/username/android-kernel-5.15-success/out/android13-5.15/dist)
    ```
    
2. Navigate to `android-cuttlefish/cf`, and then use the following command to launch Cuttlefish using the kernel we just built.
    
    ```bash
    HOME=${PWD} ./bin/launch_cvd -daemon -initramfs_path "${DIST_FOLDER}"/initramfs.img -kernel_path "${DIST_FOLDER}"/bzImage
    ```
    
3. Launch the shell of cuttlefish and check the kernel version to verify that kernel is swapped successfully
    
    ```bash
    ./bin/adb shell
    ```
    
    ```bash
    vsoc_x86_64:/proc $ cat version
    Linux version 5.15.78-maybe-dirty (build-user@build-host) (Android (8508608, based on r450784e) clang version 14.0.7 (https://android.googlesource.com/toolchain/llvm-project 4c603efb0cca074e9238af8b4106c30add4418f6), LLD 14.0.7)
    ```
    
4. To check the files of Camflow in Android Cuttlefish, we need to mount a few directories
    1. **Step_1**: Exit current Cuttlefish: `HOME=$PWD ./bin/stop_cvd`
    2. **Step_2**: Launch a new Cuttlefish: `HOME=${PWD} ./bin/launch_cvd -daemon -initramfs_path "${DIST_FOLDER}"/initramfs.img -kernel_path "${DIST_FOLDER}"/bzImage`
    3. **Step_3**: Enable root: `./bin/adb root`
    4. **Step_4**: Launch Cuttlefish root shell: `./bin/adb shell`
    5. **Step_5**: mount `securityfs` manually
        
        ```bash
        mount -t securityfs securityfs /sys/kernel/security
        ```
        
    6. **Step_6**: mount `debugfs` manually
        
        ```bash
        mount -t debugfs debugfs /sys/kernel/debug
        ```
        
        Files related to Camflow should exist in both the `debugfs` and `securityfs` directories.
        

# Reference and Useful Info

1. Google Cuttlefish Documentation: [Link](https://android.googlesource.com/device/google/cuttlefish/)
2. Cheatsheat of `launch_cvd`
    
    
    | -start_vnc_server | Start VNC server on port 6444 |
    | --- | --- |
    | -start_webrtc | Start web UI on https://localhost:8443 |
    | -console=true | Start console interface cuttlefish_runtime/console |
    | -daemon | Daemon mode (run as a background process) |
    | -pause-in-bootloader=true | Access bootloader via serial console |
    | -x_res | screen width |
    | -y_res | screen height |
    | -dpi | screen resolution |
    | -guest_enforce_security=false | SELinux in permissive mode |
    | -extra_kernel_cmdline "" | additional Linux command line |
    | -cpus | Number of CPUs to emulate |
    | -memory_mb | amount of memory to give to device |
    | -noresume | Start a new runtime: factory reset |
3. **Steps to Customize Kernel Configs**: The goal is to enable any desired CONFIGs by modifying the kernel building script.
    
    > Useful Blog: [Link](https://blog.senyuuri.info/posts/2021-06-30-ebpf-bcc-android-instrumentation/#2-build-a-custom-generic-kernel-image-gki)
    >
    Documentations relate to LSM in Android: [Link](https://android.googlesource.com/kernel/msm/+/android-5.1.0_r0.6/Documentation/security)
    > 
    
    In `android-kernel-5.15/common`, there is a config file named `build.config.gki`. Write an update config function here and configure the new CONFIG or delete unwanted CONFIG.
    
    ```bash
    DEFCONFIG=gki_defconfig
    POST_DEFCONFIG_CMDS="check_defconfig && **update_lsm_config**"
    
    function update_lsm_config() {
        ${KERNEL_DIR}/scripts/config --file ${OUT_DIR}/.config \
           -d CONFIG_LTO \
           -d CONFIG_LTO_CLANG \
           -e CONFIG_NETFILTER \
           -e CONFIG_SECURITYFS \
           -e CONFIG_SECURITY_NETWORK \
           -e CONFIG_SECURITY_PROVENANCE \
           -e CONFIG_SECURITY_TOMOYO \
           -e CONFIG_SECURITY_APPARMOR \
           -e CONFIG_LSM
        (cd ${OUT_DIR} && \
         make ${CC_LD_ARG} O=${OUT_DIR} olddefconfig)
    }
    ```
    
    - `-d` means delete configuration
    - `-e` means add configuration
    - This function will change `.config` file directly