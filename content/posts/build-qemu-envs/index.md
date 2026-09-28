---
date: "2020-01-25T00:00:00+08:00"
title: "Build QEMU environments"
author: "Frank Chang"
categories:
  - "Emulator"
tags:
  - "QEMU"
---

# Build QEMU

1. 安裝所需的套件：
    - `sudo apt install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex xinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev git libncurses5-dev libpixman-1-dev`

2. 下載 QEMU source codes：
    - `git clone --recursive git@github.com:qemu/qemu.git`
3. `cd qemu`
4. `./configure --target-list=riscv64-softmmu,riscv32-softmmu,riscv64-linux-user,riscv32-linux-user`
5. `make -j`
- 幾個好用的 debug configure 參數：
    - `--extra-cflags=CFLAGS`：append extra C compiler flags QEMU_CFLAGS
        - e.g. `--extra-cflags="-g3 --save-temps"`
    - `--extra-ldflags=LDFLAGS`：append extra linker flags LDFLAGS
    - `--enable-debug-tcg`：enable TCG debugging
    - `--disable-debug-tcg`：disable TCG debugging (default)
    - `--disable-debug-info`：disable debugging information
    - `--enable-debug`：enable common debug build options
    - `--disable-strip`：disable stripping binaries
    - `--disable-werror`：disable compilation abort on warning
    - `--enable-pie`：build Position Independent Executables
    - `--disable-pie`：do not build Position Independent Executables
- 幾個好用的執行參數：
    - `-d item1,...`：enable logging of specified items

        ```
        Log items (comma separated):
        out_asm         show generated host assembly code for each compiled TB
        in_asm          show target assembly code for each compiled TB
        op              show micro ops for each compiled TB
        op_opt          show micro ops after optimization
        op_ind          show micro ops before indirect lowering
        int             show interrupts/exceptions in short format
        exec            show trace before each executed TB (lots of logs)
        cpu             show CPU registers before entering a TB (lots of logs)
        fpu             include FPU registers in the 'cpu' logging
        mmu             log MMU-related activities
        pcall           x86 only: show protected mode far calls/returns/exceptions
        cpu_reset       show CPU state before CPU resets
        unimp           log unimplemented functionality
        guest_errors    log when the guest OS does something invalid (eg accessing a
        non-existent register)
        page            dump pages at beginning of user mode emulation
        nochain         do not chain compiled TBs so that "exec" and "cpu" show
        complete traces
        strace          log every user-mode syscall, its input, and its result
        trace:PATTERN   enable trace events
        ```

    - `-D logfile`：output log to logfile (default stderr)
