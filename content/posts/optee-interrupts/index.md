---
date: "2024-10-11T00:00:00+08:00"
title: "OP-TEE: Interrupts / Exceptions"
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

- Currently, RISC-V OP-TEE doesn’t define native interrupts. All interrupts are foreign interrupts:

    ```c
    // core/arch/riscv/include/kernel/thread_arch.h

    #define THREAD_EXCP_FOREIGN_INTR	(CSR_XIE_SIE | CSR_XIE_TIE | CSR_XIE_EIE)
    #define THREAD_EXCP_NATIVE_INTR	        (0)
    #define THREAD_EXCP_ALL			(THREAD_EXCP_FOREIGN_INTR |\
    					 THREAD_EXCP_NATIVE_INTR)
    ```

    - i.e. OP-TEE **DOES NOT** handle any interrupts.

---

```c
// arch/riscv/kernel/thread_rv.S

FUNC thread_trap_vect , :
	// mscratch/sscratch = 0: Trap is from kernel.
	// mscratch/sscratch = 1: Trap is from user.
	csrrw	tp, CSR_XSCRATCH, tp
	bnez	tp, 0f
	/* Read tp back */
	csrrw	tp, CSR_XSCRATCH, tp
	j	[trap_from_kernel](/posts/optee-interrupts/)
0:
	/* Now tp is [thread_core_local](https://app.notion.com/p/KVM_SET_USER_MEMORY_REGION-817fabdad8494e53b35403ee09ff4b8f?pvs=21) */
	j	[trap_from_user](/posts/optee-interrupts/)
thread_trap_vect_end:
END_FUNC thread_trap_vect
```

