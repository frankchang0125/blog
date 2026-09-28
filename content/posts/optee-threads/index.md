---
date: "2024-10-14T00:00:00+08:00"
title: "OP-TEE: Threads"
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

```c
// core/arch/riscv/include/kernel/thread_arch.h

struct thread_core_local {
	unsigned long x[4];
	uint32_t hart_id;
	vaddr_t tmp_stack_va_end;
	short int curr_thread;
	uint32_t flags;
	vaddr_t abt_stack_va_end;
#ifdef CFG_TEE_CORE_DEBUG
	unsigned int locked_count; /* Number of spinlocks held */
#endif
#ifdef CFG_CORE_DEBUG_CHECK_STACKS
	bool stackcheck_recursion;
#endif
#ifdef CFG_FAULT_MITIGATION
	struct ftmn_func_arg *ftmn_arg;
#endif
} THREAD_CORE_LOCAL_ALIGNED;
```

```c
// core/kernel/thread.c

// Per-core local threads.
struct thread_core_local thread_core_local[CFG_TEE_CORE_NB_CORE] __nex_bss;
```

---

```c
// core/arch/riscv/kernel/entry.S

/*
 * Implement based on the transport method used to communicate between
 * untrusted domain and trusted domain. It could be an SBI/ECALL-based to
 * a security monitor running in M-Mode and panic or messaging-based across
 * domains where we return to a messaging callback which parses and handles
 * messages.
 *
 * void thread_return_to_udomain(unsigned long arg0, unsigned long arg1,
 *                               unsigned long arg2, unsigned long arg3,
 *                               unsigned long arg4, unsigned long arg5);
 */
FUNC thread_return_to_udomain , :
	/* Caller should provide arguments in a0~a5 */
	// E.g. when booting:
	//   $a0: If boot core: TEEABI_OPTEED_RETURN_ENTRY_DONE.
	//        Otherwise: TEEABI_OPTEED_RETURN_ON_DONE.
	//   $a1: If boot core: thread_vector_table.
	//        Otherwise: 0x0 (OPTEE_ABI_RETURN_OK) on success
	//                   or anything else to indicate error condition.
	//   $a2 ~ $a5: 0x0
#if defined(CFG_RISCV_WITH_M_MODE_SM)
	jal	[thread_return_to_udomain_by_mpxy](/posts/optee-sbi-mpxy/)
#else
	/* Other protocol */
#endif
	/* ABI to REE should not return */
	panic_at_abi_return
END_FUNC thread_return_to_udomain
```

---

```c
// core/arch/riscv/kernel/thread_optee_abi_rv.S

/*
 * Vector table supplied to M-mode secure monitor (e.g., openSBI) at
 * initialization.
 *
 * Note that M-mode secure monitor depends on the layout of this vector table,
 * any change in layout has to be synced with M-mode secure monitor.
 */
FUNC thread_vector_table , : , .identity_map, , nobti
	.option push
	.option norvc
	j   [vector_std_abi_entry](/posts/optee-threads/)
	j   [vector_fast_abi_entry](/posts/optee-threads/)
	j   .
	j   .
	j   .
	j   .
	j   vector_fiq_entry
	j   .
	j   .
	.option pop
END_FUNC thread_vector_table
DECLARE_KEEP_PAGER thread_vector_table
```

```c
// core/arch/riscv/kernel/thread_optee_abi_rv.S

LOCAL_FUNC vector_std_abi_entry, : , .identity_map
	// sbi_mpxy_get_shmem() returns VA of the shared memory.
	jal	sbi_mpxy_get_shmem
	// Shared memory contents: struct thread_abi_args *arg
	jal	[thread_handle_std_abi](/posts/optee-threads/)
	/*
	 * Normally thread_handle_std_abi() should return via
	 * thread_exit(), thread_rpc(), but if thread_handle_std_abi()
	 * hasn't switched stack (error detected) it will do a normal "C"
	 * return.
	 */
	/* Restore thread_handle_std_abi() return value */
	mv	a1, a0
	li	a2, 0
	li	a3, 0
	li	a4, 0
	// Function ID: This will be placed as the first item in the shared memory.
	li	a0, TEEABI_OPTEED_RETURN_CALL_DONE
	mv	a5, zero

	/* Return to untrusted domain */
	j	[thread_return_to_udomain](/posts/optee-threads/)
END_FUNC vector_std_abi_entry
```

