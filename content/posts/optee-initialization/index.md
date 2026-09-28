---
date: "2024-10-06T00:00:00+08:00"
title: "OP-TEE: Initialization"
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

- `_start()`
    - Run a lottery to decide the primary hart.
        - Use `amoadd.w` to decide which core is the primary hart.
    - For primary hart:
        - [`reset_primary()`](https://app.notion.com/p/reset_primary-15ef7e87b76480beba43eeb22fd08d12?pvs=21)
    - For secondary hart:
        - [`reset_secondary()`](https://app.notion.com/p/reset_secondary-15ef7e87b7648037bb15c010859da54d?pvs=21)
- `reset_primary()`
    - Zero .bss section.

    - `set_tp()`
        - Set `$tp` to `thread_core_local[hartid]`.
        - Save current hart ID to `thread_core_local[hartid].hart_id`.
    - `thread_init_thread_core_local()`
        - Set `thread_core_local.curr_thread` to `THREAD_ID_INVALID` for all cores (`CFG_TEE_CORE_NB_CORE`).
        - Set `thread_core_local.flag` to `THREAD_CLF_TMP` to indicate that it’s using the temporary stack for all cores (`CFG_TEE_CORE_NB_CORE`).
        - Set first core’s `thread_core_local[0].tmp_stack_va_end` to `stack_tmp[0]`.
    - `plat_primary_init_early()`
        - Do nothing right now.
    - `console_init()`
        - In Andes’ demo, semihosting is used to print out the console.
    - [`core_init_mmu_map()`](/posts/optee-memory-mgnt/#hl-5-24)

    - `set_satp()`
        - Set `$satp` to `core_mmu_config.satp[hartid]`.
        - `core_mmu_config.satp[]` is configured in [`core_init_mmu_map()`](/posts/optee-memory-mgnt/#hl-5-24).
    - [`boot_init_primary_early()`](https://app.notion.com/p/boot_init_primary_early-11bf7e87b7648007980ae8d51aae36ad?pvs=21)
    - [`boot_init_primary_late()`](https://app.notion.com/p/boot_init_primary_late-11bf7e87b76480959fd0ed0bed772b77?pvs=21)
    - [Sync boot core and secondary cores.](/posts/optee-initialization/#sync-boot-core-and-secondary-cores)
    - `thread_clr_boot_thread()`
        - Set current thread (`l->curr_thread`)’s state to `THREAD_STATE_FREE`.
        - Set `l->curr_thread` to `THREAD_ID_INVALID`.
    - [thread_return_to_udomain()](/posts/optee-threads/#hl-2-14)
        - Before calling:
            - `$a0` is set to `TEEABI_OPTEED_RETURN_ENTRY_DONE`.
            - `$a1` is set to `thread_vector_table`.
            - `$a3` ~ `$a5` are set to **0**.
        - This will eventually set [`entry_vector_table`](/posts/opensbi-optee/) in OpenSBI.
- `reset_secondary()`
    - Wait for primary hart:
        - [Sync boot core and secondary cores.](/posts/optee-initialization/#sync-boot-core-and-secondary-cores)
    - Set `sem_cpu_sync[hartid]` to `1` to indicate that current hart is ready.
    - [`boot_init_secondary()`](https://app.notion.com/p/boot_init_secondary-11bf7e87b76480949cddf9946cce24fd?pvs=21)
- `boot_init_primary_early()`
    - `init_primary()`
        - `thread_init_core_local_stacks()`
            - Set `thread_core_local.tmp_stack_va_end` to the per-core `stack_tmp` for all cores (`CFG_TEE_CORE_NB_CORE`).
                - Temporary stack (`stack_tmp`) is used in the **non-thread** context, e.g.
                    - `interrupt_from_kernel()`
                    - `interrupt_from_user()`
                    - `thread_std_abi_entry()`
                    - `thread_rpc_xstatus()`
            - Set `thread_core_local.abt_stack_va_end` to the per-core `stack_abt` for all cores (`CFG_TEE_CORE_NB_CORE`).
                - Abort stack (`stack_abt`) is used in the **non-thread** context for exception (except for **ecall**), e.g.
                    - `exception_from_kernel()`
                    - `exception_from_user()`
        - Call `thread_set_exceptions()` with `THREAD_EXCP_ALL` to mask both native and foreign interrupts.
        - `init_runtime()`
            - Add heap section to the malloc pool.
        - `thread_init_boot_thread()`
            - `thread_init_threads()`
                - `init_thread_stacks()`
                    - Call `thread_init_stack()` to set `thread_ctx.stack_va_end` to the per-thread stack (`#ifndef CFG_WITH_PAGER`: `stack_thread`; `#else`: dynamically allocated stack) for all threads (`CFG_NUM_THREADS`).
                        - P.S. `$sp` will be set to `threads[0].stack_va_end` after `boot_init_primary_early()` is returned, before jumping to `boot_init_primary_late()`.
                - `pgt_init()`
                    - See:
                        - [https://github.com/OP-TEE/optee_os/commit/2a142248aea34179d8fd4d2f8ceba8404da03f26](https://github.com/OP-TEE/optee_os/commit/2a142248aea34179d8fd4d2f8ceba8404da03f26)
                        - [https://github.com/OP-TEE/optee_os/commit/878b4097b6c38a841cd93a385d692aba5151e7e4](https://github.com/OP-TEE/optee_os/commit/878b4097b6c38a841cd93a385d692aba5151e7e4)
            - Set `l->curr_thread` to **Thread 0**.
            - Set **Thread 0**’s state to `Active`.
        - `thread_init_primary()`
            - `thread_init_canaries()`
            - `init_user_kcode()`
                - Do nothing in RISC-V.

        - `thread_init_per_cpu()`
            - Set `mtvec`/`stvec` to [`thread_trap_vect()`](/posts/optee-interrupts/#hl-1-3).
            - Set `mscratch`/`sscratch` to `0` to indicate that the following traps are from kernel.

        - `init_sec_mon()`
            - Do nothing as RISC-V doesn't have a secure monitor.
                - Secure monitor is OpenSBI.
- `boot_init_primary_late()`
    - `init_external_dt()`
        - Initialize the external DTB located at the given address:
            1. Add MMU mapping of the external DTB.
            2. Initialize device tree overlay.
    - `discover_nsec_memory()`
        - Call `get_nsec_memory()` to find all non-secure memories from DT.
            - Lookup for the DT nodes with `device_type = “memory”`.
        - Call `core_mmu_set_discovered_nsec_ddr()` to set:
            - `discovered_nsec_ddr_start` to the first non-secure memory.
                - Non-secure memories are sorted by the physical address in ascending order.
            - `discovered_nsec_ddr_nelems` to the number of the non-secure memories.
    - `update_external_dt()`
        - Call `add_optee_dt_node()` to add `/firmware/optee` DT node.
            - `compatible = "linaro,optee-tz";`
        - Call `mark_tddram_as_reserved()` to add `/reserved-memory/optee_core` DT node.
            - Reserve the secure memory regions in DRAM used by OP-TEE (`CFG_TDDRAM_START` ~ `(CFG_TDDRAM_START + CFG_TDDRAM_SIZE - 1)`) to prevent Linux from using it.
                - If `CFG_WITH_PAGER` is set and `CFG_TDSRAM_START` is defined, **TEE core secure RAM (TEE_RAM)** is allocated in SRAM, instead of DRAM. We don’t need to reserve the memory region for it.
                    - However, we still need to reserve other secure memory regions in DRAM (**TA_RAM**) used by OP-TEE.
    - `#ifdef CFG_RISCV_S_MODE`


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
    - `boot_primary_init_intc()`
        - `plic_init()`
            - Initialize interrupt controller, e.g. PLIC.
    - `init_tee_runtime()`
        - `core_mmu_init_ta_ram()`
            - Initialize the memory region for static TAs.
                - `MEM_AREA_TA_RAM`: Secure RAM where teecore loads/exec TA instances.
        - `call_preinitcalls()`
            - Call the preinitcalls defined in `.scattered_array_preinitcall` section.
            - e.g.
                - `mobj_mapped_shm_init()`
                - … etc
        - `call_initcalls()`
            - Call the initcalls defined in `.scattered_array_initcall` section.
            - e.g.
                - `probe_dt_drivers_early()`
                - `check_ta_store()`
                - `early_ta_init()`
                - `verify_pseudo_tas_conformance()`
                - `tee_cryp_init()`
                - … etc
    - `call_finalcalls()`
        - Call the finalcalls defined in `scattered_array_call_finalcall` section.
        - e.g.
            - `release_external_dt()`
            - … etc
    - `#ifdef CFG_RISCV_S_MODE`
        - `start_secondary_cores()`
            - Call `sbi_hsm_hart_start()` to start the secondary cores.
                - Start address = `start_addr` = `_start`

---

- `boot_init_secondary()`
    - `init_secondary_helper()`
        - `boot_secondary_init_intc()`
            - `plic_hart_init()`
                - Do nothing.

---

- {{< anchor id="sync-boot-core-and-secondary-cores" >}}How to sync boot core and secondary cores during boot up (`#ifdef CFG_BOOT_SYNC_CPU`):

    ```c
    // core/arch/riscv/kernel/entry.S

    // Boot core.
    LOCAL_FUNC reset_primary , : , .identity_map

      .....

      cpu_is_ready
    	flush_cpu_semaphores
    	wait_secondary

      .....

    END_FUNC reset_primary
    ```

    ```c
    // core/arch/riscv/kernel/entry.S

    // Secondary cores.
    LOCAL_FUNC reset_secondary , : , .identity_map

    	wait_primary

    	.....

    	cpu_is_ready

    	.....

    END_FUNC reset_secondary
    ```

    ```c
    // core/arch/riscv/kernel/boot.c

    // Each core owns its own sem_cpu_sync.
    uint32_t sem_cpu_sync[CFG_TEE_CORE_NB_CORE];
    ```

    ```c
    // core/arch/riscv/kernel/entry.S

    #ifdef CFG_BOOT_SYNC_CPU
    .equ SEM_CPU_READY, 1
    #endif

    .....

    #ifdef CFG_BOOT_SYNC_CPU
    LOCAL_DATA sem_cpu_sync_start , :
    	.word	sem_cpu_sync
    END_DATA sem_cpu_sync_start

    LOCAL_DATA sem_cpu_sync_end , :
    	// Shifted by 4 bytes (uint32_t).
    	.word	sem_cpu_sync + (CFG_TEE_CORE_NB_CORE << 2)
    END_DATA sem_cpu_sync_end
    #endif
    ```

    ```c
    // core/arch/riscv/kernel/entry.S

    // Set sem_cpu_sync[hartid] to SEM_CPU_READY (1).
    .macro cpu_is_ready
    #ifdef CFG_BOOT_SYNC_CPU
      // hartid is stored in $xscratch.
    	csrr	t0, CSR_XSCRATCH
    	la	t1, sem_cpu_sync
    	slli	t0, t0, 2
    	add	t1, t1, t0
    	li	t2, SEM_CPU_READY
    	sw	t2, 0(t1)
    	fence
    #endif
    .endm
    ```

    ```c
    // core/arch/riscv/kernel/entry.S

    #ifdef CFG_BOOT_SYNC_CPU
    // Looks like it doesn't do anything useful in RISC-V... ?!
    //
    // In ARM, flush_cpu_semaphores() is defined to:
    // flush_cache_vrange(sem_cpu_sync_start, sem_cpu_sync_end),
    // which flushes sem_cpu_sync[0] ~ sem_cpu_sync[CFG_TEE_CORE_NB_CORE - 1]
    // in boot core caches so that secondary cores can read the updated semaphore
    // for the boot core.
    #define flush_cpu_semaphores \
    		la	t0, sem_cpu_sync_start
    		la	t1, sem_cpu_sync_end
    		fence
    #else
    #define flush_cpu_semaphores
    #endif
    ```

    ```c
    // core/arch/riscv/kernel/entry.S

    // Wait until sem_cpu_sync[1] ~ sem_cpu_sync[CFG_TEE_CORE_NB_CORE - 1]
    // are all SEM_CPU_READY (1).
    .macro wait_secondary
    #ifdef CFG_BOOT_SYNC_CPU
    	la	t0, sem_cpu_sync
    	li	t1, CFG_TEE_CORE_NB_CORE
    	li	t2, SEM_CPU_READY
    1:
    	addi	t1, t1, -1
    	beqz	t1, 3f
    	addi	t0, t0, 4
    2:
    	fence
    	lw	t1, 0(t0)
    	bne	t1, t2, 2b
    	j	1b
    3:
    #endif
    .endm
    ```

    ```c
    // core/arch/riscv/kernel/entry.S

    // Wait until sem_cpu_sync[0] is SEM_CPU_READY (1).
    .macro wait_primary
    #ifdef CFG_BOOT_SYNC_CPU
    	la	t0, sem_cpu_sync
    	li	t2, SEM_CPU_READY
    1:
    	fence	w, w
    	lw	t1, 0(t0)
    	bne	t1, t2, 1b
    #endif
    .endm
    ```