```c
// arch/riscv/kernel/thread_vs.S

LOCAL_FUNC trap_from_kernel, :
	/* Save sp, a0, a1 into temporary spaces of thread_core_local */
	store_xregs tp, THREAD_CORE_LOCAL_X0, REG_SP
	store_xregs tp, THREAD_CORE_LOCAL_X1, REG_A0, REG_A1

	csrr	a0, CSR_XCAUSE
	/* MSB of cause differentiates between interrupts and exceptions */
	// P.S. bge is signed comparison.
	bge	a0, zero, exception_from_kernel

interrupt_from_kernel:
	/* Get thread context as sp */
	get_thread_ctx sp, a0

	/* Load and save kernel sp */
	load_xregs tp, THREAD_CORE_LOCAL_X0, REG_A0
	store_xregs sp, THREAD_CTX_REG_SP, REG_A0

	/* Restore user a0, a1 which can be saved later */
	load_xregs tp, THREAD_CORE_LOCAL_X1, REG_A0, REG_A1

	/* Save all other GPRs */
	store_xregs sp, THREAD_CTX_REG_RA, REG_RA
	store_xregs sp, THREAD_CTX_REG_GP, REG_GP
	store_xregs sp, THREAD_CTX_REG_T0, REG_T0, REG_T2
	store_xregs sp, THREAD_CTX_REG_S0, REG_S0, REG_S1
	store_xregs sp, THREAD_CTX_REG_A0, REG_A0, REG_A7
	store_xregs sp, THREAD_CTX_REG_S2, REG_S2, REG_S11
	store_xregs sp, THREAD_CTX_REG_T3, REG_T3, REG_T6
	/* Save XIE */
	csrr	t0, CSR_XIE
	store_xregs sp, THREAD_CTX_REG_IE, REG_T0
	/* Mask all interrupts */
	csrw	CSR_XIE, x0
	/* Save XSTATUS */
	csrr	t0, CSR_XSTATUS
	store_xregs sp, THREAD_CTX_REG_STATUS, REG_T0
	/* Save XEPC */
	csrr	t0, CSR_XEPC
	store_xregs sp, THREAD_CTX_REG_EPC, REG_T0

	/*
	 * a0 = cause
	 * a1 = sp
	 * Call thread_interrupt_handler(cause, regs)
	 */
	csrr	a0, CSR_XCAUSE
	mv	a1, sp
	/* Load tmp_stack_va_end as current sp. */
	load_xregs tp, THREAD_CORE_LOCAL_TMP_STACK_VA_END, REG_SP
	call	[thread_interrupt_handler](/posts/optee-interrupts/)

	/* Get thread context as sp */
	get_thread_ctx sp, t0
	/* Restore XEPC */
	load_xregs sp, THREAD_CTX_REG_EPC, REG_T0
	csrw	CSR_XEPC, t0
	/* Restore XIE */
	load_xregs sp, THREAD_CTX_REG_IE, REG_T0
	csrw	CSR_XIE, t0
	/* Restore XSTATUS */
	load_xregs sp, THREAD_CTX_REG_STATUS, REG_T0
	csrw	CSR_XSTATUS, t0
	/* Set scratch as thread_core_local */
	csrw	CSR_XSCRATCH, tp
	/* Restore all GPRs */
	load_xregs sp, THREAD_CTX_REG_RA, REG_RA
	load_xregs sp, THREAD_CTX_REG_GP, REG_GP
	load_xregs sp, THREAD_CTX_REG_T0, REG_T0, REG_T2
	load_xregs sp, THREAD_CTX_REG_S0, REG_S0, REG_S1
	load_xregs sp, THREAD_CTX_REG_A0, REG_A0, REG_A7
	load_xregs sp, THREAD_CTX_REG_S2, REG_S2, REG_S11
	load_xregs sp, THREAD_CTX_REG_T3, REG_T3, REG_T6
	load_xregs sp, THREAD_CTX_REG_SP, REG_SP
	XRET

exception_from_kernel:
	/*
	 * Update core local flags.
	 * flags = (flags << THREAD_CLF_SAVED_SHIFT) | THREAD_CLF_ABORT;
	 */
	lw	a0, THREAD_CORE_LOCAL_FLAGS(tp)
	slli	a0, a0, THREAD_CLF_SAVED_SHIFT
	ori	a0, a0, THREAD_CLF_ABORT
	li	a1, (THREAD_CLF_ABORT << THREAD_CLF_SAVED_SHIFT)
	and	a1, a0, a1
	bnez	a1, sel_tmp_sp

	/* Select abort stack */
	load_xregs tp, THREAD_CORE_LOCAL_ABT_STACK_VA_END, REG_A1
	j	set_sp

sel_tmp_sp:
	/* We have an abort while using the abort stack, select tmp stack */
	load_xregs tp, THREAD_CORE_LOCAL_TMP_STACK_VA_END, REG_A1
	ori	a0, a0, THREAD_CLF_TMP	/* flags |= THREAD_CLF_TMP; */

set_sp:
	mv	sp, a1
	sw	a0, THREAD_CORE_LOCAL_FLAGS(tp)

	/*
	 * Save state on stack
	 */
	addi	sp, sp, -THREAD_ABT_REGS_SIZE

	/* Save kernel sp */
	load_xregs tp, THREAD_CORE_LOCAL_X0, REG_A0
	store_xregs sp, THREAD_ABT_REG_SP, REG_A0

	/* Restore kernel a0, a1 which can be saved later */
	load_xregs tp, THREAD_CORE_LOCAL_X1, REG_A0, REG_A1

	/* Save all other GPRs */
	store_xregs sp, THREAD_ABT_REG_RA, REG_RA
	store_xregs sp, THREAD_ABT_REG_GP, REG_GP
	store_xregs sp, THREAD_ABT_REG_TP, REG_TP
	store_xregs sp, THREAD_ABT_REG_T0, REG_T0, REG_T2
	store_xregs sp, THREAD_ABT_REG_S0, REG_S0, REG_S1
	store_xregs sp, THREAD_ABT_REG_A0, REG_A0, REG_A7
	store_xregs sp, THREAD_ABT_REG_S2, REG_S2, REG_S11
	store_xregs sp, THREAD_ABT_REG_T3, REG_T3, REG_T6
	/* Save XIE */
	csrr	t0, CSR_XIE
	store_xregs sp, THREAD_ABT_REG_IE, REG_T0
	/* Mask all interrupts */
	csrw	CSR_XIE, x0
	/* Save XSTATUS */
	csrr	t0, CSR_XSTATUS
	store_xregs sp, THREAD_ABT_REG_STATUS, REG_T0
	/* Save XEPC */
	csrr	t0, CSR_XEPC
	store_xregs sp, THREAD_ABT_REG_EPC, REG_T0
	/* Save XTVAL */
	csrr	t0, CSR_XTVAL
	store_xregs sp, THREAD_ABT_REG_TVAL, REG_T0
	/* Save XCAUSE */
	csrr	a0, CSR_XCAUSE
	store_xregs sp, THREAD_ABT_REG_CAUSE, REG_A0

	/*
	 * a0 = cause
	 * a1 = sp (struct thread_abort_regs *regs)
	 * Call abort_handler(cause, regs)
	 */
	mv	a1, sp
	call	abort_handler

	/*
	 * Restore state from stack
	 */

	/* Restore XEPC */
	load_xregs sp, THREAD_ABT_REG_EPC, REG_T0
	csrw	CSR_XEPC, t0
	/* Restore XIE */
	load_xregs sp, THREAD_ABT_REG_IE, REG_T0
	csrw	CSR_XIE, t0
	/* Restore XSTATUS */
	load_xregs sp, THREAD_ABT_REG_STATUS, REG_T0
	csrw	CSR_XSTATUS, t0
	/* Set scratch as thread_core_local */
	csrw	CSR_XSCRATCH, tp

	/* Update core local flags */
	lw	a0, THREAD_CORE_LOCAL_FLAGS(tp)
	srli	a0, a0, THREAD_CLF_SAVED_SHIFT
	sw	a0, THREAD_CORE_LOCAL_FLAGS(tp)

	/* Restore all GPRs */
	load_xregs sp, THREAD_ABT_REG_RA, REG_RA
	load_xregs sp, THREAD_ABT_REG_GP, REG_GP
	load_xregs sp, THREAD_ABT_REG_TP, REG_TP
	load_xregs sp, THREAD_ABT_REG_T0, REG_T0, REG_T2
	load_xregs sp, THREAD_ABT_REG_S0, REG_S0, REG_S1
	load_xregs sp, THREAD_ABT_REG_A0, REG_A0, REG_A7
	load_xregs sp, THREAD_ABT_REG_S2, REG_S2, REG_S11
	load_xregs sp, THREAD_ABT_REG_T3, REG_T3, REG_T6
	load_xregs sp, THREAD_ABT_REG_SP, REG_SP
	XRET
END_FUNC trap_from_kernel
```

