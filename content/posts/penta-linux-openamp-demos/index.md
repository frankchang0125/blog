---
date: "2025-01-17T00:00:00+08:00"
title: "AMD PetaLinux + OpenAMP Demos"
author: "Frank Chang"
categories:
  - "AMP"
tags:
  - "OpenAMP"
---

> ⚠️ This demo is based on **PetaLinux 2024.1 release**.

- [PetaLinux Tools Documentation: Reference Guide (UG1144) (2024.1 English)](https://docs.amd.com/r/2024.1-English/ug1144-petalinux-tools-reference-guide/Overview)

---

- Download **2024.1 PetaLinux Tools Installer** & **ZCU102 BSP:** [https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/embedded-design-tools/2024-1.html](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/embedded-design-tools/2024-1.html)
- Switch to bash:

    ```sh
    $ bash
    ```

- Set `$SHELL` to `/bin/bash`:

    ```sh
    $ export SHELL=/bin/bash
    ```

- Disable dash as the default system shell (/bin/sh):

    ```sh
    $ sudo dpkg-reconfigure dash
    ```

    - Select: **<No>**
- Install PetaLinux Tools Installer:

    ```sh
    $ chmod u+x petalinux-v2024.1-05202009-installer.run
    $ ./petalinux-v2024.1-05202009-installer.run -d install
    ```

    - Installer will be installed to: `./install`
- Source PetaLinux environments:

    ```sh
    $ cd ./install
    $ source settings.sh
    ```

- Create project from ZCU102 BSP:

    ```sh
    $ petalinux-create project -s xilinx-zcu102-v2024.1-05230256.bsp
    $ cd ./xilinx-zcu102-2024.1
    ```

    - The project will be created to: `./xilinx-zcu102-2024.1`
- Build OpenAMP dtb:

    ```sh
    $ petalinux-config
    ```

    - Select configs:
        - DTG Settings → Enable openamp dtsi
        - Yocto Settings → Parallel thread execution → Set number of bb threads (BB_NUMBER_THREADS)
            - E.g. `8`
    - PetaLinux project config is located at: `./project-spec/configs/config`
- Enable OpenAMP and its examples in rootfs:

    ```sh
    $ petalinux-config -c rootfs
    ```

    - Select the configs:
        - Filesystem Packages → misc → openamp-fw-echo-testd
        - Filesystem Packages → misc → openamp-fw-mat-muld
        - Filesystem Packages → misc → openamp-fw-rpc-demo
        - Petalinux Package Groups → packagegroup-petalinux-openamp → packagegroup-petalinux-openamp
- Enable Linux Kernel debug info:

    ```sh
    $ petalinux-config -c kernel
    ```

    - Select the configs:
        - Kernel hacking → Kernel debugging
        - Kernel hacking → Compile-time checks and compiler options → Debug information → Rely on the toolchain’s implicit default DWARF version
        - General setup →Compiler optimization level → Optimize for size (-Os)
        - Device Drivers → Character devices → /dev/mem virtual device support
- {{< anchor id="build-zynqmp-r5-remoteproc-driver-into-kernel" >}}If we want to build `zynqmp_r5_remoteproc` driver into kernel, instead of kernel module:
    - Select the configs:
        - Device Drivers → Remoteproc drivers → Support for Remote Processor subsystem → ZynqMP R5 remoteproc support, choose: `y`
- Build the project:

    ```sh
    $ nice -19 petalinux-build
    ```

- Boot QEMU with Linux kernel and rootfs we just built:

    ```sh
    $ petalinux-boot qemu --kernel
    [INFO] Set QEMU tftp to "/tftpboot"
    [INFO] qemu-system-microblazeel -M microblaze-fdt  -serial mon:stdio -serial /dev/null -display none  -kernel /home/frank.chang/Projects/sifive/petalinux/2024.1/xilinx-zcu102-2024.1/images/linux/pmu_rom_qemu_sha3.elf -device loader,file=/home/frank.chang/Projects/sifive/petalinux/2024.1/xilinx-zcu102-2024.1/images/linux/pmufw.elf -hw-dtb /home/frank.chang/Projects/sifive/petalinux/2024.1/xilinx-zcu102-2024.1/images/linux/zynqmp-qemu-multiarch-pmu.dtb -machine-path /tmp/tmp29lts2le -device loader,addr=0xfd1a0074,data=0x1011003,data-len=4 -device loader,addr=0xfd1a007C,data=0x1010f03,data-len=4 &
    Image resized.
    [INFO] qemu-system-aarch64 -M arm-generic-fdt  -serial mon:stdio -serial /dev/null -display none  -device loader,file=/home/frank.chang/Projects/sifive/petalinux/2024.1/xilinx-zcu102-2024.1/images/linux/system.dtb,addr=0x100000,force-raw=on  -device loader,file=/home/frank.chang/Projects/sifive/petalinux/2024.1/xilinx-zcu102-2024.1/images/linux/u-boot.elf -device loader,file=/home/frank.chang/Projects/sifive/petalinux/2024.1/xilinx-zcu102-2024.1/images/linux/Image,addr=0x200000,force-raw=on  -device loader,file=/home/frank.chang/Projects/sifive/petalinux/2024.1/xilinx-zcu102-2024.1/images/linux/ramdisk.cpio.gz.u-boot,addr=0x4000000,force-raw=on  -device loader,file=/home/frank.chang/Projects/sifive/petalinux/2024.1/xilinx-zcu102-2024.1/images/linux/bl31.elf,cpu-num=0 -drive if=sd,format=raw,index=1,file=/home/frank.chang/Projects/sifive/petalinux/2024.1/xilinx-zcu102-2024.1/images/linux/rootfs.ext4 -global xlnx,zynqmp-boot.cpu-num=0 -global xlnx,zynqmp-boot.use-pmufw=true  -global xlnx,zynqmp-boot.drive=pmu-cfg -blockdev node-name=pmu-cfg,filename=/home/frank.chang/Projects/sifive/petalinux/2024.1/xilinx-zcu102-2024.1/images/linux/pmu-conf.bin,driver=file -hw-dtb /home/frank.chang/Projects/sifive/petalinux/2024.1/xilinx-zcu102-2024.1/images/linux/zynqmp-qemu-multiarch-arm.dtb -device loader,file=/home/frank.chang/Projects/sifive/petalinux/2024.1/xilinx-zcu102-2024.1/images/linux/boot.scr,addr=0x20000000,force-raw=on  -gdb tcp:localhost:9000  -net nic -net nic -net nic -net nic,netdev=eth3 -netdev user,id=eth3,tftp=/tftpboot -machine-path /tmp/tmp29lts2le  -m 4G
    qemu-system-microblazeel: Failed to connect to '/tmp/tmp29lts2le/qemu-rport-_pmu@0': No such file or directory
    qemu-system-microblazeel: info: QEMU waiting for connection on: disconnected:unix:/tmp/tmp29lts2le/qemu-rport-_pmu@0,server=on
    qemu-system-aarch64: warning: hub 0 is not connected to host network
    ```

    - When `--kernel` is specified, PetaLinux will try to load all the images from `<plnx-proj-root>/images/linux/`:
        - For Zynq UltraScale+ MPSoC:
            - TF-A image: `<plnx-proj-root>/images/linux/bl31.elf`
            - U-Boot: `<plnx-proj-root>/images/linux/u-boot.elf`
            - Linux kernel image: `<plnx-proj-root>/images/linux/Image`
            - Initrd/Initramfs image: `<plnx-proj-root>/images/linux/ramdisk.cpio.gz.u-boot`
            - Rootfs (as SD card image): `<plnx-proj-root>/images/linux/rootfs.ext4`
            - DTB: `<plnx-proj-root>/images/linux/system.dtb`
            - HW DTB: `<plnx-proj-root>/images/linux/zynqmp-qemu-multiarch-arm.dtb`
            - … etc
            - P.S. The TF-A boots the loaded kernel image, with PMU firmware running in the background.
- After booting to Linux shell:
    - Default login:
        - Username: ***petalinux***
        - Password: ***<Set a password during the first login>***
    - If we didn’t [build `zynqmp_r5_remoteproc` driver into kernel](#build-zynqmp-r5-remoteproc-driver-into-kernel), we need to make sure `zynqmp_r5_remoteproc` kernel module is installed:

        ```sh
        $ lsmod
        zynqmp_r5_remoteproc    16384  0
        uio_pdrv_genirq        12288  0
        cfg80211              319488  0
        ```

        - If not, `modprobe` to install it:

            ```sh
            $ modprobe zynqmp_r5_remoteproc
            ```

    - `rpmsg_ctrl`, `rpmsg_char` kernel modules will be installed automatically when starting remoteproc.
    - Change remoteproc file permissions:

        ```sh
        $ sudo chmod 666 /sys/class/remoteproc/remoteproc0/firmware
        $ sudo chmod 666 /sys/class/remoteproc/remoteproc0/state
        ```

    - Follow the tutorials to run OpenAMP demos:
        - **echo_test**:
            - [https://openamp.readthedocs.io/en/latest/openamp-system-reference/examples/linux/rpmsg-echo-test/README.html](https://openamp.readthedocs.io/en/latest/openamp-system-reference/examples/linux/rpmsg-echo-test/README.html)
        - **matrix multiply**:
            - [https://openamp.readthedocs.io/en/latest/openamp-system-reference/examples/linux/rpmsg-mat-mul/README.html](https://openamp.readthedocs.io/en/latest/openamp-system-reference/examples/linux/rpmsg-mat-mul/README.html)
        - **proxy_app**:
            - [https://openamp.readthedocs.io/en/latest/openamp-system-reference/examples/linux/rpmsg-proxy-app/README.html](https://openamp.readthedocs.io/en/latest/openamp-system-reference/examples/linux/rpmsg-proxy-app/README.html)
        - Firmware is located in `/lib/firmware`:

            ```sh
            $ ls /lib/firmware/
            image_echo_test   image_matrix_multiply  image_rpc_demo
            ```

        - P.S. We need to use `sudo` to run the test application, e.g.

            ```sh
            $ sudo echo_test
            ```

- Remote debug Linux kernel:

    ```sh
    $ petalinux-util gdb linux/image/vmlinux
    (gdb) target remote :9000
    ```


---

- QEMU source: [https://github.com/Xilinx/qemu/tree/xlnx_rel_v2024.1](https://github.com/Xilinx/qemu/tree/xlnx_rel_v2024.1)
- Linux kernel source: [https://github.com/Xilinx/linux-xlnx/tree/xlnx_rebase_v6.6_LTS](https://github.com/Xilinx/linux-xlnx/tree/xlnx_rebase_v6.6_LTS)
    - zynqmp_r5_remoteproc: `drivers/remoteproc/zynqmp_r5_remoteproc.c`
    - rpmsg_ctrl: `drivers/rpmsg/rpmsg_ctrl.c`
    - rpmsg_char: `drivers/rpmsg/rpmsg_char.c`
