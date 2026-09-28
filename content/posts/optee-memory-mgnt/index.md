---
date: "2024-10-08T00:00:00+08:00"
title: "OP-TEE: Memory Management"
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
// core/arch/riscv/include/mm/generic_ram_layout.h

/*
 * Generic RAM layout configuration directives
 *
 * Mandatory directives:
 * CFG_TDDRAM_START
 * CFG_TDDRAM_SIZE
 * CFG_SHMEM_START
 * CFG_SHMEM_SIZE
 *
 * Optional directives:
 * CFG_TEE_LOAD_ADDR	If defined sets TEE_LOAD_ADDR. If not, TEE_LOAD_ADDR
 *			is set by the platform or defaults to TEE_RAM_START.
 * CFG_TEE_RAM_VA_SIZE	Some platforms may have specific needs
 *
 * Optional directives when pager is enabled:
 * CFG_TDSRAM_START	If no set, emulated at CFG_TDDRAM_START
 * CFG_TDSRAM_SIZE	Default to CFG_CORE_TDSRAM_EMUL_SIZE
 *
 * Optional directive when CFG_SECURE_DATA_PATH is enabled:
 * CFG_TEE_SDP_MEM_SIZE	If CFG_TEE_SDP_MEM_BASE is not defined, SDP test
 *			memory byte size can be set by CFG_TEE_SDP_MEM_SIZE.
 *
 * This header file produces the following generic macros upon the mandatory
 * and optional configuration directives listed above:
 *
 * TEE_RAM_START	TEE core RAM physical base address
 * TEE_RAM_VA_SIZE	TEE core virtual memory address range size
 * TEE_RAM_PH_SIZE	TEE core physical RAM byte size
 * TA_RAM_START		TA contexts/pagestore RAM physical base address
 * TA_RAM_SIZE		TA contexts/pagestore RAM byte size
 * TEE_SHMEM_START	Non-secure static shared memory physical base address
 * TEE_SHMEM_SIZE	Non-secure static shared memory byte size
 *
 * TDDRAM_BASE		Main/external secure RAM base address
 * TDDRAM_SIZE		Main/external secure RAM byte size
 * TDSRAM_BASE		On-chip secure RAM base address, required by pager.
 * TDSRAM_SIZE		On-chip secure RAM byte size, required by pager.
 *
 * TEE_LOAD_ADDR	Only defined here if CFG_TEE_LOAD_ADDR is defined.
 *			Otherwise we expect the platform_config.h to define it
 *			unless which LEE_LOAD_ADDR defaults to TEE_RAM_START.
 *
 * TEE_RAM_VA_SIZE	Set to CFG_TEE_RAM_VA_SIZE or defaults to
 *			CORE_MMU_PGDIR_SIZE.
 *
 * TEE_SDP_TEST_MEM_BASE Define if a SDP memory pool is required and none set.
 *			 Always defined in the inner top (high addresses)
 *			 of CFG_TDDRAM_START/_SIZE.
 * TEE_SDP_TEST_MEM_SIZE Set to CFG_TEE_SDP_MEM_SIZE or a default size.
 *
 * ----------------------------------------------------------------------------
 * TEE RAM layout without CFG_WITH_PAGER
 *_
 *  +----------------------------------+ <-- CFG_TDDRAM_START
 *  | TEE core secure RAM (TEE_RAM)    |
 *  +----------------------------------+
 *  | Trusted Application RAM (TA_RAM) |
 *  +----------------------------------+
 *  | SDP test memory (optional)       |
 *  +----------------------------------+ <-- CFG_TDDRAM_START + CFG_TDDRAM_SIZE
 *
 *  +----------------------------------+ <-- CFG_SHMEM_START
 *  | Non-secure static SHM            |
 *  +----------------------------------+ <-- CFG_SHMEM_START + CFG_SHMEM_SIZE
 *
 * ----------------------------------------------------------------------------
 * TEE RAM layout with CFG_WITH_PAGER=y and undefined CFG_TDSRAM_START/_SIZE
 *
 *  +----------------------------------+ <-- CFG_TDDRAM_START
 *  | TEE core secure RAM (TEE_RAM)    |   | | CFG_CORE_TDSRAM_EMUL_SIZE
 *  +----------------------------------+ --|-'
 *  |   reserved (for kasan)           |   | TEE_RAM_VA_SIZE
 *  +----------------------------------+ --'
 *  | TA RAM / Pagestore (TA_RAM)      |
 *  +----------------------------------+ <---- align with CORE_MMU_PGDIR_SIZE
 *  +----------------------------------+ <--
 *  | SDP test memory (optional)       |   | CFG_TEE_SDP_MEM_SIZE
 *  +----------------------------------+ <-+ CFG_TDDRAM_START + CFG_TDDRAM_SIZE
 *
 *  +----------------------------------+ <-- CFG_SHMEM_START
 *  | Non-secure static SHM            |   |
 *  +----------------------------------+   v CFG_SHMEM_SIZE
 *
 * ----------------------------------------------------------------------------
 * TEE RAM layout with CFG_WITH_PAGER=y and define CFG_TDSRAM_START/_SIZE
 *
 *  +----------------------------------+ <-- CFG_TDSRAM_START
 *  | TEE core secure RAM (TEE_RAM)    |   | CFG_TDSRAM_SIZE
 *  +----------------------------------+ --'
 *
 *  +----------------------------------+  <- CFG_TDDRAM_START
 *  | TA RAM / Pagestore (TA_RAM)      |
 *  |----------------------------------+ <---- align with CORE_MMU_PGDIR_SIZE
 *  |----------------------------------+ <--
 *  | SDP test memory (optional)       |   | CFG_TEE_SDP_MEM_SIZE
 *  +----------------------------------+ <-+ CFG_TDDRAM_START + CFG_TDDRAM_SIZE
 *
 *  +----------------------------------+ <-- CFG_SHMEM_START
 *  | Non-secure static SHM            |   |
 *  +----------------------------------+   v CFG_SHMEM_SIZE
 */
