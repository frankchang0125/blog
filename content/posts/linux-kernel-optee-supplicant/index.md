---
date: "2024-12-28T00:00:00+08:00"
title: "Linux Kernel: OP-TEE Supplicant"
author: "Frank Chang"
categories:
  - "Security"
tags:
  - "Linux Kernel"
  - "OP-TEE"
series:
  - "OP-TEE Code Trace"
---

> ⚠️ The code is based on: [https://gitlab.com/riseproject/riscv-optee/linux/-/tree/dev-optee-mpxy](https://gitlab.com/riseproject/riscv-optee/linux/-/tree/dev-optee-mpxy)
>
> Commit ID: **`df5dc01764820f113312f7a39f221b49985bbd7a`**

- TEE supplicant
    - `/etc/init.d/S30-tee-supplicant`
    - `/dev/teepriv0`

---

- `main()`
    - [`process_one_request()`](/posts/linux-kernel-optee-supplicant/#process-one-request)
- {{< anchor id="process-one-request" >}}`process_one_request()`
    - `read_request()`
        - Issue [`TEE_IOC_SUPPL_RECV`](/posts/linux-kernel-tee/#tee-ioc-suppl-recv) [ioct](/posts/linux-kernel-tee/#tee-ioctl)l to receive the TEE supplicant request from OP-TEE.
        - This will be blocked until TEE supplicant request is received.
    - Spawn a new thread to process for the new request: `thread_main()`
        - [`process_one_request()`](/posts/linux-kernel-optee-supplicant/#process-one-request)
    - The original thread will continue to handle the TEE supplicant request from OP-TEE:
        - If RPC command:
            - `OPTEE_MSG_RPC_CMD_LOAD_TA`:
                - Call `load_ta()` to load the TA according to the UUID.
                    - `load_ta()` will call `TEECI_LoadSecureModule()` to load the TA.
                        - `TEECI_LoadSecureModule()` will call `fopen()`, `ftell()` to open TA and get the size of TA.
                        - If the buffer size (`ta_size`) is not enough to hold the TA, return the required size to let the caller increase the buffer size and try again.
                        - Otherwise, call `fread()` to read TA and save it to the buffer.
            - …
        - Call [`write_response()`](/posts/linux-kernel-optee-supplicant/#write-response) to send the TEE supplicant response to OP-TEE.
- {{< anchor id="write-response" >}}`write_response()`
    - Issue [`TEE_IOC_SUPPL_SEND`](/posts/linux-kernel-tee/#tee-ioc-suppl-send) [ioctl](/posts/linux-kernel-tee/#tee-ioctl) to send the TEE supplicant response to OP-TEE.
        - This will unblock [Call `wait_for_completion_interruptible(**&req->c**)` to wait for TEE supplicant to process.](/posts/linux-kernel-optee/#optee-supp-thrd-req)