```c
// arch/riscv/kernel/thread_rv.S

LOCAL_FUNC trap_from_user, :
	/* Save user sp, a0, a1 into temporary spaces of thread_core_local */
	store_xregs tp, THREAD_CORE_LOCAL_X0, REG_SP
	store_xregs tp, THREAD_CORE_LOCAL_X1, REG_A0, REG_A1

	csrr	a0, CSR_XCAUSE
	/* MSB of cause differentiates between interrupts and exceptions */
	bge	a0, zero, exception_from_user

interrupt_from_user:
	/* Get thread context as sp */
	get_thread_ctx sp, a0

	/* Save user sp */
	load_xregs tp, THREAD_CORE_LOCAL_X0, REG_A0
	store_xregs sp, THREAD_CTX_REG_SP, REG_A0

	/* Restore user a0, a1 which can be saved later */
	load_xregs tp, THREAD_CORE_LOCAL_X1, REG_A0, REG_A1

	/* Save user gp */
	store_xregs sp, THREAD_CTX_REG_GP, REG_GP

	/*
	 * Set the scratch register to 0 such in case of a recursive
	 * exception thread_trap_vect() knows that it is emitted from kernel.
	 */
	csrrw	gp, CSR_XSCRATCH, zero
	/* Save user tp we previously swapped into CSR_XSCRATCH */
	store_xregs sp, THREAD_CTX_REG_TP, REG_GP
	/* Set kernel gp */
.option push
.option norelax
	la	gp, __global_pointer$
.option pop
	/* Save all other GPRs */
	store_xregs sp, THREAD_CTX_REG_RA, REG_RA
	store_xregs sp, THREAD_CTX_REG_T0, REG_T0, REG_T2
	store_xregs sp, THREAD_CTX_REG_S0, REG_S0, REG_S1
	store_xregs sp, THREAD_CTX_REG_A0, REG_A0, REG_A7
	store_xregs sp, THREAD_CTX_REG_S2, REG_S2, REG_S11
	store_xregs sp, THREAD_CTX_REG_T3, REG_T3, REG_T6
	/* Save XIE */
	csrr	t0, CSR_XIE
	store_xregs sp, THREAD_CTX_REG_IE, REG_T0
	/* Mask all interrupts */
	csrw	CSR_XIE, x0
	/* Save XSTATUS */
	csrr	t0, CSR_XSTATUS
	store_xregs sp, THREAD_CTX_REG_STATUS, REG_T0
	/* Save XEPC */
	csrr	t0, CSR_XEPC
	store_xregs sp, THREAD_CTX_REG_EPC, REG_T0

	/*
	 * a0 = cause
	 * a1 = sp
	 * Call thread_interrupt_handler(cause, regs)
	 */
	csrr	a0, CSR_XCAUSE
	mv	a1, sp
	/* Load tmp_stack_va_end as current sp. */
	load_xregs tp, THREAD_CORE_LOCAL_TMP_STACK_VA_END, REG_SP
	call	[thread_interrupt_handler](/posts/optee-interrupts/)

	/* Get thread context as sp */
	get_thread_ctx sp, t0
	/* Restore XEPC */
	load_xregs sp, THREAD_CTX_REG_EPC, REG_T0
	csrw	CSR_XEPC, t0
	/* Restore XIE */
	load_xregs sp, THREAD_CTX_REG_IE, REG_T0
	csrw	CSR_XIE, t0
	/* Restore XSTATUS */
	load_xregs sp, THREAD_CTX_REG_STATUS, REG_T0
	csrw	CSR_XSTATUS, t0
	/* Set scratch as thread_core_local */
	csrw	CSR_XSCRATCH, tp
	/* Restore all GPRs */
	load_xregs sp, THREAD_CTX_REG_RA, REG_RA
	load_xregs sp, THREAD_CTX_REG_GP, REG_GP
	load_xregs sp, THREAD_CTX_REG_TP, REG_TP
	load_xregs sp, THREAD_CTX_REG_T0, REG_T0, REG_T2
	load_xregs sp, THREAD_CTX_REG_S0, REG_S0, REG_S1
	load_xregs sp, THREAD_CTX_REG_A0, REG_A0, REG_A7
	load_xregs sp, THREAD_CTX_REG_S2, REG_S2, REG_S11
	load_xregs sp, THREAD_CTX_REG_T3, REG_T3, REG_T6
	load_xregs sp, THREAD_CTX_REG_SP, REG_SP
	XRET

exception_from_user:
	/* a0 is CSR_XCAUSE */
	li	a1, CAUSE_USER_ECALL
	bne	a0, a1, abort_from_user
ecall_from_user:
	/* Load and set kernel sp from thread context */
	get_thread_ctx a0, a1
	load_xregs a0, THREAD_CTX_KERN_SP, REG_SP

	/* Now sp is kernel sp, create stack for struct thread_scall_regs */
	addi	sp, sp, -THREAD_SCALL_REGS_SIZE
	/* Save user sp */
	load_xregs tp, THREAD_CORE_LOCAL_X0, REG_A0
	store_xregs sp, THREAD_SCALL_REG_SP, REG_A0

	/* Restore user a0, a1 which can be saved later */
	load_xregs tp, THREAD_CORE_LOCAL_X1, REG_A0, REG_A1

	/* Save user gp */
	store_xregs sp, THREAD_SCALL_REG_GP, REG_GP
	/*
	 * Set the scratch register to 0 such in case of a recursive
	 * exception thread_trap_vect() knows that it is emitted from kernel.
	 */
	csrrw	gp, CSR_XSCRATCH, zero
	/* Save user tp we previously swapped into CSR_XSCRATCH */
	store_xregs sp, THREAD_SCALL_REG_TP, REG_GP
	/* Set kernel gp */
.option push
.option norelax
	la	gp, __global_pointer$
.option pop

	/* Save other caller-saved registers */
	store_xregs sp, THREAD_SCALL_REG_RA, REG_RA
	store_xregs sp, THREAD_SCALL_REG_T0, REG_T0, REG_T2
	store_xregs sp, THREAD_SCALL_REG_A0, REG_A0, REG_A7
	store_xregs sp, THREAD_SCALL_REG_T3, REG_T3, REG_T6
	/* Save XIE */
	csrr	a0, CSR_XIE
	store_xregs sp, THREAD_SCALL_REG_IE, REG_A0
	/* Mask all interrupts */
	csrw	CSR_XIE, zero
	/* Save XSTATUS */
	csrr	a0, CSR_XSTATUS
	store_xregs sp, THREAD_SCALL_REG_STATUS, REG_A0
	/* Save XEPC */
	csrr	a0, CSR_XEPC
	store_xregs sp, THREAD_SCALL_REG_EPC, REG_A0

	/*
	 * a0 = struct thread_scall_regs *regs
	 * Call thread_scall_handler(regs)
	 */
	mv	a0, sp
	call	[thread_scall_handler](/posts/optee-interrupts/)

	/*
	 * Save kernel sp we'll had at the beginning of this function.
	 * This is when this TA has called another TA because
	 * __thread_enter_user_mode() also saves the stack pointer in this
	 * field.
	 */
	get_thread_ctx a0, a1
	addi	t0, sp, THREAD_SCALL_REGS_SIZE
	store_xregs a0, THREAD_CTX_KERN_SP, REG_T0

	/*
	 * We are returning to U-Mode, on return, the program counter
	 * is set to xsepc (pc=xepc), we add 4 (size of an instruction)
	 * to continue to next instruction.
	 */
	load_xregs sp, THREAD_SCALL_REG_EPC, REG_T0
	addi	t0, t0, 4
	csrw	CSR_XEPC, t0

	/* Restore XIE */
	load_xregs sp, THREAD_SCALL_REG_IE, REG_T0
	csrw	CSR_XIE, t0
	/* Restore XSTATUS */
	load_xregs sp, THREAD_SCALL_REG_STATUS, REG_T0
	csrw	CSR_XSTATUS, t0
	/* Set scratch as thread_core_local */
	csrw	CSR_XSCRATCH, tp
	/* Restore caller-saved registers */
	load_xregs sp, THREAD_SCALL_REG_RA, REG_RA
	load_xregs sp, THREAD_SCALL_REG_GP, REG_GP
	load_xregs sp, THREAD_SCALL_REG_TP, REG_TP
	load_xregs sp, THREAD_SCALL_REG_T0, REG_T0, REG_T2
	load_xregs sp, THREAD_SCALL_REG_A0, REG_A0, REG_A7
	load_xregs sp, THREAD_SCALL_REG_T3, REG_T3, REG_T6
	load_xregs sp, THREAD_SCALL_REG_SP, REG_SP
	XRET

abort_from_user:
	/*
	 * Update core local flags
	 */
	lw	a0, THREAD_CORE_LOCAL_FLAGS(tp)
	slli	a0, a0, THREAD_CLF_SAVED_SHIFT
	ori	a0, a0, THREAD_CLF_ABORT
	sw	a0, THREAD_CORE_LOCAL_FLAGS(tp)

	/*
	 * Save state on stack
	 */

	/* Load abt_stack_va_end and set it as sp */
	load_xregs tp, THREAD_CORE_LOCAL_ABT_STACK_VA_END, REG_SP

	/* Now sp is abort sp, create stack for struct thread_abort_regs */
	addi	sp, sp, -THREAD_ABT_REGS_SIZE

	/* Save user sp */
	load_xregs tp, THREAD_CORE_LOCAL_X0, REG_A0
	store_xregs sp, THREAD_ABT_REG_SP, REG_A0

	/* Restore user a0, a1 which can be saved later */
	load_xregs tp, THREAD_CORE_LOCAL_X1, REG_A0, REG_A1

	/* Save user gp */
	store_xregs sp, THREAD_ABT_REG_GP, REG_GP

	/*
	 * Set the scratch register to 0 such in case of a recursive
	 * exception thread_trap_vect() knows that it is emitted from kernel.
	 */
	csrrw	gp, CSR_XSCRATCH, zero
	/* Save user tp we previously swapped into CSR_XSCRATCH */
	store_xregs sp, THREAD_ABT_REG_TP, REG_GP
	/* Set kernel gp */
.option push
.option norelax
	la	gp, __global_pointer$
.option pop
	/* Save all other GPRs */
	store_xregs sp, THREAD_ABT_REG_RA, REG_RA
	store_xregs sp, THREAD_ABT_REG_T0, REG_T0, REG_T2
	store_xregs sp, THREAD_ABT_REG_S0, REG_S0, REG_S1
	store_xregs sp, THREAD_ABT_REG_A0, REG_A0, REG_A7
	store_xregs sp, THREAD_ABT_REG_S2, REG_S2, REG_S11
	store_xregs sp, THREAD_ABT_REG_T3, REG_T3, REG_T6
	/* Save XIE */
	csrr	t0, CSR_XIE
	store_xregs sp, THREAD_ABT_REG_IE, REG_T0
	/* Mask all interrupts */
	csrw	CSR_XIE, x0
	/* Save XSTATUS */
	csrr	t0, CSR_XSTATUS
	store_xregs sp, THREAD_ABT_REG_STATUS, REG_T0
	/* Save XEPC */
	csrr	t0, CSR_XEPC
	store_xregs sp, THREAD_ABT_REG_EPC, REG_T0
	/* Save XTVAL */
	csrr	t0, CSR_XTVAL
	store_xregs sp, THREAD_ABT_REG_TVAL, REG_T0
	/* Save XCAUSE */
	csrr	a0, CSR_XCAUSE
	store_xregs sp, THREAD_ABT_REG_CAUSE, REG_A0

	/*
	 * a0 = cause
	 * a1 = sp (struct thread_abort_regs *regs)
	 * Call abort_handler(cause, regs)
	 */
	mv	a1, sp
	call	abort_handler

	/*
	 * Restore state from stack
	 */

	/* Restore XEPC */
	load_xregs sp, THREAD_ABT_REG_EPC, REG_T0
	csrw	CSR_XEPC, t0
	/* Restore XIE */
	load_xregs sp, THREAD_ABT_REG_IE, REG_T0
	csrw	CSR_XIE, t0
	/* Restore XSTATUS */
	load_xregs sp, THREAD_ABT_REG_STATUS, REG_T0
	csrw	CSR_XSTATUS, t0
	/* Set scratch as thread_core_local */
	csrw	CSR_XSCRATCH, tp

	/* Update core local flags */
	lw	a0, THREAD_CORE_LOCAL_FLAGS(tp)
	srli	a0, a0, THREAD_CLF_SAVED_SHIFT
	sw	a0, THREAD_CORE_LOCAL_FLAGS(tp)

	/* Restore all GPRs */
	load_xregs sp, THREAD_ABT_REG_RA, REG_RA
	load_xregs sp, THREAD_ABT_REG_GP, REG_GP
	load_xregs sp, THREAD_ABT_REG_TP, REG_TP
	load_xregs sp, THREAD_ABT_REG_T0, REG_T0, REG_T2
	load_xregs sp, THREAD_ABT_REG_S0, REG_S0, REG_S1
	load_xregs sp, THREAD_ABT_REG_A0, REG_A0, REG_A7
	load_xregs sp, THREAD_ABT_REG_S2, REG_S2, REG_S11
	load_xregs sp, THREAD_ABT_REG_T3, REG_T3, REG_T6
	load_xregs sp, THREAD_ABT_REG_SP, REG_SP
	XRET
END_FUNC trap_from_user
```

