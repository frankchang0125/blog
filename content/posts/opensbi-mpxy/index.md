---
date: "2024-12-23T00:00:00+08:00"
title: "OpenSBI: MPXY"
author: "Frank Chang"
categories:
  - "Security"
tags:
  - "OP-TEE"
  - "OpenSBI"
series:
  - "OP-TEE Code Trace"
---

> ⚠️ The code is based on: [https://gitlab.com/riseproject/riscv-optee/opensbi/-/tree/dev-optee-mpxy](https://gitlab.com/riseproject/riscv-optee/opensbi/-/tree/dev-optee-mpxy)
>
> Commit ID: **`7d4c90953afe3bd86f9e2501bd4c2501e8db1898`**

- `init_coldboot()`
    - [`sbi_mpxy_init()`](/posts/opensbi-mpxy/#sbi-mpxy-init)
- {{< anchor id="sbi-mpxy-init" >}}`sbi_mpxy_init()`
    - Allocate `struct mpxy_state` for each domain.
    - `sbi_platform_mpxy_init()`
        - Call platform-defined `mpxy_init()`.
            - e.g. For generic platform: `fdt_mpxy_init()`.
- `fdt_mpxy_init()`
    - Iterate MPXY drivers in `fdt_mpxy_drivers[]`.
        - MPXY drivers are generated at compile time:

            ```c
            // lib/utils/mpxy/objects.mk

            libsbiutils-objs-$(CONFIG_FDT_MPXY) += mpxy/fdt_mpxy.o
            libsbiutils-objs-$(CONFIG_FDT_MPXY) += mpxy/fdt_mpxy_drivers.o

            carray-fdt_mpxy_drivers-$(CONFIG_FDT_MPXY_RPMI_MBOX) += fdt_mpxy_rpmi_mbox
            libsbiutils-objs-$(CONFIG_FDT_MPXY_RPMI_MBOX) += mpxy/fdt_mpxy_rpmi_mbox.o
            carray-fdt_mpxy_drivers-$(CONFIG_FDT_MPXY_MM) += fdt_mpxy_mm
            libsbiutils-objs-$(CONFIG_FDT_MPXY_MM) += mpxy/fdt_mpxy_mm.o
            carray-fdt_mpxy_drivers-$(CONFIG_FDT_MPXY_OPTEED) += fdt_mpxy_opteed
            libsbiutils-objs-$(CONFIG_FDT_MPXY_OPTEED) += mpxy/fdt_mpxy_opteed.o
            ```

            ```c
            // lib/utils/mpxy/fdt_mpxy_drivers.carray

            HEADER: sbi_utils/mpxy/fdt_mpxy.h
            TYPE: struct fdt_mpxy
            NAME: fdt_mpxy_drivers
            ```

            ```c
            // include/sbi_utils/mpxy/fdt_mpxy.h

            struct fdt_mpxy {
            	const struct fdt_match *match_table;
            	int (*init)(void *fdt, int nodeoff, const struct fdt_match *match);
            	void (*exit)(void);
            };
            ```

        - Call `drv->init()`.
            - e.g. [`mpxy_opteed_init()`](/posts/opensbi-optee/#mpxy-opteed-init)
- {{< anchor id="sbi-ecall-mpxy-handler" >}}`sbi_ecall_mpxy_handler()`
    - `sbi_ecall_mpxy_handler()` is the handler for SBI MPXY ecalls, which will call the corresponding function based on SBI function ID, e.g.
        - [`sbi_mpxy_set_shmem()`](/posts/opensbi-optee/#sbi-mpxy-set-shmem)
        - [`sbi_mpxy_send_message()`](/posts/opensbi-optee/#sbi-mpxy-send-message)