```

---

```c
// core/arch/riscv/mm/core_mmu_arch.c

static struct mmu_pgt root_pgt[CFG_TEE_CORE_NB_CORE]
	__aligned(RISCV_PGSIZE)
	__section(".nozi.mmu.root_pgt");

static struct mmu_pgt pool_pgts[RISCV_MMU_MAX_PGTS]
	__aligned(RISCV_PGSIZE) __section(".nozi.mmu.pool_pgts");

static struct mmu_pgt user_pgts[CFG_NUM_THREADS]
	__aligned(RISCV_PGSIZE) __section(".nozi.mmu.usr_pgts");

struct mmu_partition {
	struct mmu_pgt *root_pgt;
	struct mmu_pgt *pool_pgts;
	struct mmu_pgt *user_pgts;
	unsigned int pgts_used;
	unsigned int asid;
};

static struct mmu_partition default_partition __nex_data  = {
	.root_pgt = root_pgt, // Root page tables, per-core.
	.pool_pgts = pool_pgts, // Page tables pool.
	.user_pgts = user_pgts, // User page tables, per-thread.
	.pgts_used = 0, // Increased when a pool page table is allocated.
	                // See: core_mmu_pgt_alloc().
	.asid = 0
};
```

---

```c
// include/mm/tee_mmu_types.h

struct tee_mmap_region {
	unsigned int type; /* enum teecore_memtypes */
	unsigned int region_size;
	paddr_t pa;
	vaddr_t va;
	size_t size;
	uint32_t attr; /* TEE_MATTR_* above */
};
```

```c
// include/mm/core_mmu.h