```c
// core/arch/riscv/kernel/thread_arch.c

void thread_interrupt_handler(unsigned long cause, struct thread_ctx_regs *regs)
{
    switch (cause & LONG_MAX) {
    case IRQ_XTIMER:
        [thread_foreign_interrupt_handler](/posts/optee-interrupts/)(regs);
        break;
    case IRQ_XSOFT:
        [thread_foreign_interrupt_handler](/posts/optee-interrupts/)(regs);
        break;
    case IRQ_XEXT:
        [thread_foreign_interrupt_handler](/posts/optee-interrupts/)(regs);
        break;
    default:
        thread_unhandled_trap(cause, regs);
    }
}
```

```c
// core/arch/riscv/kernel/thread_rv.S

/*
 * void thread_foreign_interrupt_handler(struct thread_ctx_regs *regs)
 */
FUNC thread_foreign_interrupt_handler , :
	/* Mask all interrupt. */
	csrw	CSR_XIE, x0

	mv	s0, a0
	/* tp = struct thread_core_local */

	/*
	 * Update core local flags
	 */
	lw	s2, THREAD_CORE_LOCAL_FLAGS(tp)
	slli	s2, s2, THREAD_CLF_SAVED_SHIFT
	ori	s2, s2, THREAD_CLF_TMP
	ori	s2, s2, THREAD_CLF_FIQ
	sw	s2, THREAD_CORE_LOCAL_FLAGS(tp)

	/*
	 * Mark current thread as suspended.
	 * a0 = THREAD_FLAGS_EXIT_ON_FOREIGN_INTR
	 * a1 = status
	 * a2 = epc
	 * thread_state_suspend(flags, status, pc)
	 */
	li	a0, THREAD_FLAGS_EXIT_ON_FOREIGN_INTR
	LDR	a1, THREAD_CTX_REG_STATUS(s0)
	LDR	a2, THREAD_CTX_REG_EPC(s0)
	call	[thread_state_suspend](/posts/optee-threads/)
	/* Now return value a0 contains suspended thread ID. */

	/* Update core local flags */
	lw	s3, THREAD_CORE_LOCAL_FLAGS(tp)
	srli	s3, s3, THREAD_CLF_SAVED_SHIFT
	ori	s3, s3, THREAD_CLF_TMP
	sw	s3, THREAD_CORE_LOCAL_FLAGS(tp)

	/* Passing thread index in a0, and prepare to return to REE. */
	mv	a4, a0
	li	a0, TEEABI_OPTEED_RETURN_CALL_DONE
	li	a1, OPTEE_ABI_RETURN_RPC_FOREIGN_INTR
	mv	a2, zero
	mv	a3, zero
	mv	a5, zero
	j	[thread_return_to_udomain](/posts/optee-threads/)
END_FUNC thread_foreign_interrupt_handler
```

