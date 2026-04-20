# Lab 4: Virtual Memory & Demand Paging

## 1. Memory Inspection and Eager Allocation
In default xv6, memory is allocated eagerly. You implemented two system calls
to inspect the relationship between virtual and physical memory.

* **When is `sbrk()` invoked?**
    When a program starts, it gets a small heap. As it calls `malloc()`, the
    standard library allocates chunks. When it runs out of space, it calls
    `sbrk(n)` to ask the OS to increase the process size by `n` bytes.
    * *The Flow*: Traps into `syscall.c` -> routes to `sys_sbrk()` in
        `kernel/sysproc.c` -> retrieves `n` and calls `growproc(n)` in
        `kernel/proc.c`, updating `p->sz`.
* **Why do `countvp` and `countpp` return the same value?**
    Because xv6 uses **eager allocation**. `growproc()` immediately asks `vm.c`
    to assign actual physical RAM. It calls `kalloc()` to grab a physical page
    and immediately calls `mappages()` to wire the virtual address to the
    physical page. Thus, every virtual page is directly mapped to a physical one.

```c
// kernel/sysproc.c - Virtual Page Count
uint64 sys_countvp(void) {
  struct proc *p = myproc();
  return PGROUNDUP(p->sz) / PGSIZE;
}

// kernel/sysproc.c - Physical Page Count
uint64 sys_countpp(void) {
  struct proc *p = myproc();
  uint64 count = 0, va;
  pte_t *pte;
  for(va = 0; va < p->sz; va += PGSIZE){
    pte = walk(p->pagetable, va, 0);
    if(pte != 0 && (*pte & PTE_V) != 0) count++;
  }
  return count;
}
```

## 2. Page Faults and The Trampoline
Traps route execution into the kernel via `usertrap()` in `kernel/trap.c`.

* **Detecting Page Faults**: When a page fault occurs (Load: `scause 13`,
    Store: `scause 15`), the default xv6 trap handler catches it in an `else`
    block. It prints the trap cause (`r_scause()`), the PID (`p->pid`), the
    program counter where it failed (`sepc`), and the associated memory
    address (`stval`), before setting `p->killed = 1`.
* **The Trampoline Page Secret**: The trampoline page is mapped to the
    highest virtual address (`0x3FFFFFFFF000`) in *both* the user and kernel
    page tables.
    * *Why?* When a trap occurs, the CPU must change the root page table
        address stored in the `satp` register from the user table to the
        kernel table.
    * When the `csrw satp, t1` instruction executes, the virtual memory layout
        changes *instantaneously*. The next instruction to execute will be at
        `0x3FFFFFFFF000 + 4`.
    * If the trampoline wasn't identically mapped in both tables, the CPU
        would fetch arbitrary data or hit unmapped memory immediately after
        the swap, resulting in a fatal instruction page fault.

## 3. Implementing Demand Paging (Lazy Allocation)
Demand paging updates virtual limits without allocating physical RAM immediately.

### Step A: Intercepting `sbrk()`
Instead of eager allocation, we increase `p->sz` and mark new PTEs as lazy.
```c
// kernel/riscv.h
#define PTE_LAZY (1L << 8) // Custom Lazy allocation flag

// kernel/sysproc.c - Inside sys_sbrk()
if(n > 0){
  if(addr + n >= MAXVA) return -1;
  if(uvmlazy(p->pagetable, addr, addr + n) < 0) return -1;
  p->sz += n;
}

// kernel/vm.c - uvmlazy() implementation
int uvmlazy(pagetable_t pagetable, uint64 start, uint64 end) {
  uint64 a = PGROUNDUP(start);
  for (; a < end; a += PGSIZE){
    pte_t *pte = walk(pagetable, a, 1);
    if(pte == 0) return -1;
    *pte = PTE_LAZY; // Valid bit (PTE_V) remains 0!
  }
  return 0;
}
```

### Step B: Catching the Fault & Allocating (`trap.c`)
RAM is only allocated when a particular virtual memory address is touched.
```c
// kernel/trap.c - Inside usertrap()
} else if(r_scause() == 13 || r_scause() == 15) {
  uint64 va = r_stval(); // Virtual address that failed

  if(va > 0 && va < p->sz) {
    uint64 va_page = PGROUNDDOWN(va);
    char *pa = kalloc();

    if(pa == 0) {
      p->killed = 1;
    } else {
      memset(pa, 0, PGSIZE);
      if(mappages(p->pagetable, va_page, PGSIZE, (uint64)pa,
                  PTE_W|PTE_R|PTE_U) != 0) {
        kfree(pa);
        p->killed = 1;
      }
    }
  } else {
    p->killed = 1;
  }
}
```

### Step C: Fixing Kernel Panics (`uvmunmap` & `uvmcopy`)
We must modify `vm.c` to safely ignore pages flagged with `PTE_LAZY`.
```c
// kernel/vm.c - Inside uvmunmap() and uvmcopy()
if((*pte & PTE_V) == 0) {
  if(*pte & PTE_LAZY) {
    continue; // Safely skip this unallocated page
  } else {
    panic("uvmunmap: not mapped");
  }
}
```

## 4. Concept Summary: Why Demand Paging?
**Why does Demand Paging reduce memory usage?**
Demand paging minimizes unused heap. It simply updates the virtual limits,
leaving `PTE_V` set to 0 and `PTE_LAZY` set to 1, allocating zero physical
RAM upfront. Physical RAM is reserved only for processes which actually need
it. For example, if a program allocates a massive hash table but only writes
data into a few scattered slots, only those specific slots will trigger a
page fault and be mapped to physical memory.