/*
 * Memory area type:
 * MEM_AREA_END:      Reserved, marks the end of a table of mapping areas.
 * MEM_AREA_TEE_RAM:  core RAM (read/write/executable, secure, reserved to TEE)
 * MEM_AREA_TEE_RAM_RX:  core private read-only/executable memory (secure)
 * MEM_AREA_TEE_RAM_RO:  core private read-only/non-executable memory (secure)
 * MEM_AREA_TEE_RAM_RW:  core private read/write/non-executable memory (secure)
 * MEM_AREA_INIT_RAM_RO: init private read-only/non-executable memory (secure)
 * MEM_AREA_INIT_RAM_RX: init private read-only/executable memory (secure)
 * MEM_AREA_NEX_RAM_RO: nexus private read-only/non-executable memory (secure)
 * MEM_AREA_NEX_RAM_RW: nexus private r/w/non-executable memory (secure)
 * MEM_AREA_TEE_COHERENT: teecore coherent RAM (secure, reserved to TEE)
 * MEM_AREA_TEE_ASAN: core address sanitizer RAM (secure, reserved to TEE)
 * MEM_AREA_IDENTITY_MAP_RX: core identity mapped r/o executable memory (secure)
 * MEM_AREA_TA_RAM:   Secure RAM where teecore loads/exec TA instances.
 * MEM_AREA_NSEC_SHM: NonSecure shared RAM between NSec and TEE.
 * MEM_AREA_NEX_NSEC_SHM: nexus non-secure shared RAM between NSec and TEE.
 * MEM_AREA_RAM_NSEC: NonSecure RAM storing data
 * MEM_AREA_RAM_SEC:  Secure RAM storing some secrets
 * MEM_AREA_ROM_SEC:  Secure read only memory storing some secrets
 * MEM_AREA_IO_NSEC:  NonSecure HW mapped registers
 * MEM_AREA_IO_SEC:   Secure HW mapped registers
 * MEM_AREA_EXT_DT:   Memory loads external device tree
 * MEM_AREA_MANIFEST_DT: Memory loads manifest device tree
 * MEM_AREA_TRANSFER_LIST: Memory area mapped for Transfer List
 * MEM_AREA_RES_VASPACE: Reserved virtual memory space
 * MEM_AREA_SHM_VASPACE: Virtual memory space for dynamic shared memory buffers
 * MEM_AREA_TS_VASPACE: TS va space, only used with phys_to_virt()
 * MEM_AREA_DDR_OVERALL: Overall DDR address range, candidate to dynamic shm.
 * MEM_AREA_SEC_RAM_OVERALL: Whole secure RAM
 * MEM_AREA_MAXTYPE:  lower invalid 'type' value
 */
enum teecore_memtypes {
	MEM_AREA_END = 0,
	MEM_AREA_TEE_RAM,
	MEM_AREA_TEE_RAM_RX,
	MEM_AREA_TEE_RAM_RO,
	MEM_AREA_TEE_RAM_RW,
	MEM_AREA_INIT_RAM_RO,
	MEM_AREA_INIT_RAM_RX,
	MEM_AREA_NEX_RAM_RO,
	MEM_AREA_NEX_RAM_RW,
	MEM_AREA_TEE_COHERENT,
	MEM_AREA_TEE_ASAN,
	MEM_AREA_IDENTITY_MAP_RX,
	MEM_AREA_TA_RAM,
	MEM_AREA_NSEC_SHM,
	MEM_AREA_NEX_NSEC_SHM,
	MEM_AREA_RAM_NSEC,
	MEM_AREA_RAM_SEC,
	MEM_AREA_ROM_SEC,
	MEM_AREA_IO_NSEC,
	MEM_AREA_IO_SEC,
	MEM_AREA_EXT_DT,
	MEM_AREA_MANIFEST_DT,
	MEM_AREA_TRANSFER_LIST,
	MEM_AREA_RES_VASPACE,
	MEM_AREA_SHM_VASPACE,
	MEM_AREA_TS_VASPACE,
	MEM_AREA_PAGER_VASPACE,
	MEM_AREA_SDP_MEM,
	MEM_AREA_DDR_OVERALL,
	MEM_AREA_SEC_RAM_OVERALL,
	MEM_AREA_MAXTYPE
};
```

```c
// mm/core_mmu.c

/* Define the platform's memory layout. */
struct memaccess_area {
	paddr_t paddr;
	size_t size;
};

#define MEMACCESS_AREA(a, s) { .paddr = a, .size = s }

// secure_only[] defines the bases and the sizes of secure memories
// used by OP-TEE.
static struct memaccess_area secure_only[] __nex_data = {
#ifdef CFG_CORE_PHYS_RELOCATABLE
	MEMACCESS_AREA(0, 0),
#else
#ifdef TRUSTED_SRAM_BASE
	MEMACCESS_AREA(TRUSTED_SRAM_BASE, TRUSTED_SRAM_SIZE),
#endif
	// e.g.
	//   TRUSTED_DRAM_BASE = TDDRAM_BASE = CFG_TDDRAM_START = 0xf1000000
	//   TRUSTED_DRAM_SIZE = TDDRAM_SIZE = CFG_TDDRAM_SIZE = 0x01000000 (16 MB)
	MEMACCESS_AREA(TRUSTED_DRAM_BASE, TRUSTED_DRAM_SIZE),
#endif
};

