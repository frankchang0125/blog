---
date: "2024-12-21T00:00:00+08:00"
title: "OpenSBI: OP-TEE"
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

- {{< anchor id="mpxy-opteed-init" >}}`mpxy_opteed_init()`
    - Match compatible string: `"riscv,sbi-mpxy-opteed"`.
    - Allocate channel.
    - `opteed_domain_setup()`
        - Setup domain for OP-TEE dispatcher by looking up the domain specified by `opensbi-domain-instance` phandle in DTS.
        - Assign the domain name to `opteed_domain_name`.
    - Get channel ID from DTS property: `riscv,sbi-mpxy-channel-id`.
    - Initialize channel:

        ```c
        // lib/utils/mpxy/fdt_mpxy_opteed.c

        channel->channel_id = channel_id;
        channel->send_message = mpxy_opteed_send_message;
        channel->attrs.msg_proto_id = SBI_MPXY_MSGPROTO_TEE_ID;
        channel->attrs.msg_data_maxlen = PAGE_SIZE;
        ```

    - Register channel: [`sbi_mpxy_register_channel()`](/posts/opensbi-optee/#sbi-mpxy-register-channel)
    - Assign `tdomain` (**Trusted domain**).
        - Look up the domain with the name specified in `opteed_domain_name`.
    - Assign `udomain` (**Untrusted domain**).
        - Look up the domain with the name `"untrusted-domain"`.
- {{< anchor id="mpxy-opteed-send-message" >}}`mpxy_opteed_send_message()`
    - {{< anchor id="opteed-msg-communicate" >}}If message ID is `OPTEED_MSG_COMMUNICATE` (`0x1`) (Called from `udomain`):
        - If `tdomain`’s shared memory is **NOT** valid, return `SBI_EINVAL`.
        - Otherwise:
            - Direction: `udomain` -> OpenSBI
            - Copy the source data from `udomain` to `tdomain`’s shared memory.
        - Call [`sbi_ecall_tee_domain_enter()`](/posts/opensbi-optee/#sbi-ecall-tee-domain-enter) to switch to `tdomain`.
            - Direction: `udomain` -> `tdomain`.
            - The entry point to be switched to is defined by the function type:
                - Function type is defined by the first `unsigned long` in `udomain`’s shared memory: `((ulong *)shmem_base)[0]`.
                - If function type is: `ABI_ENTRY_TYPE_FAST` (i.e. `ARM_SMCCC_FAST_CALL`)
                    - Entry point ⇒ `entry_vector_table->fast_abi_entry`
                        - `entry_vector_table` ⇒ [`thread_vector_table`](/posts/optee-threads/#hl-3-10) is defined in OP-TEE. Therefore, `entry_vector_table->fast_abi_entry` ⇒ [`vector_fast_abi_entry()`](/posts/optee-threads/#hl-5-3)
                - Otherwise:
                    - Entry point ⇒ `entry_vector_table->yield_abi_entry`
                        - `entry_vector_table` ⇒ [`thread_vector_table`](/posts/optee-threads/#hl-3-10) is defined in OP-TEE. Therefore, `entry_vector_table->yield_abi_entry` ⇒ [`vector_std_abi_entry()`](/posts/optee-threads/#hl-4-3)
    - If message ID is `OPTEED_MSG_COMPLETE` (`0x2`) (Called from `tdomain`):
        - If `udomain`’s shared memory is **NOT** valid:
            - Direction: `tdomain` -> OpenSBI.
            - If `((ulong *)msgbuf)[0]` (from `tdomain`’s shared memory, i.e. `$a0`) is `TEEABI_OPTEED_RETURN_CALL_DONE` (`0xbe000000`):
                - Register OP-TEE entry table:
                    - Set `entry_vector_table` to `((ulong *)msgbuf)[1]` (i.e. `$a1`), i.e. [`thread_vector_table`](/posts/optee-threads/#hl-3-10) defined in OP-TEE.
        - Otherwise, copy the source data from `tdomain` to `udomain`’s shared memory.
            - Direction: `tdomain` -> `udomain`.
            - Only `$a1` ~ `$a4` (from `tdomain`’s shared memory) are copied to `udomain`’s shared memory, `$a0` is skipped.
        - Call `sbi_ecall_tee_domain_exit()` to exit from the current `tdomain`.
            - The domain to switch to:
                - Switch to the previous domain
                - Switch to the use-define next domain.
                - Fallback to the root domain.
    - P.S. Domain switching only involves domain contexts save and restore. The mode is switched by `mret` in the trap handler.

---

- {{< anchor id="sbi-mpxy-register-channel" >}}`sbi_mpxy_register_channel()`
    - Initialize channel’s attributes: `mpxy_std_attrs_init()`
        - Check whether MSI, SSE and Events State capabilities are available in the domain.
    - Add the allocated channel to `mpxy_channel_list` list.
- {{< anchor id="sbi-mpxy-set-shmem" >}}`sbi_mpxy_set_shmem()`
    - Validate passed in shared memory address and size.
    - Set current hart’s `sbi_domain` shared memory address and size.
- {{< anchor id="sbi-mpxy-send-message" >}}`sbi_mpxy_send_message()`
    - Call `channel->send_message()`.
        - e.g. [`mpxy_opteed_send_message()`](/posts/opensbi-optee/#mpxy-opteed-send-message)

---

- {{< anchor id="sbi-ecall-tee-domain-enter" >}}`sbi_ecall_tee_domain_enter()`
    - The domain is `tdomain`.
    - `sbi_domain_context_set_mepc()`
        - Set `mepc` to the entry point.
    - `sbi_domain_context_enter()`
        - Switch to `tdomain`.

---

- Example DTS:

    ```c
        chosen {
            opensbi-domains {

                .....

                trusted-domain {
                    compatible = "opensbi,domain,instance";
                    regions = <0x04 0x3f>;
                    possible-harts = <0x03 0x01>;
                    next-addr = <0x00 0xf1000000>;
                    next-mode = <0x01>;
                    phandle = <0x02>;
                };

                .....
            };
        };

        sbi-mpxy-opteed {
            opensbi-domain-instance = <0x02>;
            riscv,sbi-mpxy-channel-id = <0x02>;
            compatible = "riscv,sbi-mpxy-opteed";
        };
    ```