```c
// core/arch/riscv/kernel/thread_optee_abi_rv.S

LOCAL_FUNC vector_fast_abi_entry , : , .identity_map
  // sbi_mpxy_get_shmem() returns VA of the shared memory.
	jal	sbi_mpxy_get_shmem
	// Save the shared memory contents to $a0 ~ $a7
	mv	t0, a0
	ld	a0, 0(t0)
	ld	a1, 8(t0)
	ld	a2, 16(t0)
	ld	a3, 24(t0)
	ld	a4, 32(t0)
	ld	a5, 40(t0)
	ld	a6, 48(t0)
	ld	a7, 56(t0)

	// Allocate space on stack to save $a0 ~ $a7.
	addi    sp, sp, -THREAD_ABI_ARGS_SIZE
	// Store $a0 ~ $a7 to stack so that we can pass
	// the pointer to thread_handle_fast_abi()
	store_xregs sp, THREAD_ABI_ARGS_A0, REG_A0, REG_A7
	mv      a0, sp
	jal	thread_handle_fast_abi
	// Restore $a0 ~ $a7 from the stack.
	load_xregs sp, THREAD_ABI_ARGS_A0, REG_A1, REG_A7
	// Release the allocated space.
	addi    sp, sp, THREAD_ABI_ARGS_SIZE

	// Function ID: This will be placed as the first item in the shared memory.
	li	a0, TEEABI_OPTEED_RETURN_CALL_DONE
	/* Return to untrusted domain */
	j	[thread_return_to_udomain](/posts/optee-threads/)
END_FUNC vector_fast_abi_entry
```