// nsec_shared[] defines the bases and sizes of the non-secure static
// shared memories.
static struct memaccess_area nsec_shared[] __nex_data = {
#ifdef CFG_CORE_RESERVED_SHM
	// e.g.
	//   TEE_SHMEM_START = TEE_SHMEM_START = CFG_SHMEM_START
	//   TEE_SHMEM_SIZE = TEE_SHMEM_SIZE = CFG_SHMEM_SIZE
	MEMACCESS_AREA(TEE_SHMEM_START, TEE_SHMEM_SIZE),
#endif
};
```

```c
// mm/core_mmu.c

static struct tee_mmap_region static_memory_map[CFG_MMAP_REGIONS
#if defined(CFG_CORE_ASLR) || defined(CFG_CORE_PHYS_RELOCATABLE)
						+ 1
#endif
						+ 1] __nex_bss;

.....

// _start() -> core_init_mmu_map()
/*
 * core_init_mmu_map() - init tee core default memory mapping
 *
 * This routine sets the static default TEE core mapping. If @seed is > 0
 * and configured with CFG_CORE_ASLR it will map tee core at a location
 * based on the seed and return the offset from the link address.
 *
 * If an error happened: core_init_mmu_map is expected to panic.
 *
 * Note: this function is weak just to make it possible to exclude it from
 * the unpaged area.
 */
void __weak core_init_mmu_map(unsigned long seed, struct core_mmu_config *cfg)
{
#ifndef CFG_NS_VIRTUALIZATION
	vaddr_t start = ROUNDDOWN((vaddr_t)__nozi_start, SMALL_PAGE_SIZE);
#else
	vaddr_t start = ROUNDDOWN((vaddr_t)__vcore_nex_rw_start,
				  SMALL_PAGE_SIZE);
#endif
	vaddr_t len = ROUNDUP((vaddr_t)__nozi_end, SMALL_PAGE_SIZE) - start;
	// tmp_mmap is allocated from the heap (__heap1_start or __heap2_start).
	struct tee_mmap_region *tmp_mmap = get_tmp_mmap();
	unsigned long offs = 0;

	if (IS_ENABLED(CFG_CORE_PHYS_RELOCATABLE) &&
	    (core_mmu_tee_load_pa & SMALL_PAGE_MASK))
		panic("OP-TEE load address is not page aligned");

	check_sec_nsec_mem_config();

	/*
	 * Add a entry covering the translation tables which will be
	 * involved in some virt_to_phys() and phys_to_virt() conversions.
	 */
	static_memory_map[0] = (struct tee_mmap_region){
		.type = MEM_AREA_TEE_RAM,
		.region_size = SMALL_PAGE_SIZE,
		.pa = start,
		.va = start,
		.size = len,
		.attr = core_mmu_type_to_attr(MEM_AREA_IDENTITY_MAP_RX),
	};

	COMPILE_TIME_ASSERT(CFG_MMAP_REGIONS >= 13);
	// Initalize memory maps.
	offs = init_mem_map(tmp_mmap, ARRAY_SIZE(static_memory_map), seed);

	check_mem_map(tmp_mmap);
	// Set page table entries for memory maps.
	[core_init_mmu(tmp_mmap);](/posts/optee-memory-mgnt/)
	dump_xlat_table(0x0, CORE_MMU_BASE_TABLE_LEVEL);
	// Set cfg->satp[].
	core_init_mmu_regs(cfg);
	cfg->map_offset = offs;
	// Copy the temp memory maps.
	memcpy(static_memory_map, tmp_mmap, sizeof(static_memory_map));
}
```

```c
// mm/core_mmu.c

