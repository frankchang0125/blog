---
date: "2024-12-31T00:00:00+08:00"
title: "Linux Kernel: SBI MPXY"
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

```c
// arch/riscv/kernel/mpxy-sbi.c

struct sbi_mpxy {
	void *shmem;
	phys_addr_t shmem_phys_addr;
	bool active;
};

// Define per-cpu varible: sbi_mpxy.
DEFINE_PER_CPU(struct sbi_mpxy, sbi_mpxy);
```

---

- `do_initcalls()`
    - …
        - [`sbi_mpxy_init()`](/posts/linux-kernel-sbi-mpxy/#sbi-mpxy-init)

---

- {{< anchor id="sbi-mpxy-init" >}}`sbi_mpxy_init()`
    - Check SBI version and if MPXY extension is supported by OpenSBI.
    - Setup CPUHP notifier to setup shared memory on all CPUs.

        ```c
        // arch/riscv/kernel/mpxy-sbi.c

        cpuhp_setup_state(CPUHP_AP_ONLINE_DYN,
        			    "riscv/mpxy-sbi:cpu-shmem-init",
        			    __sbi_mpxy_setup_shmem,
        			    sbi_mpxy_exit);
        ```

        - `cpuhp_thread_fun()`
            - `cpuhp_invoke_callback()`
                - [`__sbi_mpxy_setup_shmem()`](/posts/linux-kernel-sbi-mpxy/#sbi-mpxy-setup-shmem)
- {{< anchor id="sbi-mpxy-setup-shmem" >}}`__sbi_mpxy_setup_shmem()`
    - Allocates 4KB shared memory (4KB aligned).
    - Call `sbi_mpxy_set_shmem` SBI call to set up MPXY shared memory for the current core. This will invoke OpenSBI’s [`sbi_mpxy_set_shmem()`](/posts/opensbi-optee/#sbi-mpxy-set-shmem) to save the shared memory address and size into current hart domain’s `mpxy_state` (**Untrusted domain**).
- {{< anchor id="sbi-mpxy-tee-probe" >}}`sbi_mpxy_tee_probe()`
    - Check SBI version and whether MPXY extension is supported by OpenSBI.
    - Get channel ID from DTS property: `riscv,sbi-mpxy-channel-id`.
    - Read MPXY protocol ID attribute (`SBI_MPXY_ATTR_MSG_PROT_ID` (`0x0`)) from channel with [`sbi_mpxy_read_attrs()`](/posts/linux-kernel-sbi-mpxy/#sbi-mpxy-read-attrs) .
    - Check if the returned MPXY protocol ID equals `SBI_MPXY_MSGPROTO_TEE_ID` (`0x2`). Throw an error if not.
- {{< anchor id="sbi-mpxy-read-attrs" >}}`sbi_mpxy_read_attrs()`
    - Disable preemption (`get_cpu()`).
        - Shared memory is per-core. We only need to disable the preemption to prevent the shared memory of current CPU from being corrupted by the other tasks.
    - Issue SBI call: `SBI_EXT_MPXY_READ_ATTRS` to read the attribute.
    - If successful, copy the attribute from the shared memory to the buffer.
    - Enable preemption (`put_cpu()`).
- `sbi_mpxy_write_attrs()`
    - Disable preemption (`get_cpu()`).
        - Shared memory is per-core. We only need to disable the preemption to prevent the shared memory of current CPU from being corrupted by the other tasks.
    - Copy the attribute from the buffer to the shared memory.
    - Issue SBI call: `SBI_EXT_MPXY_WRITE_ATTRS` to write the attribute.
    - Enable preemption (`put_cpu()`).
- {{< anchor id="sbi-mpxy-send-message-withresp" >}}`sbi_mpxy_send_message_withresp()`
    - Disable preemption (`get_cpu()`).
        - Shared memory is per-core. We only need to disable the preemption to prevent the shared memory of current CPU from being corrupted by the other tasks.
    - Issue SBI call: `SBI_EXT_MPXY_SEND_MSG_WITH_RESP` with message ID: `OPTEED_MSG_COMMUNICATE` (`0x1`) to send the message to `tdomain` and get the response.
        - This will switch to OpenSBI: [If message ID is `OPTEED_MSG_COMMUNICATE` (`0x1`) (Called from `udomain`):](/posts/opensbi-optee/#opteed-msg-communicate). OpenSBI will then switch to the `tdomain`.
    - Enable preemption (`put_cpu()`).
