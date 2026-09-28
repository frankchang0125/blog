---
date: "2024-10-18T00:00:00+08:00"
title: "OP-TEE: SBI MPXY"
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

- `mpxy_opteed_channel_init()`
    - Check if MPXY extension is supported by OpenSBI.
    - Extract MPXY channel ID from DT:
        - `compatible = “riscv,sbi-mpxy-opteed";`
        - `riscv,sbi-mpxy-channel-id` ← Defines MPXY channel ID.
            - Save MPXY channel ID to `mpxy_opteed_ctx.channel_id`.
        - `opensbi-domain-instance` ← Defines the OpenSBI domain used by OP-TEE (not used by OP-TEE).

            ```c
            chosen {
                opensbi-domains {
                    trusted-domain {
                    compatible = "opensbi,domain,instance";
                    regions = <0x04 0x3f>;
                    possible-harts = <0x03 0x01>;
                    next-addr = <0x00 0xf1000000>;
                    next-mode = <0x01>;
                    phandle = <0x02>;
                };
            };

            sbi-mpxy-opteed {
                opensbi-domain-instance = <0x02>;
                riscv,sbi-mpxy-channel-id = <0x02>;
                compatible = "riscv,sbi-mpxy-opteed";
            };
            ```


- `sbi_mpxy_setup_shmem()`
    - Allocates 4KB MPXY shared memory (4KB aligned).
    - Call `sbi_mpxy_set_shmem` SBI call to set up the allocated MPXY shared memory for the current core. This will invoke OpenSBI’s [`sbi_mpxy_set_shmem()`](/posts/opensbi-optee/#sbi-mpxy-set-shmem) to save the shared memory address and size into current hart `tdomain`’s `mpxy_state`.
- `thread_return_to_udomain_by_mpxy()`
    - Save the args (`arg0`, `arg1`, `arg2`, `arg3`, `arg4`) to TX buffer (`struct optee_msg_payload`).
    - Issue SBI call: Send Message with Response (FID #4) with message ID: `OPTEED_MSG_COMPLETE` (`0x2`).
        - [`sbi_mpxy_send_message_withresp()`](/posts/optee-sbi-mpxy/#sbi-mpxy-send-message-withresp)
- {{< anchor id="sbi-mpxy-send-message-withresp" >}}`sbi_mpxy_send_message_withresp()`
    - Send Message with Response (FID #4)
        1. Copy the message from TX buffer to the shared memory.
        2. Issue the SBI call. Switch to OpenSBI and jump to: [`sbi_ecall_mpxy_handler()`](/posts/opensbi-mpxy/#sbi-ecall-mpxy-handler)
        3. If `ret.error` is success:
            1. Copy the response message from the shared memory to the RX buffer.