// _start() -> core_init_mmu_map() -> init_mem_map() -> collect_mem_ranges()
// e.g.
//   D/TC:0   add_phys_mem:677 VCORE_UNPG_RX_PA type TEE_RAM_RX 0xf1000000 size 0x00092000
//   D/TC:0   add_phys_mem:677 VCORE_UNPG_RW_PA type TEE_RAM_RW 0xf1092000 size 0x0016e000
//   D/TC:0   add_phys_mem:677 ta_base type TA_RAM 0xf1200000 size 0x00e00000
//   D/TC:0   add_va_space:717 type RES_VASPACE size 0x00a00000
//   D/TC:0   add_va_space:717 type SHM_VASPACE size 0x02000000
//   D/TC:0   dump_mmap_table:849 type TEE_RAM_RX   va 0xf1000000..0xf1091fff pa 0xf1000000..0xf1091fff size 0x00092000 (smallpg)
//   D/TC:0   dump_mmap_table:849 type TEE_RAM_RW   va 0xf1092000..0xf11fffff pa 0xf1092000..0xf11fffff size 0x0016e000 (smallpg)
//   D/TC:0   dump_mmap_table:849 type RES_VASPACE  va 0xf1200000..0xf1bfffff pa 0x00000000..0x009fffff size 0x00a00000 (pgdir)
//   D/TC:0   dump_mmap_table:849 type SHM_VASPACE  va 0xf1c00000..0xf3bfffff pa 0x00000000..0x01ffffff size 0x02000000 (pgdir)
//   D/TC:0   dump_mmap_table:849 type TA_RAM       va 0xf3c00000..0xf49fffff pa 0xf1200000..0xf1ffffff size 0x00e00000 (pgdir)
static size_t collect_mem_ranges(struct tee_mmap_region *memory_map,
				 size_t num_elems)
{
	const struct core_mmu_phys_mem *mem = NULL;
	vaddr_t ram_start = secure_only[0].paddr;
	size_t last = 0;

#define ADD_PHYS_MEM(_type, _addr, _size) \
		add_phys_mem(memory_map, num_elems, #_addr, (_type), \
			     (_addr), (_size),  &last)

	if (IS_ENABLED(CFG_CORE_RWDATA_NOEXEC)) {
		ADD_PHYS_MEM(MEM_AREA_TEE_RAM_RO, ram_start,
			     VCORE_UNPG_RX_PA - ram_start);
		ADD_PHYS_MEM(MEM_AREA_TEE_RAM_RX, VCORE_UNPG_RX_PA,
			     VCORE_UNPG_RX_SZ);
		ADD_PHYS_MEM(MEM_AREA_TEE_RAM_RO, VCORE_UNPG_RO_PA,
			     VCORE_UNPG_RO_SZ);

		if (IS_ENABLED(CFG_NS_VIRTUALIZATION)) {
			ADD_PHYS_MEM(MEM_AREA_NEX_RAM_RO, VCORE_UNPG_RW_PA,
				     VCORE_UNPG_RW_SZ);
			ADD_PHYS_MEM(MEM_AREA_NEX_RAM_RW, VCORE_NEX_RW_PA,
				     VCORE_NEX_RW_SZ);
		} else {
			ADD_PHYS_MEM(MEM_AREA_TEE_RAM_RW, VCORE_UNPG_RW_PA,
				     VCORE_UNPG_RW_SZ);
		}

		if (IS_ENABLED(CFG_WITH_PAGER)) {
			ADD_PHYS_MEM(MEM_AREA_INIT_RAM_RX, VCORE_INIT_RX_PA,
				     VCORE_INIT_RX_SZ);
			ADD_PHYS_MEM(MEM_AREA_INIT_RAM_RO, VCORE_INIT_RO_PA,
				     VCORE_INIT_RO_SZ);
		}
	} else {
		ADD_PHYS_MEM(MEM_AREA_TEE_RAM, TEE_RAM_START, TEE_RAM_PH_SIZE);
	}

	if (IS_ENABLED(CFG_NS_VIRTUALIZATION)) {
		ADD_PHYS_MEM(MEM_AREA_SEC_RAM_OVERALL, TRUSTED_DRAM_BASE,
			     TRUSTED_DRAM_SIZE);
	} else {
		/*
		 * Every guest will have own TA RAM if virtualization
		 * support is enabled.
		 */
		paddr_t ta_base = 0;
		size_t ta_size = 0;

		core_mmu_get_ta_range(&ta_base, &ta_size);
		ADD_PHYS_MEM(MEM_AREA_TA_RAM, ta_base, ta_size);
	}

	if (IS_ENABLED(CFG_CORE_SANITIZE_KADDRESS) &&
	    IS_ENABLED(CFG_WITH_PAGER)) {
		/*
		 * Asan ram is part of MEM_AREA_TEE_RAM_RW when pager is
		 * disabled.
		 */
		ADD_PHYS_MEM(MEM_AREA_TEE_ASAN, ASAN_MAP_PA, ASAN_MAP_SZ);
	}

#undef ADD_PHYS_MEM

	/* Collect device memory info from SP manifest */
	if (IS_ENABLED(CFG_CORE_SEL2_SPMC))
		collect_device_mem_ranges(memory_map, num_elems, &last);

	for (mem = phys_mem_map_begin; mem < phys_mem_map_end; mem++) {
		/* Only unmapped virtual range may have a null phys addr */
		assert(mem->addr || !core_mmu_type_to_attr(mem->type));

		add_phys_mem(memory_map, num_elems, mem->name, mem->type,
			     mem->addr, mem->size, &last);
	}

	if (IS_ENABLED(CFG_SECURE_DATA_PATH))
		verify_special_mem_areas(memory_map, phys_sdp_mem_begin,
					 phys_sdp_mem_end, "SDP");

	add_va_space(memory_map, num_elems, MEM_AREA_RES_VASPACE,
		     CFG_RESERVED_VASPACE_SIZE, &last);

	add_va_space(memory_map, num_elems, MEM_AREA_SHM_VASPACE,
		     SHM_VASPACE_SIZE, &last);

	memory_map[last].type = MEM_AREA_END;

	return last;
}
```

```c
// mm/core_mmu.c

