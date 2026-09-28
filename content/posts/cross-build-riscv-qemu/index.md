---
date: "2020-01-26T00:00:00+08:00"
title: "Cross-build RISC-V QEMU"
author: "Frank Chang"
categories:
  - "Emulator"
tags:
  - "QEMU"
---

## 在 x86-64 上 cross-build RISC-V QEMU

*OS 為 Ubuntu 22.04*

1. dpkg 加入 RISC-V 架構：
    - `sudo dpkg --add-architecture riscv64`
2. 新增 RISC-V package sources：
    - `sudo vim /etc/apt/sources.list`
    - 新增以下 RISC-V sources：

        ```text
        deb [arch=riscv64] http://ports.ubuntu.com jammy main restricted universe multiverse
        deb [arch=riscv64] http://ports.ubuntu.com jammy-updates main restricted universe multiverse
        deb [arch=riscv64] http://ports.ubuntu.com jammy-security main restricted universe multiverse
        ```

    - 並限制以下的 sources 為 amd64 (加上：`[arch=amd64]`)：

        ```text
        deb [arch=amd64] http://security.ubuntu.com/ubuntu jammy-security main restricted
        deb [arch=amd64] http://security.ubuntu.com/ubuntu jammy-security universe
        deb [arch=amd64] http://security.ubuntu.com/ubuntu jammy-security multiverse
        ```

3. 更新套件清單：
    - `sudo apt update`
4. 安裝開發套件：
    - `sudo apt install crossbuild-essential-riscv64`
    - `sudo apt install zlib1g-dev:riscv64 libglib2.0-dev:riscv64 libpixman-1-dev:riscv64 libpcre2-dev:riscv64 libffi-dev:riscv64 libselinux1-dev:riscv64`
5. 由於 Ubuntu 沒提供 `libmount.a`，需手動編譯 `util-linux` 並處理 `crc32c` 符號衝突：
    1. 下載 `util-linux` source codes：
        - `git clone git@github.com:util-linux/util-linux.git`
    2. `cd util-linux`
    3. `./autogen.sh`
    4. `./configure --host=riscv64-linux-gnu --enable-static --disable-shared --disable-all-programs --enable-libmount --enable-libblkid CFLAGS="-Dcrc32c=util_linux_crc32c"`
    5. `make -j`
6. 複製 `libmount.a` 及 `libblkid.a` 至系統交叉編譯路徑：
    - `sudo cp .libs/libmount.a .libs/libblkid.a /usr/lib/riscv64-linux-gnu/`
7. 設定環境變數：
    - `export PKG_CONFIG=pkg-config`
    - `export PKG_CONFIG_LIBDIR=/usr/lib/riscv64-linux-gnu/pkgconfig:/usr/share/pkgconfig`
    - `export PKG_CONFIG_SYSROOT_DIR=/`
8. 切到 QEMU 的資料夾並 configure QEMU (with debug info and SLiRP)：
    - `./configure --cross-prefix=riscv64-linux-gnu- --cpu=riscv64 --target-list=riscv64-softmmu --static --enable-debug --enable-slirp -Ddefault_library=static`
        - `default_library` 是 meson built-in 的參數用來控制當使用 `library()` 的時候要怎麼編譯：
            - `shared`：將 library 編譯成 shared library (e.g., `.so`)
            - `static`：將 library 編譯成 static library (e.g., `.a`)
            - `both`：同時將 library 編譯成 shared 及 static libraries
        - `-Ddefault_library=static` 可以指定將 subprojects 編譯成 static libraries (e.g., `libslirp.a`)
9. 編譯 QEMU：
    - `make -j`
10. 最後 QEMU 的執行檔會在：`./build/qemu-system-riscv64`

    ```sh
    $ file build/qemu-system-riscv64
    build/qemu-system-riscv64: ELF 64-bit LSB executable, UCB RISC-V, RVC, double-float ABI, version 1 (SYSV), statically linked, BuildID[sha1]=e5fb33b2ec9617aaae8890df5979b12e245f4f00, for GNU/Linux 4.15.0, with debug_info, not stripped
    ```
