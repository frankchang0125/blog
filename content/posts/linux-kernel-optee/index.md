---
date: "2024-12-26T00:00:00+08:00"
title: "Linux Kernel: OP-TEE"
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

- OP-TEE provides a **pseudo Trusted Application (PTA):** `drivers/tee/optee/device.c` in order to support **device enumeration**. In other words, OP-TEE driver invokes **this application** to retrieve **a list of Trusted Applications** which can be registered as **devices** on the TEE bus.

---

```c
// drivers/tee/optee/optee_private.h

/**
 * struct optee - main service struct
 * @supp_teedev:	supplicant device
 * @teedev:		client device
 * @ops:		internal callbacks for different ways to reach secure
 *			world
 * @ctx:		driver internal TEE context
 * @smc:		specific to SMC ABI
 * @ffa:		specific to FF-A ABI
 * @call_queue:		queue of threads waiting to call @invoke_fn
 * @notif:		notification synchronization struct
 * @supp:		supplicant synchronization struct for RPC to supplicant
 * @pool:		shared memory pool
 * @rpc_param_count:	If > 0 number of RPC parameters to make room for
 * @scan_bus_done	flag if device registation was already done.
 * @scan_bus_work	workq to scan optee bus and register optee drivers
 */
struct optee {
	struct tee_device *supp_teedev;
	struct tee_device *teedev;
	const struct optee_ops *ops;
	struct tee_context *ctx;
	union {
		struct optee_smc smc;
		struct optee_ffa ffa;
	};
	struct optee_shm_arg_cache shm_arg_cache;
	struct optee_call_queue call_queue;
	struct optee_notif notif;
	struct optee_supp supp;
	struct tee_shm_pool *pool;
	unsigned int rpc_param_count;
	bool   scan_bus_done;
	struct work_struct scan_bus_work;
};
```

```c
// drivers/tee/optee/optee_private.h

/**
 * struct optee_ops - OP-TEE driver internal operations
 * @do_call_with_arg:	enters OP-TEE in secure world
 * @to_msg_param:	converts from struct tee_param to OPTEE_MSG parameters
 * @from_msg_param:	converts from OPTEE_MSG parameters to struct tee_param
 *
 * These OPs are only supposed to be used internally in the OP-TEE driver
 * as a way of abstracting the different methogs of entering OP-TEE in
 * secure world.
 */
struct optee_ops {
	int (*do_call_with_arg)(struct tee_context *ctx,
				struct tee_shm *shm_arg, u_int offs,
				bool system_thread);
	int (*to_msg_param)(struct optee *optee,
			    struct optee_msg_param *msg_params,
			    size_t num_params, const struct tee_param *params);
	int (*from_msg_param)(struct optee *optee, struct tee_param *params,
			      size_t num_params,
			      const struct optee_msg_param *msg_params);
};
```

---