static void add_phys_mem(struct tee_mmap_region *memory_map, size_t num_elems,
			 const char *mem_name __maybe_unused,
			 enum teecore_memtypes mem_type,
			 paddr_t mem_addr, paddr_size_t mem_size, size_t *last)
{
	size_t n = 0;
	paddr_t pa;
	paddr_size_t size;

	if (!mem_size)	/* Discard null size entries */
		return;
	/*
	 * If some ranges of memory of the same type do overlap
	 * each others they are coalesced into one entry. To help this
	 * added entries are sorted by increasing physical.
	 *
	 * Note that it's valid to have the same physical memory as several
	 * different memory types, for instance the same device memory
	 * mapped as both secure and non-secure. This will probably not
	 * happen often in practice.
	 */
	DMSG("%s type %s 0x%08" PRIxPA " size 0x%08" PRIxPASZ,
	     mem_name, teecore_memtype_name(mem_type), mem_addr, mem_size);
	// Iternate the existing memory maps.
	// If there's any existing memory region overlap with the one
	// we intend to add, if both of their memory types are same,
	// merge them into a single memory region.
	// After the iteration, 'n' will be the position to be added into memory maps.
	while (true) {
		if (n >= (num_elems - 1)) {
			EMSG("Out of entries (%zu) in memory_map", num_elems);
			panic();
		}
		if (n == *last)
			break;
		pa = memory_map[n].pa;
		size = memory_map[n].size;
		// Merge the overlapped memory regions.
		if (mem_type == memory_map[n].type &&
		    ((pa <= (mem_addr + (mem_size - 1))) &&
		    (mem_addr <= (pa + (size - 1))))) {
			DMSG("Physical mem map overlaps 0x%" PRIxPA, mem_addr);
			memory_map[n].pa = MIN(pa, mem_addr);
			memory_map[n].size = MAX(size, mem_size) +
					     (pa - memory_map[n].pa);
			return;
		}
		if (mem_type < memory_map[n].type ||
		    (mem_type == memory_map[n].type && mem_addr < pa))
			break; /* found the spot where to insert this memory */
		n++;
	}

  // Insert the memory region into the memory maps.
	memmove(memory_map + n + 1, memory_map + n,
		sizeof(struct tee_mmap_region) * (*last - n));
	(*last)++;
	memset(memory_map + n, 0, sizeof(memory_map[0]));
	memory_map[n].type = mem_type;
	memory_map[n].pa = mem_addr;
	memory_map[n].size = mem_size;
}
```

```c
// mm/core_mmu.c

