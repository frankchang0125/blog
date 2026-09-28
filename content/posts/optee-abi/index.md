---
date: "2024-12-15T00:00:00+08:00"
title: "OP-TEE: ABI"
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

- {{< anchor id="std-abi-entry" >}}`std_abi_entry()`
    - If `args->a0`:
        - `OPTEE_ABI_CALL_WITH_ARG` or `OPTEE_ABI_CALL_WITH_RPC_ARG`:
            - `std_entry_with_parg()`
                - `call_entry_std()`
                    - `tee_entry_std()`
                        - [`__tee_entry_std()`](/posts/optee-abi/#tee-entry-std-internal)
        - `OPTEE_ABI_CALL_WITH_REGD_ARG`:
            - `std_entry_with_regd_arg()`
- {{< anchor id="tee-entry-std-internal" >}}`__tee_entry_std()`
    - Call `thread_set_foreign_intr()` to enable all foreign interrupts.
    - If `arg->cmd`:
        - `OPTEE_MSG_CMD_OPEN_SESSION`:
            - [`entry_open_session()`](/posts/optee-ta/#entry-open-session)
        - `OPTEE_MSG_CMD_CLOSE_SESSION`:
            - `entry_close_session()`
        - `OPTEE_MSG_CMD_INVOKE_COMMAND`:
            - [`entry_invoke_command()`](/posts/optee-ta/#entry-invoke-command)
        - `OPTEE_MSG_CMD_CANCEL`:
            - `entry_cancel()`
        - `OPTEE_MSG_CMD_REGISTER_SHM`:
            - `register_shm()`
        - `OPTEE_MSG_CMD_UNREGISTER_SHM`:
            - `unregister_shm()`
        - …
