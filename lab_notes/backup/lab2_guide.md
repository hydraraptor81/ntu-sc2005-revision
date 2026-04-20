**OS Lab Study Guide: Process Management, Scheduling & User-Level Threading in xv6**

This guide synthesizes the core operating system concepts demonstrated across the lab exercises, patch implementations, and debugging sessions.

---

## 1. Process Management & Lifecycle

### Key Concepts
- **Process Abstraction**: xv6 uses `struct proc` (defined in `kernel/proc.h`) to represent each process, maintaining state including PID, parent pointer, address space, and execution context g.
- **Process States**: UNUSED → USED → RUNNABLE → RUNNING → SLEEPING/ZOMBIE. The kernel maintains a process table (`proc[NPROC]`) as an array of these structures g.

### Process Creation Flow
The `fork()` system call demonstrates the complete process lifecycle:

1. **Allocation**: `allocproc()` scans the process table for an UNUSED slot, assigns a PID via `allocpid()`, and initializes the kernel stack and trapframe g.
2. **Context Setup**: The new process's context is initialized with `p->context.ra = (uint64)forkret`, ensuring the first schedule resumes at `forkret()` g.
3. **Memory Duplication**: `uvmcopy()` duplicates the parent's page table and memory into the child g.
4. **Scheduler Integration**: The child state transitions to RUNNABLE, making it eligible for selection by the scheduler g.

**Debug Insight**: The first time a process runs, it enters via `forkret()`, not the entry point of its code. This function releases the process lock and returns to user space g.

---

## 2. Scheduling Algorithms

xv6's default scheduler is **preemptive round-robin**. The lab implements two alternatives:

### First-Come, First-Served (FCFS)
- **Mechanism**: Selects the RUNNABLE process with the earliest `ctime` (creation time) g.
- **Implementation**: Track creation time in `allocproc()` using `p->ctime = ticks`, then iterate through the process table to find the minimum g.
- **Characteristics**: Non-preemptive; once a process starts, it runs until completion or voluntary yield.

### Priority Scheduling (Even PIDs)
- **Mechanism**: Two-pass search—first scan for even PIDs, then scan for odd PIDs if no even process is runnable g.
- **Implication**: Demonstrates how priority inversion and starvation can occur if higher-priority processes monopolize the CPU.

### Common Scheduler Questions

**Q: Why does `swtch` only save/restore callee-saved registers?**
A: Caller-saved registers (temporary registers) are already preserved by the C compiler on the stack when calling functions. Only callee-saved registers (s0-s11, ra, sp) must be explicitly saved in the context structure because the ABI guarantees these survive function calls g.

