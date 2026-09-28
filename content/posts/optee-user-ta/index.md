---
date: "2024-11-30T00:00:00+08:00"
title: "OP-TEE: User TAs"
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

- {{< anchor id="tee-ta-init-user-ta-session" >}}`tee_ta_init_user_ta_session()`
    - Initialize user TA for the UUID, assign `ctx->ts_ctx.ops` to `user_ta_ops` by calling `set_ta_ctx_ops()`. `user_ta_ops` will be the global callbacks for the user TAs; assign `tee_ta_session->ts_sess.handle_scall` to [`scall_handle_user_ta()`](/posts/optee-user-ta/#hl-1-21). [`scall_handle_user_ta()`](/posts/optee-user-ta/#hl-1-21) will be the default callback to handle the syscall from user TA:

        ```c
        // core/kernel/user_ta.c

        TEE_Result tee_ta_init_user_ta_session(const TEE_UUID *uuid,
        				       struct tee_ta_session *s)
        {
        	TEE_Result res = TEE_SUCCESS;
        	struct user_ta_ctx *utc = NULL;

        	.....

        	// Initialize lists.
        	TAILQ_INIT(&utc->open_sessions);
        	TAILQ_INIT(&utc->cryp_states);
        	TAILQ_INIT(&utc->objects);
        	TAILQ_INIT(&utc->storage_enums);
        	condvar_init(&utc->ta_ctx.busy_cv);
        	utc->ta_ctx.ref_count = 1;

        	/*
        	 * Set context TA operation structure. It is required by generic
        	 * implementation to identify userland TA versus pseudo TA contexts.
        	 */
        	// utc->ta_ctx->ts_ctx.ops = [user_ta_ops](/posts/optee-user-tas/).
        	set_ta_ctx_ops(&utc->ta_ctx);

        	utc->ta_ctx.ts_ctx.uuid = *uuid;
        	// vm_info_init() will set: utc->uctx->ts_ctx = &utc->ta_ctx.ts_ctx.
        	res = vm_info_init(&utc->uctx, &utc->ta_ctx.ts_ctx);
        	if (res) {
        		condvar_destroy(&utc->ta_ctx.busy_cv);
        		free_utc(utc);
        		return res;
        	}

        	.....

        	utc->ta_ctx.is_initializing = true;

        	.....

        	s->ts_sess.ctx = &utc->ta_ctx.ts_ctx;
        	s->ts_sess.handle_scall = s->ts_sess.ctx->ops->handle_scall;
        	/*
        	 * Another thread trying to load this same TA may need to wait
        	 * until this context is fully initialized. This is needed to
        	 * handle single instance TAs.
        	 */
        	TAILQ_INSERT_TAIL(&tee_ctxes, &utc->ta_ctx, link);

        	return TEE_SUCCESS;
        }
        ```

        ```c
        // core/kernel/user_ta.c

        /*
         * Note: this variable is weak just to ease breaking its dependency chain
         * when added to the unpaged area.
         */
        const struct ts_ops user_ta_ops __weak __relrodata_unpaged("user_ta_ops") = {
        	.enter_open_session = user_ta_enter_open_session,
        	.enter_invoke_cmd = user_ta_enter_invoke_cmd,
        	.enter_close_session = user_ta_enter_close_session,
        #if defined(CFG_TA_STATS)
        	.dump_mem_stats = user_ta_enter_dump_memstats,
        #endif
        	.dump_state = user_ta_dump_state,
        #ifdef CFG_FTRACE_SUPPORT
        	.dump_ftrace = user_ta_dump_ftrace,
        #endif
        	.release_state = user_ta_release_state,
        	.destroy = user_ta_ctx_destroy,
        	.get_instance_id = user_ta_get_instance_id,
        	.handle_scall = scall_handle_user_ta,
        #ifdef CFG_TA_GPROF_SUPPORT
        	.gprof_set_status = user_ta_gprof_set_status,
        #endif
        };
        ```


---

- {{< anchor id="tee-ta-complete-user-ta-session" >}}`tee_ta_complete_user_ta_session()`
    - Call [`ldelf_load_ldelf()`](/posts/optee-ldelf/#hl-0-8) to load `ldelf` program to the memory for this User TA.
        - `ldelf` is responsible for loading the user TA ELF image residing in REE to the memory.
        - If [`ldelf_load_ldelf()`](/posts/optee-ldelf/#hl-0-8) returns `TEE_SUCCESS`, call [`ldelf_init_with_ldelf()`](/posts/optee-ldelf/#hl-1-3)
            - [`ldelf_load_ldelf()`](/posts/optee-ldelf/#hl-0-8) loads user TA ELF and fill in user TA ELF’s information to `struct user_mode_ctx`.

---

- {{< anchor id="user-ta-enter-open-session" >}}`user_ta_enter_open_session()`
    - Call [`user_ta_enter()`](/posts/optee-user-ta/#user-ta-enter) with function ID: `UTEE_ENTRY_FUNC_OPEN_SESSION`.

---

- {{< anchor id="user-ta-enter-invoke-cmd" >}}`user_ta_enter_invoke_cmd()`
    - Call [`user_ta_enter()`](/posts/optee-user-ta/#user-ta-enter) with function ID: `UTEE_ENTRY_FUNC_INVOKE_COMMAND`.

---

- {{< anchor id="user-ta-enter" >}}`user_ta_enter()`
    - Call [`thread_enter_user_mode()`](/posts/optee-threads/#thread-enter-user-mode) to switch to U-mode. `utc->utcx.entry_func` (user TA’s entry function address, filled by `ldelf`) will be called after switching to U-mode.
        - E.g. For `optee_example_hello_world`, i.e. `8aaaf200-2450-11e4-abe2-0002a5d5c51b.elf`, the `entry_func` is `0x400405f8` => `__ta_entry().`
            - `__ta_entry()` is the first user TA API called from TEE core (defined in `ta/user_ta_header.c`).
                - It’s assigned in TA’s Makefile:

                    ```makefile
                    # ta/link.mk

                    link-ldflags  = -e__ta_entry -pie
                    ```

                - `__ta_entry()` will call `__utee_entry()` (defined in `lib/libutee/user_ta_entry.c`) to invoke the function (e.g. `TA_OpenSessionEntryPoint()`, `TA_InvokeCommandEntryPoint()`… etc) defined by user TA based on function ID.
                    - User TA is linked with `libutee`.
                - And the end, `__ta_entry()` will call `__utee_return()` (defined in `lib/libutee/user_ta_entry.c`), to return from user TA.
                - `__utee_return()` is actually a syscall, syscall ID: `TEE_SCN_RETURN`. Therefore, `tee_ta_session->ts_sess.handle_scall`, e.g. [`scall_handle_user_ta()`](/posts/optee-user-ta/#hl-1-21), will eventually be called.

---

- {{< anchor id="scall-handle-user-ta" >}}`scall_handle_user_ta()`:
    - Handle syscall according to `tee_syscall_table`:

    ```c
    // core/kernel/syscall.c

    /*
     * This array is ordered according to the SYSCALL ids TEE_SCN_xxx
     */
    static const struct syscall_entry tee_syscall_table[] = {
    	SYSCALL_ENTRY(syscall_sys_return),
    	SYSCALL_ENTRY(syscall_log),
    	SYSCALL_ENTRY(syscall_panic),
    	SYSCALL_ENTRY(syscall_get_property),
    	SYSCALL_ENTRY(syscall_get_property_name_to_index),
    	SYSCALL_ENTRY(syscall_open_ta_session),
    	SYSCALL_ENTRY(syscall_close_ta_session),
    	SYSCALL_ENTRY(syscall_invoke_ta_command),
    	SYSCALL_ENTRY(syscall_check_access_rights),
    	SYSCALL_ENTRY(syscall_get_cancellation_flag),
    	SYSCALL_ENTRY(syscall_unmask_cancellation),
    	SYSCALL_ENTRY(syscall_mask_cancellation),
    	SYSCALL_ENTRY(syscall_wait),
    	SYSCALL_ENTRY(syscall_get_time),
    	SYSCALL_ENTRY(syscall_set_ta_time),
    	SYSCALL_ENTRY(syscall_cryp_state_alloc),
    	SYSCALL_ENTRY(syscall_cryp_state_copy),
    	SYSCALL_ENTRY(syscall_cryp_state_free),
    	SYSCALL_ENTRY(syscall_hash_init),
    	SYSCALL_ENTRY(syscall_hash_update),
    	SYSCALL_ENTRY(syscall_hash_final),
    	SYSCALL_ENTRY(syscall_cipher_init),
    	SYSCALL_ENTRY(syscall_cipher_update),
    	SYSCALL_ENTRY(syscall_cipher_final),
    	SYSCALL_ENTRY(syscall_cryp_obj_get_info),
    	SYSCALL_ENTRY(syscall_cryp_obj_restrict_usage),
    	SYSCALL_ENTRY(syscall_cryp_obj_get_attr),
    	SYSCALL_ENTRY(syscall_cryp_obj_alloc),
    	SYSCALL_ENTRY(syscall_cryp_obj_close),
    	SYSCALL_ENTRY(syscall_cryp_obj_reset),
    	SYSCALL_ENTRY(syscall_cryp_obj_populate),
    	SYSCALL_ENTRY(syscall_cryp_obj_copy),
    	SYSCALL_ENTRY(syscall_cryp_derive_key),
    	SYSCALL_ENTRY(syscall_cryp_random_number_generate),
    	SYSCALL_ENTRY(syscall_authenc_init),
    	SYSCALL_ENTRY(syscall_authenc_update_aad),
    	SYSCALL_ENTRY(syscall_authenc_update_payload),
    	SYSCALL_ENTRY(syscall_authenc_enc_final),
    	SYSCALL_ENTRY(syscall_authenc_dec_final),
    	SYSCALL_ENTRY(syscall_asymm_operate),
    	SYSCALL_ENTRY(syscall_asymm_verify),
    	SYSCALL_ENTRY(syscall_storage_obj_open),
    	SYSCALL_ENTRY(syscall_storage_obj_create),
    	SYSCALL_ENTRY(syscall_storage_obj_del),
    	SYSCALL_ENTRY(syscall_storage_obj_rename),
    	SYSCALL_ENTRY(syscall_storage_alloc_enum),
    	SYSCALL_ENTRY(syscall_storage_free_enum),
    	SYSCALL_ENTRY(syscall_storage_reset_enum),
    	SYSCALL_ENTRY(syscall_storage_start_enum),
    	SYSCALL_ENTRY(syscall_storage_next_enum),
    	SYSCALL_ENTRY(syscall_storage_obj_read),
    	SYSCALL_ENTRY(syscall_storage_obj_write),
    	SYSCALL_ENTRY(syscall_storage_obj_trunc),
    	SYSCALL_ENTRY(syscall_storage_obj_seek),
    	SYSCALL_ENTRY(syscall_obj_generate_key),
    	SYSCALL_ENTRY(syscall_not_supported),
    	SYSCALL_ENTRY(syscall_not_supported),
    	SYSCALL_ENTRY(syscall_not_supported),
    	SYSCALL_ENTRY(syscall_not_supported),
    	SYSCALL_ENTRY(syscall_not_supported),
    	SYSCALL_ENTRY(syscall_not_supported),
    	SYSCALL_ENTRY(syscall_not_supported),
    	SYSCALL_ENTRY(syscall_not_supported),
    	SYSCALL_ENTRY(syscall_not_supported),
    	SYSCALL_ENTRY(syscall_not_supported),
    	SYSCALL_ENTRY(syscall_not_supported),
    	SYSCALL_ENTRY(syscall_not_supported),
    	SYSCALL_ENTRY(syscall_not_supported),
    	SYSCALL_ENTRY(syscall_not_supported),
    	SYSCALL_ENTRY(syscall_not_supported),
    	SYSCALL_ENTRY(syscall_cache_operation),
    };
    ```