static void add_va_space(struct tee_mmap_region *memory_map, size_t num_elems,
			 enum teecore_memtypes type, size_t size, size_t *last)
{
	size_t n = 0;

	DMSG("type %s size 0x%08zx", teecore_memtype_name(type), size);
	// Find the position to insert the memory region into the memory maps.
	while (true) {
		if (n >= (num_elems - 1)) {
			EMSG("Out of entries (%zu) in memory_map", num_elems);
			panic();
		}
		if (n == *last)
			break;
		if (type < memory_map[n].type)
			break;
		n++;
	}

  // Insert the memory region into the memory maps without physical address.
	memmove(memory_map + n + 1, memory_map + n,
		sizeof(struct tee_mmap_region) * (*last - n));
	(*last)++;
	memset(memory_map + n, 0, sizeof(memory_map[0]));
	memory_map[n].type = type;
	memory_map[n].size = size;
}
```

---

```c
// core/arch/riscv/mm/core_mmu_arch.c

// _start() -> reset_primary() -> core_init_mmu_map() -> core_init_mmu()
// The memory maps are created in init_mem_map().
// core_init_mmu() will set the page table entries for memory maps
// on the root page table (root_pgt) by calling:
// core_init_mmu_prtn_tee() -> core_mmu_map_region().
// Page table entries are also copied to the root page tables of the
// rest of the cores in core_init_mmu_prtn_tee().
void core_init_mmu(struct tee_mmap_region *mm)
{
	uint64_t max_va = 0;
	size_t n = 0;

	static_assert((RISCV_MMU_MAX_PGTS * RISCV_MMU_PGT_SIZE) ==
			    sizeof(pool_pgts));

	/* Initialize default pagetables */
	core_init_mmu_prtn_tee(&default_partition, mm);

	for (n = 0; !core_mmap_is_end_of_table(mm + n); n++) {
		vaddr_t va_end = mm[n].va + mm[n].size - 1;

		if (va_end > max_va)
			max_va = va_end;
	}

	set_user_va_idx(&default_partition);

	core_init_mmu_prtn_ta(&default_partition);

	assert(max_va < BIT64(RISCV_MMU_VA_WIDTH));
}
```

---

```c
// core/arch/riscv/kernel/entry.S

LOCAL_DATA boot_mmu_config , : /* struct core_mmu_config */
	.skip	CORE_MMU_CONFIG_SIZE
END_DATA boot_mmu_config
```

```c
// core/arch/riscv/mm/core_mmu_arch.h

struct core_mmu_config {
	// satp[] is set to root page table (root_pgt) for each core by:
	// _start() -> reset_primary() -> core_init_mmu_map() ->
	// core_init_mmu_regs().
	// And is set to satp CSR in: set_satp() for each core.
	unsigned long satp[CFG_TEE_CORE_NB_CORE];
	uint32_t map_offset;
};
```

---

```c
// core/include/mm/tee_mm.h

struct _tee_mm_entry_t {
	struct _tee_mm_pool_t *pool;
	struct _tee_mm_entry_t *next;
	uint32_t offset;	/* offset in pages/sections */
	uint32_t size;		/* size in pages/sections */
};
typedef struct _tee_mm_entry_t tee_mm_entry_t;

struct _tee_mm_pool_t {
	tee_mm_entry_t *entry;
	paddr_t lo;		/* low boundary of the pool */
	paddr_size_t size;	/* pool size */
	uint32_t flags;		/* Config flags for the pool */
	uint8_t shift;		/* size shift */
	unsigned int lock;
#ifdef CFG_WITH_STATS
	size_t max_allocated;
#endif
};
typedef struct _tee_mm_pool_t tee_mm_pool_t;
```

```c
// core/mm/core_mmu.c

/* Physical Secure DDR pool */
tee_mm_pool_t tee_mm_sec_ddr;

/* Virtual memory pool for core mappings */
tee_mm_pool_t core_virt_mem_pool;

/* Virtual memory pool for shared memory mappings */
tee_mm_pool_t core_virt_shm_pool;
```