```c
// core/arch/riscv/kernel/thread_arch.c

void thread_scall_handler(struct thread_scall_regs *regs)
{
    struct ts_session *sess = NULL;
    uint32_t state = 0;

    /* Enable native interrupts */
    state = thread_get_exceptions();
    thread_unmask_exceptions(state & ~THREAD_EXCP_NATIVE_INTR);

    // Do nothing in RISC-V.
    thread_user_save_vfp();

    sess = ts_get_current_session();

    /* Restore foreign interrupts which are disabled on exception entry */
    thread_restore_foreign_intr();

    assert(sess && sess->handle_scall);

    if (!sess->handle_scall(regs)) {
        setup_unwind_user_mode(regs);
        [thread_exit_user_mode](/posts/optee-interrupts/)(regs->a0, regs->a1, regs->a2,
                        regs->a3, regs->sp, regs->ra,
                        regs->status);
    }
}
```

```c
// core/arch/riscv/kernel/thread_rv.S

/*
 * void thread_exit_user_mode(unsigned long a0, unsigned long a1,
 *			       unsigned long a2, unsigned long a3,
 *			       unsigned long sp, unsigned long pc,
 *			       unsigned long status);
 */
FUNC thread_exit_user_mode , :
	/* Set kernel stack pointer */
	mv	sp, a4

	/* Set xSTATUS */
	csrw	CSR_XSTATUS, a6

	/*
	 * Zeroize xSCRATCH to indicate to thread_trap_vect()
	 * that we are executing in kernel.
	 */
	csrw	CSR_XSCRATCH, zero

	/*
	 * Mask all interrupts first. Interrupts will be unmasked after
	 * returning from __thread_enter_user_mode().
	 */
	csrw	CSR_XIE, zero

	/* Set epc as thread_unwind_user_mode() */
	csrw	CSR_XEPC, a5

	XRET
END_FUNC thread_exit_user_mode

```