- References：
    - [RISC-V QEMU Wiki](https://github.com/riscv/riscv-qemu/wiki)
    - [QEMU Build Considerations For Debugging](https://www.cnblogs.com/root-wang/p/8005212.html)

---

# Build RISC-V toolchains

1. 安裝所需的套件：

    ```
    sudo apt-get install autoconf automake autotools-dev curl libmpc-dev \
                         libmpfr-dev libgmp-dev gawk build-essential bison \
                         flex texinfo gperf libtool patchutils bc zlib1g-dev \
                         libexpat-dev
    ```

2. 下載 toolchains source codes：
    - `git clone --recursive [git@github.com](mailto:git@github.com):riscv/riscv-gnu-toolchain.git`
3. `cd riscv-gnu-toolchain`
4. Install：
    - Newlib：
        1. `./configure --prefix=/opt/riscv`
        2. `make`
    - Linux：
        1. `./configure --prefix=/opt/riscv`
        2. `make linux`
5. 將 RISC-V toolchain binary 路徑加入 `PATH` 環境變數：
    - `export PATH=/opt/riscv/bin:$PATH`

---

# Build Buildroot

1. 下載 Buildroot source codes：
    - `git clone [git@github.com](mailto:git@github.com):buildroot/buildroot.git`
2. `make qemu_riscv64_virt_defconfig menuconfig`
    - 如果不想由 bulidroot 編譯 Linux Kernel：
        - 取消勾選：Kernel → Linux Kernel (`BR2_LINUX_KERNEL`)
    - 如果不想由 bulidroot 編譯 OpenSBI：
        - 取消勾選：Bootloaders → opensbi (`BR2_TARGET_OPENSBI`)
    - 如果不想由 bulidroot 編譯 U-Boot：
        - 取消勾選：Bootloaders →U-Boot (`BR2_TARGET_UBOOT`)
    - 如果不想由 buildroot 編譯 QEMU：
        - 取消勾選：Host utilities → host qemu (`BR2_PACKAGE_HOST_QEMU`)
    - 如果想使用自行編譯的 toolchain：
        - Toolchain →Toolchain type → 選擇：External toolchain (`BR2_TOOLCHAIN_EXTERNAL`)
        - Toolchain → Toolchain → 選擇：Custom toolchain (`BR2_TOOLCHAIN_EXTERNAL_CUSTOM`)
        - Toolchain → Toolchain origin → 選擇：Pre-installed toolchain (`BR2_TOOLCHAIN_EXTERNAL_PREINSTALLED`)
        - 設置 toolchain 的路徑：
            - Toolchain → Toolchain path (`BR2_TOOLCHAIN_EXTERNAL_PATH`)：
                - e.g. `/scratch3/yihaoc/sifive/toolchains/riscv64-unknown-linux-gnu-toolsuite-4.2.0-x86_64-linux-redhat8`
        - 設置 toolchain 的 prefix：
            - Toolchain → Toolchain prefix (`BR2_TOOLCHAIN_EXTERNAL_CUSTOM_PREFIX`)：
                - e.g. `$(ARCH)-unknown-linux-gnu`
        - 此外，還需根據所使用的 toolchain 設定其他的選項：
            - 選擇 toolchain 的 gcc 版本：
                - Toolchain → External toolchain gcc version：
                    - e.g. `14.x` (`BR2_TOOLCHAIN_EXTERNAL_GCC_14`)
            - 選擇 toolchain 的 Kernel headers series：
                - Toolchain → External toolchain kernel headers series：
                    - e.g. `6.18.x` (`BR2_TOOLCHAIN_EXTERNAL_HEADERS_6_18`)
            - 選擇 toolchain 的 C library：
                - Toolchain → External toolchain C library：
                    - e.g. `glibc` (`BR2_TOOLCHAIN_EXTERNAL_CUSTOM_GLIBC`)
            - 選擇 toolchain 是否支援 RPC：
                - Toolchain → Toolchain has RPC support? (`BR2_TOOLCHAIN_EXTERNAL_INET_RPC`)
                    - SiFive toolchain → **No**
            - 選擇 toolchain 是否支援 C++：
                - Toolchain → Toolchain has C++ support? (`BR2_TOOLCHAIN_EXTERNAL_CXX`)
                    - SiFive toolchain → **Yes**
            - 選擇 toolchain 是否支援 Fortran：
                - Toolchain → Toolchain has Fortran support? (`BR2_TOOLCHAIN_EXTERNAL_FORTRAN`)
                    - SiFive toolchain → **Yes**
            - 選擇 toolchain 是否支援 OpenMP：
                - Toolchain → Toolchain has OpenMP support?(`BR2_TOOLCHAIN_EXTERNAL_OPENMP`)
                    - SiFive toolchain → **Yes**
    - 如果想使用 initramfs：
        - 勾選：Filesystem images →cpio the root filesystem (for use as an initial RAM filesystem) (`BR2_TARGET_ROOTFS_CPIO`)
    - 編譯 rootfs 以及選擇 rootfs 的格式：
        - Filesystem images → ext2/3/4 root filesystem (`BR2_TARGET_ROOTFS_EXT2`)
        - Filesystem images → ext2/3/4 root filesystem → ext2/3/4 variant：
            - e.g. ext4 (`BR2_TARGET_ROOTFS_EXT2_4`)
3. `make -j`

---

# Build Busybox

1. 下載 Busybox source codes：
    - `git clone [git@github.com](mailto:git@github.com):mirror/busybox.git`
2. 編譯 Busybox：
    - `make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- menuconfig`
    - 勾選：`Settings` → `Build static binary (no shared libs)` `(CONFIG_STATIC)`，以 statically linked 的方式編譯 Busybox
        - 若無勾選，則預設會採用 dynamic linked 的方式編譯 Busybox，在建立 rootfs 的時候需要將所需的 library files 複製至 libs 資料夾，否則 Linux Kernel boot up 的時候會出現： `kernel panic - not syncing: No working init found.` 的錯誤訊息 (此訊息不一定是代表 `init`檔案找不到，無法正確執行 `init` 程式亦會顯示同樣的錯誤訊息)。
    - `make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- -j`
3. 將 Busybox 安裝至 `_install` 資料夾：
    - `make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- install`

---

# Build rootfs from busybox (if rootfs is not built in Linux Kernel image)

1. 建立大小為 10 MB 的 disk image：
    - `dd if=/dev/zero of=root.ext2 bs=1M count=10`
2. 將 disk image 格式化為 `ext2`格式：
    - `mkfs.ext2 -F root.ext2`
3. 掛載 rootfs：
    - `mkdir rootfs`
    - `sudo mount root.ext2 rootfs`
4. `cd rootfs`
5. 複製 Busybox 至 rootfs：
    - `sudo cp -r ../busybox/_install/* .`
        - (假設 Busybox 的路徑為：`../busybox`)
6. 建立所需的資料夾：
    - `sudo mkdir etc dev proc sys tmp`
7. 建立 `etc/inittab` 檔：

    ```
    ::sysinit:/bin/busybox mount -t proc proc /proc
    ::sysinit:/bin/busybox mount -t tmpfs tmpfs /tmp
    ::sysinit:/bin/busybox --install -s
    ::shutdown:/bin/umount -a -r
    ::shutdown:/sbin/swapoff -a
    /dev/console::sysinit:-/bin/ash
    ```

8. 建立所需的裝置檔案：
    - `sudo mknod dev/console c 5 1`
    - `sudo mknod dev/null c 1 3`
9. 解除掛載：
    - `cd .. && sudo umount rootfs`

---

# Bulid OpenSBI

1. 下載 OpenSBI source codes：
    - `git clone [git@github.com](mailto:git@github.com):riscv-software-src/opensbi.git`
2. `make CROSS_COMPILE=riscv64-unknown-linux-gnu- PLATFORM=generic FW_TEXT_START=0x80000000 -j`

---

# Build U-Boot

1. 下載 U-Boot source codes：
    - `git clone [git@github.com](mailto:git@github.com):u-boot/u-boot.git`
2. **TODO**

---

# Build Linux Kernel

1. 下載 Linux Kernel source codes：
    - `git clone [git@github.com](mailto:git@github.com):torvalds/linux.git`
2. `make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- menuconfig`
    - 如果要使用 initramfs，且將 rootfs 直接編譯至核心中，則需：
        - 勾選：General setup → Initial RAM filesystem and RAM disk (initramfs/initrd) support (`CONFIG_BLK_DEV_INITRD`)
        - 並設置 initramfs rootfs 路徑 (`CONFIG_INITRAMFS_SOURCE`)
            - e.g. `/home/yihaoc/frankchang/buildroot/output/images/rootfs.cpio`
    - 如果想包含 debug symbols：
        - 開啟 Kernel debugging：
            - Kernel hacking → Kernel debugging (`DEBUG_KERNEL`)
        - 選擇 DWARF debug info 版本：
            - Kernel hacking → Compile-time checks and compiler options → Debug information
                - e.g. Rely on the toolchain’s implicit default DWARF version (`DEBUG_INFO_DWARF_TOOLCHAIN_DEFAULT`)
    - 如果想要包含 devmem：
        - Device Drivers → Character devices → /dev/mem virtual device support (`CONFIG_DEVMEM`)
    - 切換 Compiler optimization level 為 Optimize for size (`-Os`)
        - 勾選：General setup →Compiler optimization level → Optimize for size (-Os) (`CONFIG_CC_OPTIMIZE_FOR_SIZE`)
3. `make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- -j`

---

# Boot QEMU with Linux Kernel and rootfs

## Method 1: rootfs built-in in Linux Kernel image:

```
./qemu/build/qemu-system-riscv64 \
        -nographic \
        -machine virt \
        -smp 4 \
        -m 2G \
        -bios ./opensbi/build/platform/generic/firmware/fw_jump.elf \
        -kernel ./linux/arch/riscv/boot/Image \
        -append "root=/dev/vda ro console=ttyS0 earlycon" \
        -device virtio-net-device,netdev=usernet \
        -netdev user,id=usernet,hostfwd=tcp::10000-:22
```

- (假設 QEMU 執行檔的路徑為：`./qemu/build/qemu-system-riscv64`)
- (假設 OpenSBI image 的路徑為：`./opensbi/build/platform/generic/firmware/fw_jump.elf`)
- (假設 Linux Kernel image 的路徑為：`./linux/arch/riscv/boot/Image`)

---

## Method 2: Load rootfs from disk:

```
./qemu/build/qemu-system-riscv64 \
        -nographic \
        -machine virt \
        -smp 4 \
        -m 2G \
        -bios ./opensbi/build/platform/generic/firmware/fw_jump.elf \
        -kernel ./linux/arch/riscv/boot/Image \
        -append "root=/dev/vda ro console=ttyS0 earlycon" \
        -device virtio-blk-device,drive=hd0 \
        -drive file=./images/root.ext2,format=raw,id=hd0 \
        -device virtio-net-device,netdev=usernet \
        -netdev user,id=usernet,hostfwd=tcp::10000-:22
```

- (假設 QEMU 執行檔的路徑為：`./qemu/build/qemu-system-riscv64`)
- (假設 OpenSBI image 的路徑為：`./opensbi/build/platform/generic/firmware/fw_jump.elf`)
- (假設 Linux Kernel image 的路徑為：`./linux/arch/riscv/boot/Image`)
- (假設 rootfs disk image 的路徑為：`./images/root.ext2`)
- References：
    - [第二日：環境架設及過程解析 - iT 邦幫忙 - iThome](https://ithelp.ithome.com.tw/articles/10192454)
    - [使用 Busybox 建立 RISC-V 的迷你系統](https://coldnew.github.io/6cc46ece/)

---

# Debug Linux Kernel with GDB with QEMU

- Linux Kernel 需要勾選：
    - `Kernel hacking` → `Kernel debugging` `(CONFIG_DEBUG_KERNEL)`
    - `Kernel hacking` → `Compile-time checks and compiler options` → `Compile the kernel with debug info` `(CONFIG_DEBUG_INFO)`
    - `Kernel hacking` → `Compile-time checks and compiler options` → `Provide GDB scripts for kernel debugging` `(CONFIG_GDB_SCRIPTS)`
- 啟動 QEMU：

    ```
    ./qemu/riscv64-softmmu/qemu-system-riscv64 \
            -nographic \
            -machine virt \
            -smp 4 \
            -m 2G \
            -kernel ./riscv-pk/build/bbl \
            -append "root=/dev/vda ro console=ttyS0 earlycon" \
            -device virtio-blk-device,drive=hd0 \
            -drive file=root.ext2,format=raw,id=hd0 \
            -device virtio-net-device,netdev=usernet \
            -netdev user,id=usernet,hostfwd=tcp::10000-:22 \
            -s -S
    ```

    - `-S`：Freeze CPU at startup (use 'c' to start execution).
    - `-s`：Shorthand for `-gdb tcp::1234`.
    - `-gdb dev`：Wait for gdb connection on `dev`.
- 使用 GDB debug Linux Kernel：

    ```
    riscv64-unknown-linux-gnu-gdb vmlinux
    > (gdb) target remote :1234
    ```

- 使用 CGDB debug Linux Kernel：

    ```
    cgdb -d riscv64-unknown-linux-gnu-gdb vmlinux
    > (gdb) target remote :1234
    ```

- References：
    - [QEMU/Debugging with QEMU](https://en.wikibooks.org/wiki/QEMU/Debugging_with_QEMU)
    - [使用QEMU和GDB調試Linux內核](https://consen.github.io/2018/01/17/debug-linux-kernel-with-qemu-and-gdb/)

---

# Debug QEMU

- 編譯 QEMU：

    ```
    ./configure --help

    Advanced options (experts only):
      --extra-cflags=CFLAGS     append extra C compiler flags QEMU_CFLAGS
      --extra-cxxflags=CXXFLAGS append extra C++ compiler flags QEMU_CXXFLAGS
      --extra-ldflags=LDFLAGS   append extra linker flags LDFLAGS
      --enable-debug            enable common debug build options
      --disable-strip           disable stripping binaries
      --enable-gprof            QEMU profiling with gprof
      --enable-profiler         profiler support

    Optional features, enabled with --enable-FEATURE and
    disabled with --disable-FEATURE, default is enabled if available:
      pie                       Position Independent Executables (--disable-pie)
    ```

    - `./configure --target-list=riscv64-softmmu --enable-debug`
    - `./configure --target-list=riscv64-softmmu --enable-debug --extra-cflags="-g3 --save-temps" --extra-ldflags="-g3" --disable-strip --disable-pie`
- 使用 GDB：

    ```
    gdb --args ./qemu/riscv64-softmmu/qemu-system-riscv64 \
            -nographic \
            -machine virt \
            -smp 4 \
            -m 2G \
            -kernel ./riscv-pk/build/bbl \
            -append "root=/dev/vda ro console=ttyS0" \
            -device virtio-blk-device,drive=hd0 \
            -drive file=root.ext2,format=raw,id=hd0 \
            -device virtio-net-device,netdev=usernet \
            -netdev user,id=usernet,hostfwd=tcp::10000-:22
    ```

- 使用 CGDB：

    ```
    cgdb --args ./qemu/riscv64-softmmu/qemu-system-riscv64 \
            -nographic \
            -machine virt \
            -smp 4 \
            -m 2G \
            -kernel ./riscv-pk/build/bbl \
            -append "root=/dev/vda ro console=ttyS0" \
            -device virtio-blk-device,drive=hd0 \
            -drive file=root.ext2,format=raw,id=hd0 \
            -device virtio-net-device,netdev=usernet \
            -netdev user,id=usernet,hostfwd=tcp::10000-:22
    ```

- References：
    - [Setups For Debugging QEMU with GDB and DDD](https://www.cnblogs.com/root-wang/p/8005212.html)
    - [Tricks for debugging QEMU — rr](https://translatedcode.wordpress.com/2015/05/30/tricks-for-debugging-qemu-rr/)

---

# GDB / CGDB

- [《100個gdb小技巧》](https://github.com/hellogcc/100-gdb-tips)
- [cgdb](https://cgdb.github.io/)
- [CGDB – 更好用的 GDB](https://tech.mozilla.com.tw/?p=3826)

---

# Build bbl (Berkery Boot Loader)

1. 下載 riscv-pk source codes：
    - `git clone [git@github.com](mailto:git@github.com):riscv/riscv-pk.git`
2. `cd riscv-pk`
3. `mkdir build`
4. `cd build`
5. `../configure --enable-logo --with-payload=../../linux/vmlinux --host=riscv64-unknown-linux-gnu`
    - (假設 Linux Kernel vmlinux 的路徑為：`../../linux/vmlinux`)
6. `make -j`

P.S. 如果要使用 BBL 開機的話，QEMU 的 -kernel 參數要指定 BBL image：

e.g. `-kernel ./riscv-pk/build/bbl`

- (假設 bbl 的路徑為： `./riscv-pk/build/bbl`)
- References：
    - [RISC-V Proxy Kernel and Boot Loader Github](https://github.com/riscv/riscv-pk)
