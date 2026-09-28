---
date: "2024-12-20T00:00:00+08:00"
title: "OP-TEE: Misc"
author: "Frank Chang"
categories:
  - "Security"
tags:
  - "OP-TEE"
series:
  - "OP-TEE Code Trace"
---

> ⚠️ The code is based on: [https://gitlab.com/riseproject/riscv-optee/optee_os/-/tree/dev-optee-mpxy](https://gitlab.com/riseproject/riscv-optee/optee_os/-/tree/dev-optee-mpxy)
>
> Commit ID: **`75df9ba41a404aec897399ead0ff0aebcbff48ca`**

- Base addresses:
    - U-Boot SPL: `0x80000000`
    - OpenSBI: `0x80100000`
    - OP-TEE: `0xf1000000`
    - U-Boot proper: `0x81200000` / Relocated to: `0x17f76f000`, relocate offset = `0xfe558000`
        - E.g. `0x81217b66` → `0x17f76fb66`; `0x81232884` → `0x17f78a884`
    - Linux Kernel: `0x80200000` / `0xffffffff80000000`

    ```sh
    (gdb) add-symbol-file u-boot-spl 0x80000000
    (gdb) add-symbol-file fw_dynamic.elf 0x80100000
    (gdb) add-symbol-file tee.elf 0xf1000000
    ```

    ```sh
    # For the relocated U-Boot
    (gdb) add-symbol-file u-boot -o 0xfe558000
    ```


---

- When traps, `mscratch`/`sscratch`:
    - If `0`: Trap from kernel.
    - If `!0`: Trap from user.
    - See `thread_init_per_cpu()`, [`thread_trap_vect()`](/posts/optee-interrupts/#hl-1-3).
        - `thread_init_per_cpu()` sets `mscratch`/`sscratch` to `0` to indicate that the trap is from kernel.

---

- **Fast Calls** and **Yielding Calls**
    - Fast Calls execute atomic operations. The call appears to be **atomic from the perspective of the calling PE**, and **returns when the requested operation has completed**.
    - Yielding Calls start operations that **can be pre-empted by a Non-secure interrupt**. The call **can return before the requested operation has completed**.