- `do_initcalls()`
    - …
        - [`optee_core_init()`](/posts/linux-kernel-optee/#optee-core-init)

---

- {{< anchor id="optee-core-init" >}}`optee_core_init()`
    - [`optee_smc_abi_register()`](/posts/linux-kernel-optee/#optee-smc-abi-register)
    - `optee_ffa_abi_register()`
        - Not supported in RISC-V.
- {{< anchor id="optee-smc-abi-register" >}}`optee_smc_abi_register()`
    - Register `optee_driver`.

        ```c
        // drivers/tee/optee/smc_abi.c

        static struct platform_driver optee_driver = {
        	.probe  = optee_probe,
        	.remove_new = optee_smc_remove,
        	.shutdown = optee_shutdown,
        	.driver = {
        		.name = "optee",
        		.of_match_table = optee_dt_match,
        	},
        };
        ```

        - …
            - [`optee_probe()`](/posts/linux-kernel-optee/#optee-probe)
- {{< anchor id="optee-probe" >}}`optee_probe()`
    - [`sbi_mpxy_tee_probe()`](/posts/linux-kernel-sbi-mpxy/#sbi-mpxy-tee-probe)
    - Get invoke function.


        - E.g. [`optee_smccc_smc()`](/posts/linux-kernel-optee/#optee-smccc-smc)
            - P.S. SMCCC = SMC Calling Convention, defined by ARM.
    - `optee_msg_api_uid_is_optee_api()`
        - Check UUID:
            - Call the invoke function, e.g. [`optee_smccc_smc()`](/posts/linux-kernel-optee/#optee-smccc-smc) with function ID: `OPTEE_SMC_CALLS_UID` to retrieve the UUID.
                - OP-TEE will return UUID. Return `true` if the returned UUID is `OPTEE_MSG_UID` (`384fb3e0-e7f8-11e3-af63-0002a5d5c51b`), `false` otherwise.
    - `optee_msg_get_os_revision()`
        - Get OS revision
            - Call the invoke function, e.g. [`optee_smccc_smc()`](/posts/linux-kernel-optee/#optee-smccc-smc) with function ID: `OPTEE_SMC_CALL_GET_OS_REVISION` to get OS version.
                - OP-TEE will return the OS revision.
    - `optee_msg_api_revision_is_compatible()`
        - Check whether the API revision is compatible:
            - Call the invoke function, e.g. [`optee_smccc_smc()`](/posts/linux-kernel-optee/#optee-smccc-smc) with function ID: `OPTEE_SMC_CALLS_REVISION` to retrieve the API revision.
                - OP-TEE will return the API revision. Return `true` if the returned API revision is `OPTEE_MSG_REVISION` (`2.0`), `false` otherwise.
    - `optee_msg_get_thread_count()`
        - Get OP-TEE thread count:
            - Call the invoke function, e.g. [`optee_smccc_smc()`](/posts/linux-kernel-optee/#optee-smccc-smc) with function ID: `OPTEE_SMC_GET_THREAD_COUNT` to get the OP-TEE thread count.
    - `optee_msg_exchange_capabilities()`
        - Exchange capabilities between Linux and OP-TEE:
            - Linux sets:
                - If UP, `OPTEE_SMC_NSEC_CAP_UNIPROCESSOR`.
            - Call the invoke function, e.g. [`optee_smccc_smc()`](/posts/linux-kernel-optee/#optee-smccc-smc) with function ID: `OPTEE_SMC_EXCHANGE_CAPABILITIES` to exchange the capabilities between Linux and OP-TEE.
            - OP-TEE sets:
                - If reserved shared memory is enabled in secure world: `OPTEE_ABI_SEC_CAP_HAVE_RESERVED_SHM`.
                - If dynamic shared memory is enabled in secure world: `OPTEE_ABI_SEC_CAP_DYNAMIC_SHM`.
                - If virtualization is enabled in secure world: `OPTEE_ABI_SEC_CAP_VIRTUALIZATION`.
                - Shared memory with a NULL reference is supported in secure world: `OPTEE_ABI_SEC_CAP_MEMREF_NULL`.
                - If asynchronous notification of normal world is supported in secure world: `OPTEE_ABI_SEC_CAP_ASYNC_NOTIF`.
                - Pre-allocating RPC arg struct is supported in secure world: `OPTEE_ABI_SEC_CAP_RPC_ARG`.
    - If the capabilities has `OPTEE_ABI_SEC_CAP_DYNAMIC_SHM`, create page-based allocator pool based on `alloc_pages()`.
    - If the capabilities has `OPTEE_ABI_SEC_CAP_HAVE_RESERVED_SHM`, use the static memory pool.
        - Call the invoke function, e.g. [`optee_smccc_smc()`](/posts/linux-kernel-optee/#optee-smccc-smc) with function ID: `OPTEE_SMC_GET_SHM_CONFIG` to get the base address and the size of the static memory pool allocated by OP-TEE.
            - OP-TEE returns `default_nsec_shm_paddr` and `default_nsec_shm_size` indicating the physical base address and the size of the non-secure shared memory initialized in `teecore_init_pub_ram()`.
                - Non-secured shared memory is declared statically in OP-TEE:

                    ```c
                    // core/arch/riscv/kernel/sbi_mpxy.c

                    #ifdef CFG_CORE_RESERVED_SHM
                    register_phys_mem(MEM_AREA_NSEC_SHM, TEE_SHMEM_START, TEE_SHMEM_SIZE);
                    #endif
                    ```

                - `teecore_init_pub_ram()` is declared with `early_init()` when `CFG_CORE_RESERVED_SHM` is defined.
    - Allocate `struct optee`:
        - `optee->ops = &optee_ops`

        ```c
        // drivers/tee/optee/smc_abi.c

        static const struct optee_ops optee_ops = {
        	.do_call_with_arg = optee_smc_do_call_with_arg,
        	.to_msg_param = optee_to_msg_param,
        	.from_msg_param = optee_from_msg_param,
        };
        ```

    - Call `tee_device_alloc()` to allocate a new `struct tee_device` instance for TEE client device: `optee-clnt` with `optee_clnt_desc`. `struct optee` is passed as `driver_data`.

        ```c
        // drivers/tee/optee/smc_abi.c

        static const struct tee_desc optee_clnt_desc = {
        	.name = DRIVER_NAME "-clnt",
        	.ops = &optee_clnt_ops,
        	.owner = THIS_MODULE,
        };
        ```

        ```c
        // drivers/tee/optee/smc_abi.c

        static const struct tee_driver_ops optee_clnt_ops = {
        	.get_version = optee_get_version,
        	.open = optee_smc_open,
        	.release = optee_release,
        	.open_session = optee_open_session,
        	.close_session = optee_close_session,
        	.system_session = optee_system_session,
        	.invoke_func = optee_invoke_func,
        	.cancel_req = optee_cancel_req,
        	.shm_register = optee_shm_register,
        	.shm_unregister = optee_shm_unregister,
        };
        ```

        - The allocated TEE client device is assigned to `optee->teedev`.
    - Call `tee_device_alloc()` to allocate a new `struct tee_device` instance for TEE supplicant device: `optee-supp` with `optee_supp_desc`. `struct optee` is passed as `driver_data`.

        ```c
        // drivers/tee/optee/smc_abi.c

        static const struct tee_desc optee_supp_desc = {
        	.name = DRIVER_NAME "-supp",
        	.ops = &optee_supp_ops,
        	.owner = THIS_MODULE,
        	// If TEE_DESC_PRIVILEGED is set, the name of
        	// tee_device will be: teeprivX.
        	// Otherwise, the name of tee_device will be: teeX.
        	.flags = TEE_DESC_PRIVILEGED,
        };
        ```

        ```c
        // drivers/tee/optee/smc_abi.c

        static const struct tee_driver_ops optee_supp_ops = {
        	.get_version = optee_get_version,
        	.open = optee_smc_open,
        	.release = optee_release_supp,
        	.supp_recv = optee_supp_recv,
        	.supp_send = optee_supp_send,
        	.shm_register = optee_shm_register_supp,
        	.shm_unregister = optee_shm_unregister_supp,
        };
        ```

        - The allocated TEE supplicant device is assigned to `optee->supp_teedev`.
    - Call [`tee_device_register()`](/posts/linux-kernel-tee/#tee-device-register) to register `optee-clnt` device.
    - Call [`tee_device_register()`](/posts/linux-kernel-tee/#tee-device-register) to register `optee-supp` device.
    - Call `platform_set_drvdata()` to set `optee` device driver data to: `struct optee`.
    - Open OP-TEE client device (`optee->teedev`):
        - [`tee_open()`](/posts/linux-kernel-tee/#tee-open)
            - This will call `optee_clnt_desc->open()`:
                - {{< anchor id="optee-smc-open" >}}`optee_smc_open()`
                    - [`optee_open()`](/posts/linux-kernel-optee/#optee-open)
    - If `OPTEE_SMC_SEC_CAP_ASYNC_NOTIF` capability is supported:
        - Call `platform_get_irq()` to get (non-secure) IRQ **0**.
        - Call `optee_smc_notif_init_irq()` to request IRQ.
            - If `irq_is_percpu_devid()`:
                - `init_pcpu_irq()`
                    - `request_percpu_irq()`
                        - IRQ handler: `notif_pcpu_irq_handler()`
                            - `irq_handler()`
                    - Init work: `optee->smc.notif_pcpu_work`:
                        - Handler: `notif_pcpu_irq_work_fn()`
                            - [`optee_do_bottom_half()`](/posts/linux-kernel-optee/#optee-do-bottom-half)
                    - Create work queue: `optee->smc.notif_pcpu_wq`
                        - `optee->smc.notif_pcpu_work` is queued into `optee->scm.notif_pcpu_wq` in `notif_pcpu_irq_handler()`
            - Otherwise:
                - `init_irq()`
                    - `request_threaded_irq()`
                        - IRQ handler: `notif_irq_handler()`
                            - `irq_handler()`
                        - Handler thread: `notif_irq_thread_fn()`
                            - [`optee_do_bottom_half()`](/posts/linux-kernel-optee/#optee-do-bottom-half)
        - Call [`optee_enumerate_devices()`](/posts/linux-kernel-optee/#optee-enumerate-devices) to enumerate PTA devices (`PTA_CMD_GET_DEVICES`).
- {{< anchor id="optee-smccc-smc" >}}`optee_smccc_smc()`
    - Call [`sbi_mpxy_send_message_withresp()`](/posts/linux-kernel-sbi-mpxy/#sbi-mpxy-send-message-withresp) to message ID: `OPTEED_MSG_COMMUNICATE` (`0x01`).
- {{< anchor id="optee-open" >}}`optee_open()`
    - Allocate `struct optee_context_data`.
    - If `tee_device` is a  OP-TEE supplicant devices:
        - If `!optee->scan_bus_done`:
            - Initialize `scan_bus_work` work queue.
            - Handler: [`optee_bus_scan()`](/posts/linux-kernel-optee/#optee-bus-scan)
            - Schedule the work queue.
            - Set `optee->scan_bus_done` to `true`.
    - Assign the allocated `struct optee_context_data` to `tee_context->data`.
- {{< anchor id="optee-open-session" >}}`optee_open_session()`
    - Call `optee_get_msg_arg()` to allocate the shared memory for the message arg (`struct optee_msg_arg`).
    - Call `tee_session_calc_client_uuid()` to create Linux environment client UUID from the session arg.
    - Initialize message arg and add the meta parameters needed when opening a session with the session arg and Linux environment client UUID.
    - Call `optee->ops->to_msg_param()` to convert the passed-in `struct tee_param` to `OPTEE_MSG` parameters (`struct optee_msg_arg`). Save the converted `OPTEE_MSG` parameters to message arg.
        - E.g. `optee_to_msg_param()`
    - Call `optee->ops->do_call_with_arg()` with the command: `OPTEE_MSG_CMD_OPEN_SESSION` to enter OP-TEE in secure world with the message arg.
        - E.g. [`optee_smc_do_call_with_arg()`](/posts/linux-kernel-optee/#optee-smc-do-call-with-arg)
        - OP-TEE will call [`entry_open_session()`](/posts/optee-ta/#entry-open-session) to open the session based on the **UUID** (`tee_ioctl_open_session_arg->uuid`, e.g. PTA UUID) passed from Linux. The UUID is used to look up or initialize the TA with the same UUID. The opened session ID is saved to the message arg and passed back to Linux.
    - If [`optee_smc_do_call_with_arg()`](/posts/linux-kernel-optee/#optee-smc-do-call-with-arg) returns success, add the session to the sessions list (`ctxdata->sess_list`)

    - Call `optee->ops->from_msg_param()` to convert the returned session `OPTEE_MSG` parameters to `struct tee_param`. Update the results to session arg.
        - E.g. `optee_from_msg_param()`
            - Store the open session ID to `struct tee_ioctl_open_session_arg->session`.
        - If conversion fails, call `optee_close_session()` to close the session.
    - Call `optee_free_msg_arg()` to free the allocated message arg.
- {{< anchor id="optee-smc-do-call-with-arg" >}}`optee_smc_do_call_with_arg()`
    - Initialize RPC parameter (`struct optee_rpc_param`) from the message arg.
        - message arg is stored in the shared memory indicated by `shm` and `offs`.
    - Call `optee->smc.invoke_fn()` to enter OP-TEE in secure world with the message arg.
    - If SMC return is a RPC:
        - Call [`optee_handle_rpc()`](/posts/linux-kernel-optee/#optee-handle-rpc) to handle the RPC from OP-TEE.
- {{< anchor id="optee-invoke-func" >}}`optee_invoke_func()`
    - Similar flow to [`optee_open_session()`](/posts/linux-kernel-optee/#optee-open-session). Except calling `optee->ops->do_call_with_arg()` with the command: `OPTEE_MSG_CMD_INVOKE_COMMAND` to enter OP-TEE in secure world with the message arg.
        - E.g. [`optee_smc_do_call_with_arg()`](/posts/linux-kernel-optee/#optee-smc-do-call-with-arg)
        - OP-TEE will call [`entry_invoke_command()`](/posts/optee-ta/#entry-invoke-command) to invoke the command based on the passed-in function (e.g. `PTA_CMD_GET_DEVICES`) in the opened session.
            - E.g. If the opened session is registered for pseudo TA (OP-TEE: `core/pta/pseudo_ta.c`). The `enter_invoke_cmd()` callback in OP-TEE is: [`pseudo_ta_enter_invoke_cmd()`](/posts/optee-pseudo-ta/#pseudo-ta-enter-invoke-cmd).
- {{< anchor id="optee-bus-scan" >}}`optee_bus_scan()`
    - Call [`optee_enumerate_devices()`](/posts/linux-kernel-optee/#optee-enumerate-devices) to enumerate PTA supplicant devices (`PTA_CMD_GET_DEVICES_SUPP`).
- {{< anchor id="optee-do-bottom-half" >}}`optee_do_bottom_half()`
    - Call `optee->ops->do_call_with_arg()` with the command: `OPTEE_MSG_CMD_DO_BOTTOM_HALF` to schedule bottom half processing of a driver.
- {{< anchor id="optee-enumerate-devices" >}}`optee_enumerate_devices()`
    - Enumerate the TA devices.
    - `__optee_enumerate_devices()`
        - PTA UUID:

            ```c
            // drivers/tee/optee/device.c

            const uuid_t pta_uuid =
            		UUID_INIT(0x7011a688, 0xddde, 0x4053,
            			  0xa5, 0xa9, 0x7b, 0x3c, 0x4d, 0xdf, 0x13, 0xb8);
            ```

        - Call [`tee_client_open_context()`](/posts/linux-kernel-tee/#tee-client-open-context) to open TEE devices with OP-TEE driver.
        - Prepare session arg (`struct tee_ioctl_open_session_arg`).
        - Call [`tee_client_open_session()`](/posts/linux-kernel-tee/#tee-client-open-session) to open session with device enumeration pseudo Trusted Application with the session arg using PTA UUID.
        - Call [`get_devices()`](/posts/linux-kernel-optee/#get-devices) to get the required shared memory size for UUIDs of pseudo TAs.
            - Shared memory size = sizeof(UUID) * Number of pseudo TAs
        - After retrieving the shared memory size, call `tee_shm_alloc_kernel_buf()` to allocate the shared memory buffer for UUIDs of pseudo TAs.
        - Call [`get_devices()`](/posts/linux-kernel-optee/#get-devices) again to get the UUIDs of pseudo TAs. The UUIDs of psuedo TA are stored in the allocated shared memory by OP-TEE.
        - For each pseudo TA, call `optee_register_device()` to register the device to TEE bus (`tee_bus_type`).
            - This will eventually call `tee_client_device_match()` to match the driver for the TA device based on UUID. Driver’s `probe()` callback will be called if matches.
                - For example: `optee-rngs` driver (`drivers/char/hw_random/optee-rngs.c`).
                    - OP-TEE driver is implemented as a module and is initialized through `module_init()`. OP-TEE driver sets its `bus` to `tee_bus_type`.

                    ```c
                    // drivers/char/hw_random/optee-rngs.c

                    static struct tee_client_driver optee_rng_driver = {
                    	.id_table	= optee_rng_id_table,
                    	.driver		= {
                    		.name		= DRIVER_NAME,
                    		**.bus		= &tee_bus_type,**
                    		.probe		= optee_rng_probe,
                    		.remove		= optee_rng_remove,
                    	},
                    };
                    ```

- {{< anchor id="get-devices" >}}`get_devices()`
    - Call [`tee_client_invoke_func()`](/posts/linux-kernel-tee/#tee-client-invoke-func) to invoke the function (e.g. `PTA_CMD_GET_DEVICES` to get the pseudo TAs) to OP-TEE.

---

- {{< anchor id="optee-handle-rpc" >}}`optee_handle_rpc()`
    - If `optee_rpc_param->a0`:
        - `OPTEE_SMC_RPC_FUNC_CMD` function:
            - Get `arg` (struct optee_msg_arg) from the shared memory.
            - [`handle_rpc_func_cmd()`](/posts/linux-kernel-optee/#handle-rpc-func-cmd) with `arg`.
        - …
    - Set `param->a0` to `OPTEE_SMC_CALL_RETURN_FROM_RPC` to indicate that we have successfully handled the RPC.
- {{< anchor id="handle-rpc-func-cmd" >}}`handle_rpc_func_cmd()`
    - If `arg->cmd`:
        - `OPTEE_RPC_CMD_SHM_ALLOC`:
            - `handle_rpc_func_cmd_shm_alloc()`
        - `OPTEE_RPC_CMD_SHM_FREE`:
            - `handle_rpc_func_cmd_shm_free()`
        - Otherwise:
            - [`optee_rpc_cmd()`](/posts/linux-kernel-optee/#optee-rpc-cmd)
- {{< anchor id="optee-rpc-cmd" >}}`optee_rpc_cmd()`
    - If `arg->cmd`:
        - `OPTEE_RPC_CMD_GET_TIME`:
            - `handle_rpc_func_cmd_get_time()`
        - …
        - Otherwise:
            - [`handle_rpc_supp_cmd()`](/posts/linux-kernel-optee/#handle-rpc-supp-cmd)
- {{< anchor id="handle-rpc-supp-cmd" >}}`handle_rpc_supp_cmd()`
    - [`optee_supp_thrd_req()`](/posts/linux-kernel-optee/#optee-supp-thrd-req)
- {{< anchor id="optee-supp-thrd-req" >}}`optee_supp_thrd_req()`
    - Insert the RPC request into the request list (`supp->reqs`).
    - Call `complete(**&supp->reqs_c**)` to tell an eventual waiter there's a new request.
        - This will unblock TEE supplicant: [Otherwise, call `wait_for_completion_interruptible(**&supp->reqs_c**)` to wait for the new request from OP-TEE.](/posts/linux-kernel-optee/#optee-supp-thrd-req)
    - Call `wait_for_completion_interruptible(**&req->c**)` to wait for TEE supplicant to process.
        - This will be blocked until TEE supplicant handles the RPC.
    - Return the result.
- {{< anchor id="optee-supp-recv" >}}`optee_supp_recv()` - TEE supplicant waits for OP-TEE’s request
    - Check if we could pop the entry from the request list (`supp->reqs`). If yes, extract the RPC parameters.
    - Otherwise, call `wait_for_completion_interruptible(**&supp->reqs_c**)` to wait for the new request from OP-TEE.
- {{< anchor id="optee-supp-send" >}}`optee_supp_send()` - TEE supplicant handles the RPC and sends the response to OP-TEE.
    - Set the return parameters (`struct tee_param`).
    - Call `complete(**&req->c**)` to unblock [Call `wait_for_completion_interruptible(**&req->c**)` to wait for TEE supplicant to process.](/posts/linux-kernel-optee/#optee-supp-thrd-req)
