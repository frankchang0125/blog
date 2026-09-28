---
date: "2024-12-18T00:00:00+08:00"
title: "OP-TEE: libteec"
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

- `TEEC_InitializeContext()`
    - `teec_open_dev()`
        - Open `/dev/teeX` device.
- `TEEC_OpenSession()`
    - Call `ioctl()` with `TEE_IOC_OPEN_SESSION` command.
        - This will eventually trap to Linux Kernel’s [`tee_ioctl()`](/posts/linux-kernel-tee/#tee-ioctl).
- `TEEC_InvokeCommand()`
    - Call `ioctl()` with `TEE_IOC_INVOKE` command. The command ID for TA is passed through `arg->func`.
- `TEEC_CloseSession()`
    - Call `ioctl()` with `TEE_IOC_CLOSE_SESSION` command.
- `TEEC_FinalizeContext()`
    - Close `/dev/teeX` device.