- `thread_handle_std_abi()`
    - If `args->a0 == OPTEE_ABI_CALL_RETURN_FROM_RPC`:
        - [`thread_resume_from_rpc()`](/posts/optee-threads/#thread-resume-from-rpc)
    - Otherwise:
        - [`thread_alloc_and_run()`](/posts/optee-threads/#thread-alloc-and-run)
- {{< anchor id="thread-alloc-and-run" >}}`thread_alloc_and_run()`
    - Call [`__thread_alloc_and_run()`](/posts/optee-threads/#thread-alloc-and-run-internal) with `pc` parameter set to [thread_std_abi_entry()](/posts/optee-threads/#hl-7-3). So when thread is resumed, [thread_std_abi_entry()](/posts/optee-threads/#hl-7-3) will be executed.
- {{< anchor id="thread-alloc-and-run-internal" >}}`__thread_alloc_and_run()`
    - Find the free thread whose state is `THREAD_STATE_FREE`.
        - If found, set thread’s state to `THREAD_STATE_ACTIVE`.
    - Set the current thread ID (`l->curr_thread`) to the founded thread ID.
    - Call `init_regs()` to initialize the registers to be restored of the thread.
        - `thread->regs.epc` is set to `pc`.
    - Call [thread_resume()](/posts/optee-threads/#hl-6-4) to resume the thread.
- {{< anchor id="thread-resume-from-rpc" >}}`thread_resume_from_rpc()`
    - Check if the state of the thread to be resumed (indicated by `thread_id`) is `THREAD_STATE_SUSPENDED`.
        - If yes, set thread’s state to `THREAD_STATE_ACTIVE`.
            - Set the current thread ID (`l->curr_thread`)  to `thread_id`.
            - Call [thread_resume()](/posts/optee-threads/#hl-6-4) to resume the thread.
        - Otherwise, return and do nothing.

```c
// core/arch/riscv/kernel/thread_rv.S

/* void thread_resume(struct thread_ctx_regs *regs) */
FUNC thread_resume , :
	/* Disable global interrupts first */
	csrc	CSR_XSTATUS, CSR_XSTATUS_IE

	/* Restore epc */
	load_xregs a0, THREAD_CTX_REG_EPC, REG_T0
	csrw	CSR_XEPC, t0

	/* Restore ie */
	load_xregs a0, THREAD_CTX_REG_IE, REG_T0
	csrw	CSR_XIE, t0

	/* Restore status */
	load_xregs a0, THREAD_CTX_REG_STATUS, REG_T0
	csrw	CSR_XSTATUS, t0

	/* Check if previous privilege mode by status.SPP */
	b_if_prev_priv_is_u t0, 1f
	/* Set scratch as zero to indicate that we are in kernel mode */
	csrw	CSR_XSCRATCH, zero
	j	2f
1:
	/* Resume to U-mode, set scratch as tp to be used in the trap handler */
	csrw	CSR_XSCRATCH, tp
2:
	/* Restore all general-purpose registers */
	load_xregs a0, THREAD_CTX_REG_RA, REG_RA, REG_TP
	load_xregs a0, THREAD_CTX_REG_T0, REG_T0, REG_T2
	load_xregs a0, THREAD_CTX_REG_S0, REG_S0, REG_S1
	load_xregs a0, THREAD_CTX_REG_S2, REG_S2, REG_S11
	load_xregs a0, THREAD_CTX_REG_T3, REG_T3, REG_T6
	load_xregs a0, THREAD_CTX_REG_A0, REG_A0, REG_A7

	XRET
END_FUNC thread_resume
```

---

```c
// core/arch/riscv/kernel/thread_optee_abi_rv.S

FUNC thread_std_abi_entry , :
	jal	[__thread_std_abi_entry](/posts/optee-threads/)

	/* Save return value */
	mv	s0, a0

	/* Disable all interrupts */
	csrw	CSR_XIE, x0

	/* Switch to temporary stack */
	jal	thread_get_tmp_sp
	mv	sp, a0

	/*
	 * We are returning from thread_alloc_and_run()
	 * set thread state as free
	 */
	// Set:
	//   current thread's state to THREAD_STATE_FREE.
	//   l->curr_thread to THREAD_ID_INVALID.
	jal	thread_state_free

	/* Restore __thread_std_abi_entry() return value */
	mv	a1, s0
	li	a2, 0
	li	a3, 0
	li	a4, 0
	li	a0, TEEABI_OPTEED_RETURN_CALL_DONE
	mv	a5, zero

	/* Return to untrusted domain */
	jal	thread_return_to_udomain
END_FUNC thread_std_abi_entry
```

- `__thread_std_abi_entry()`
    - Call [`std_abi_entry()`](/posts/optee-abi/#std-abi-entry)

---

- `thread_handle_fast_abi()`
    - [`tee_entry_fast()`](/posts/optee-threads/#tee-entry-fast)
- {{< anchor id="tee-entry-fast" >}}`tee_entry_fast()`
    - [`__tee_entry_fast()`](/posts/optee-threads/#tee-entry-fast-internal)
- {{< anchor id="tee-entry-fast-internal" >}}`__tee_entry_fast()`
    - If `args->a0`:
        - `OPTEE_ABI_CALLS_COUNT`:
            - `tee_entry_get_api_call_count()`
        - `OPTEE_ABI_CALLS_UID`:
            - `tee_entry_get_api_uuid()`
        - …

---

- {{< anchor id="thread-enter-user-mode" >}}`thread_enter_user_mode()`
    - Disable all interrupts.
    - Call `xstatus_for_xret()` to get `xstatus` with `xstatus.PIE` set to `1`, `xstatus.PP` set to `U-mode`.
        - The returned `xstatus` will be saved along with `$a0`, `$a1`, `$a2`, `$a3`, `user_sp`, `entry_func`, and `$xie` to current thread’s reg context (`struct thread_ctx_regs`).
    - Call `__thread_enter_user_mode()` to switch to U-mode. `entry_func` is set to `thread_ctx_regs.ra` and then set to `$xepc` so that when `xret` is called to return to U-mode, `entry_func` will be executed.
    - Re-enable original interrupts.
- `thread_state_suspend()`
    - Current thread's context (`struct thread_ctx`) will be updated with:
        - `flags |= THREAD_FLAGS_COPY_ARGS_ON_RETURN`
        - `regs.status = xstatus to return`
        - `regs.epc = [.thread_rpc_return](/posts/optee-rpc/#hl-0-84)` (defined within [`thread_rpc_xstatus()`](/posts/optee-rpc/#hl-0-7))
            - So when the thread is returned from RPC by [`thread_resume_from_rpc()`](/posts/optee-threads/#thread-resume-from-rpc) , [`.thread_rpc_return`](/posts/optee-rpc/#hl-0-84) will be called.
        - `state = THREAD_STATE_SUSPENDED`
        - Current thread ID (`l->curr_thread`) is set to `THREAD_ID_INVALID` to indicate no active current thread.

---

- `thread_mask_exceptions()`
    - Mask the interrupts.
- `thread_unmask_exceptions()`
    - Unmask the interrupts.
