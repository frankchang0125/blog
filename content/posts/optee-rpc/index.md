---
date: "2024-12-09T00:00:00+08:00"
title: "OP-TEE: RPC"
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

- {{< anchor id="thread-rpc-cmd" >}}`thread_rpc_cmd()`
    - Call [`thread_rpc()`](/posts/optee-rpc/#thread-rpc) with `rpc_args` (`rv[THREAD_RPC_NUM_ARGS]`). `rpc_args`’s first element is set to `OPTEE_ABI_RETURN_RPC_CMD` function.
- {{< anchor id="thread-rpc" >}}`thread_rpc()`
    - Call [`__thread_rpc()`](/posts/optee-rpc/#thread-rpc-internal)
- {{< anchor id="thread-rpc-internal" >}}{{< anchor id="thread-rpc-xstatus" >}}{{< anchor id="thread-rpc-return" >}}`__thread_rpc()`
    - Call `xstatus_for_xret()` to get `xstatus` with `xstatus.PIE` set to `0`, `xstatus.PP` set to `S-mode`.
        - The returned `xstatus` is passed to [thread_rpc_xstatus()](/posts/optee-rpc/#hl-0-7).
    - Call  [thread_rpc_xstatus()](/posts/optee-rpc/#hl-0-7).

```c
// core/arch/riscv/kernel/thread_optee_abi_rv.S

/*
 * void thread_rpc_xstatus(uint32_t rv[THREAD_RPC_NUM_ARGS],
 *                         unsigned long status);
 */
FUNC thread_rpc_xstatus , :
	// Allocate stack space for 8 registers.
	 /* Use stack for temporary storage */
	addi	sp, sp, -REGOFF(8)

	/* Read xSTATUS */
	csrr	a2, CSR_XSTATUS

	/* Mask all maskable exceptions before switching to temporary stack */
	csrw	CSR_XIE, x0

	/* Save return address xSTATUS and pointer to rv */
	// $a0: rv[THREAD_RPC_NUM_ARGS]
	// $a1: xstatus to restore
	// $a2: current xstatus
	STR	a0, REGOFF(0)(sp)
	STR	a1, REGOFF(1)(sp)
	STR	s0, REGOFF(2)(sp)
	STR	ra, REGOFF(3)(sp)
	STR	a2, REGOFF(4)(sp)
#ifdef CFG_UNWIND
	addi	s0, sp, REGOFF(8)
#endif

	/* Save thread state */
	jal	thread_get_ctx_regs
	// $a0 = current thread's struct thread_ctx.

	// Restore $ra.
	LDR	ra, REGOFF(3)(sp)
	/* Save ra, sp, gp, tp, and s0~s11 */
	store_xregs a0, THREAD_CTX_REG_RA, REG_RA, REG_TP
	store_xregs a0, THREAD_CTX_REG_S0, REG_S0, REG_S1
	store_xregs a0, THREAD_CTX_REG_S2, REG_S2, REG_S11

	/* Get to tmp stack */
	jal	thread_get_tmp_sp
	// $a0 = tmp stack.

	/* Get pointer to rv */
	LDR	s1, REGOFF(0)(sp)
	// $s1 = rv[THREAD_RPC_NUM_ARGS]

	/* xSTATUS to restore */
	LDR	a1, REGOFF(1)(sp)
	// $a1 = xstatus to restore

	/* Switch to tmp stack */
	mv	sp, a0

	/* Early load rv[] into s2-s4 */
	lw	s2, 0(s1)
	lw	s3, 4(s1)
	lw	s4, 8(s1)

	li	a0, THREAD_FLAGS_COPY_ARGS_ON_RETURN
	la	a2, .thread_rpc_return

	// We are about to return to untrusted domain for RPC,
	// suspend the current thread.
	jal	[thread_state_suspend](/posts/optee-threads/)
	// $a0 = thread index before suspend.

	mv	a4, a0	/* thread index */
	mv	a1, s2	/* rv[0] */
	mv	a2, s3	/* rv[1] */
	mv	a3, s4	/* rv[2] */
	li	a0, TEEABI_OPTEED_RETURN_CALL_DONE
	mv	a5, zero

	/* Return to untrusted domain */
	// $a0: TEEABI_OPTEED_RETURN_CALL_DONE
	// $a1: rv[0], e.g. OPTEE_ABI_RETURN_RPC_CMD
	// $a2: rv[1]
	// $a3: rv[2]
	// $a4: thread index before suspend.
	jal	[thread_return_to_udomain](/posts/optee-threads/)
.thread_rpc_return:
	/*
	 * Jumps here from thread_resume() above when RPC has returned.
	 * At this point has the stack pointer been restored to the value
	 * stored in THREAD_CTX above.
	 */

	/* Get pointer to rv[] */
	LDR	a4, REGOFF(0)(sp)

	/* Store a0-a3 into rv[] */
	sw	a0, 0(a4)
	sw	a1, 4(a4)
	sw	a2, 8(a4)
	sw	a3, 12(a4)

	/* Pop saved XSTATUS from stack */
	LDR	s0, REGOFF(4)(sp)
	csrw	CSR_XSTATUS, s0

	/* Pop s0 from stack */
	LDR	s0, REGOFF(2)(sp)

	addi	sp, sp, REGOFF(8)
	ret
END_FUNC thread_rpc_xstatus
DECLARE_KEEP_PAGER thread_rpc_xstatus
```