**Q: What happens during `swtch(&c->context, &p->context)`?**
A: The scheduler (running on the CPU's context) swaps out its registers and restores the process's registers. When `swtch` returns, it returns on the new process's stack, effectively "becoming" that process g.

---

## 3. Context Switching Architecture

### Kernel-Level Switching (`kernel/swtch.S`)
- **Trigger**: Timer interrupts (preemption) or voluntary yields (`yield()`, `sleep()`).
- **State Preservation**: Saves all callee-saved registers (ra, sp, s0-s11) in `struct context`.
- **Critical Section**: The scheduler holds `p->lock` during the switch to prevent race conditions.

### User-Level Threading
The lab implements a cooperative user-level threading library with distinct characteristics:

**Thread Structure**:
```c
struct thread {
  char stack[STACK_SIZE];
  int state;  // FREE, RUNNING, RUNNABLE
  struct context context;  // callee-saved registers
};
```
g

**Context Switch Implementation** (`user/uthread_switch.S`):
- **Save Phase**: Store ra, sp, s0-s11 into the old thread's context structure (pointed to by a0).
- **Restore Phase**: Load ra, sp, s0-s11 from the new thread's context (pointed to by a1).
- **Return**: The `ret` instruction jumps to the restored ra, resuming execution in the new thread g.

**Thread Creation**:
- Initialize context.ra to the thread function address (so `thread_switch` "returns" to it).
- Initialize context.sp to the top of the allocated stack (highest address, as stacks grow downward).
- Set state to RUNNABLE g.

---

## 4. System Calls & Kernel Interface

### Adding New System Calls (ps/pstree example)
The implementation of `ps` demonstrates the syscall plumbing:

1. **Kernel Side** (`kernel/sysproc.c`): `sys_ps()` iterates through `proc[]`, acquires locks to safely read process info, and copies data to user space using `copyout()` g.
2. **Syscall Table**: Add entry to `syscalls[]` array in `kernel/syscall.c` and define number in `kernel/syscall.h` g.
3. **User Side**: Declare in `user/user.h` and add entry point via `user/usys.pl` (which generates the assembly stub) g.

**Data Structure**:
```c
struct procinfo {
  int pid;
  int state;
  int ppid;  // -1 if no parent
  char name[16];
};
```
g

---

## 5. Debugging Techniques

### GDB for Thread Debugging
From the debug session g:

**Key Commands**:
- `file user/_uthread_test` - Load symbols for user program.
- `b uthread.c:80` - Break at thread_switch invocation.
- `p/x *next_thread` - Examine thread context (ra, sp, saved registers).
- `x/20x next_thread->stack` - Inspect stack memory contents.
- `b thread_switch` - Break at assembly context switch routine.
- `si` - Single-step through assembly instructions.

**Verification Points**:
- Check that `ra` (return address) points to the thread function on first creation (e.g., `0xa4` for thread_a) g.
- Verify `sp` points to the top of the thread's stack segment g.
- Ensure saved registers (s0-s11) are zeroed on initialization g.

---

## 6. Common Questions & Answers

**Q: What is the difference between kernel threads and user threads in this lab?**
A: The lab implements user-level threads—managed entirely in user space without kernel involvement. The kernel sees only one process; context switches between threads happen via the user-level `thread_switch`, not the kernel's `swtch`. This is cooperative multitasking (threads yield voluntarily) gg.

**Q: Why does the scheduler need to disable interrupts (`intr_off`) while scanning the process table?**
A: To prevent race conditions where a process state changes (e.g., becomes RUNNABLE via wakeup) during the scan, which could lead to inconsistent views or missed processes g.

**Q: How does `forkret()` know which process it is running for?**
A: It calls `myproc()`, which retrieves the current CPU's `c->proc` pointer that was set by the scheduler immediately before the context switch g.

**Q: What happens if `allocproc()` cannot find a free slot?**
A: It returns 0, causing `fork()` to return -1 to the parent, indicating resource exhaustion g.

**Q: Why must the scheduler hold `p->lock` when switching to a process?**
A: The lock ensures mutual exclusion on the process state. The process itself releases the lock after resuming (in `forkret` or `usertrapret`), which serializes access to the process structure between the scheduler and the running process g.

**Q: In FCFS scheduling, what happens if the oldest process is blocked (SLEEPING)?**
A: The scheduler must check `p->state == RUNNABLE`; blocked processes are skipped even if they are the oldest. The "oldest" refers to the oldest RUNNABLE process g.

**Q: Why are thread stacks allocated as arrays within `struct thread` rather than dynamically?**
A: This simplifies memory management for the user-level library. The static allocation ensures thread stacks are contiguous with their metadata, avoiding the need for a separate memory allocator g.

---

## 7. Key Files Reference

| Component | File | Purpose |
|-----------|------|---------|
| Process Table | `kernel/proc.c/h` | Process structures, fork, exit, wait, scheduler |
| Context Switch | `kernel/swtch.S` | Assembly register save/restore for kernel |
| Trap Handling | `kernel/trap.c` | Interrupt handling, syscall dispatch, timer preemption |
| Syscalls | `kernel/sysproc.c` | System call implementations (fork, ps, etc.) |
| User Threads | `user/uthread.c`, `user/uthread_switch.S` | User-level threading library |
| User Programs | `user/ps.c`, `user/pstree.c` | Process inspection utilities |

This lab provides comprehensive exposure to the mechanisms underlying process virtualization, CPU scheduling policies, and threading abstractions fundamental to operating system design.


