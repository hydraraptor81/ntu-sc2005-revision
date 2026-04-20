# Lab 2: Scheduling & Threads

## 1. Process Abstraction and Inspection (`ps`)
In xv6, the kernel manages processes using an array of `struct proc` (the
process table). To inspect this from user space, you implemented the `ps` tool.

* **The `sys_ps` System Call**: Iterates through `proc[NPROC]`. To safely
    read process states (PID, parent PID, name, state), the kernel must first
    acquire the process lock (`acquire(&p->lock)`) to avoid race conditions.
* **Crossing the Boundary (`copyout`)**: Because the kernel and user space
    have isolated memory, the kernel uses `copyout()` to safely transfer the
    populated array of `struct procinfo` to the user-provided memory address.

```c
// kernel/proc.h - Information structure exposed to user space
struct procinfo {
  int pid;
  int state;
  int ppid;
  char name[16];
};

// kernel/sysproc.c - Implementation of sys_ps
uint64 sys_ps(void) {
  struct proc *p;
  struct procinfo info[NPROC];
  uint64 addr;
  int max, n = 0;

  argaddr(0, &addr);
  argint(1, &max);
  if(max > NPROC) max = NPROC;

  for(p = proc; p < &proc[NPROC] && n < max; p++){
    acquire(&p->lock);
    if(p->state != UNUSED){
      info[n].pid = p->pid;
      info[n].state = p->state;
      info[n].ppid = (p->parent) ? p->parent->pid : -1;
      safestrcpy(info[n].name, p->name, sizeof(info[n].name));
      n++;
    }
    release(&p->lock);
  }

  // Safely copy the populated array to user space
  if(copyout(myproc()->pagetable, addr, (char*)info,
             n*sizeof(struct procinfo)) < 0)
    return -1;

  return n;
}

```

## 2. Process Lifecycle & Execution Tracing
Your debugging patch (Patch 1) reveals the hidden sequence of how processes
are born and scheduled:

* **Allocation (`allocproc`)**: When `fork()` is called, the kernel finds an
    `UNUSED` slot. It assigns a PID, allocates a trapframe, and sets up the
    kernel stack.
* **The `forkret` Trick**: A newly created process does not immediately jump
    to its user-space `main()`. Instead, `allocproc` sets the process's return
    address (`p->context.ra`) to the kernel function `forkret()`. When the
    scheduler first switches to this process, it "returns" into `forkret`,
    which releases locks and smoothly transitions to user space via `usertrapret`.

```c
// kernel/proc.c - Inside allocproc()
found:
  p->pid = allocpid();
  p->state = USED;
  p->ctime = ticks; // Track creation time for FCFS

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;
  return p;
```

## 3. Kernel Scheduling Policies
The scheduler (`scheduler()` in `kernel/proc.c`) runs an infinite loop on
each CPU core. You implemented three distinct scheduling policies:

* **Default Round-Robin**: Preemptive. Iterates sequentially and runs each
    `RUNNABLE` process. Timer interrupts force a context switch (`yield()`).
* **Priority Scheduling (Even PIDs)**: Demonstrates priority levels. Your
    patch uses a two-pass search: it first scans for any `RUNNABLE` process
    with an even PID (`p->pid % 2 == 0`). If none exist, it falls back to
    scanning for odd PIDs.

```c
// kernel/proc.c - Even PID Scheduling
struct proc *selected = 0;
// Pass 1: prioritize even PIDs
for(p = proc; p < &proc[NPROC]; p++) {
  acquire(&p->lock);
  if(p->state == RUNNABLE && (p->pid % 2) == 0) {
    selected = p;
    break; // Found highest priority, stop searching
  }
  release(&p->lock);
}
// Pass 2: Fallback to odd PIDs if no even PIDs are runnable
if(selected == 0) {
  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == RUNNABLE && (p->pid % 2) == 1) {
      selected = p;
      break;
    }
    release(&p->lock);
  }
}
```

* **First-Come, First-Served (FCFS)**:
    * *Tracking*: You added `uint ctime` to `struct proc`, recording the
        exact tick it was created (`p->ctime = ticks` in `allocproc`).
    * *Selection*: The scheduler iterates through the table to find the
        `RUNNABLE` process with the lowest `ctime`.
    * *Nature*: This is inherently non-preemptive. A long-running process can
        monopolize the CPU, leading to the "convoy effect".
    * Makefile*: make qemu FCFS_SCHED=yes to enable FCFS

```c
// kernel/proc.c - FCFS Scheduling
struct proc *oldest = 0;
for(p = proc; p < &proc[NPROC]; p++) {
  acquire(&p->lock);
  if(p->state == RUNNABLE) {
    if(oldest == 0 || p->ctime < oldest->ctime) {
      if(oldest != 0)
        release(&oldest->lock); // release previously held oldest lock
      oldest = p;
      continue; // keep lock on new oldest and continue searching
    }
  }
  release(&p->lock);
}
if(oldest) {
  oldest->state = RUNNING;
  c->proc = oldest;
  swtch(&c->context, &oldest->context); // Context switch
  c->proc = 0;
  release(&oldest->lock);
}
```

## 4. Context Switching (`swtch.S`)
Whether switching between kernel processes or user threads, a context switch
requires saving CPU registers.

* **Callee-Saved Only**: `swtch` only saves/restores 14 callee-saved registers
    (`ra`, `sp`, `s0`-`s11`). Why? Because `swtch` is called like a standard
    C function. The C compiler automatically pushes caller-saved registers
    (like `a0`-`a7`, `t0`-`t6`) to the stack *before* invoking `swtch`.
* **Execution Transfer**: The final instruction in `swtch` is `ret`. Since the
    `ra` (return address) register was swapped, `ret` executes the instruction
    where the *new* process originally left off.

```asm
# user/uthread_switch.S
.globl thread_switch
thread_switch:
    /* Save registers into old context (a0) */
    sd ra, 0(a0)
    sd sp, 8(a0)
    sd s0, 16(a0)
    ...
    sd s11, 104(a0)

    /* Load registers from new context (a1) */
    ld ra, 0(a1)
    ld sp, 8(a1)
    ld s0, 16(a1)
    ...
    ld s11, 104(a1)

    ret    /* jump to the new ra */
```

## 5. User-Level Threading (`uthread`)
While xv6 processes have independent memory, threads share the same address
space. You implemented a cooperative user-level threading library.

* **Threads vs. Kernel**: The xv6 kernel is entirely unaware of these threads.
    It schedules the parent process, and the user-level library decides which
    thread runs inside that process.
* **The `struct thread`**: Contains its own stack (`char stack[STACK_SIZE]`)
    and a `struct context` to hold registers during a switch.

```c
// user/uthread.c
struct context {
  uint64 ra;  // Return address
  uint64 sp;  // Stack pointer
  uint64 s0;  // Saved registers (s0 - s11)
  ...
  uint64 s11;
};

struct thread {
  char stack[STACK_SIZE]; /* the thread's stack */
  int state;              /* FREE, RUNNING, RUNNABLE */
  struct context context;
};
```

* **Thread Creation Setup**:
    1. `t->context.ra = (uint64)func;` -> Sets the entry point so the thread
        starts executing the target function when first scheduled.
    2. `t->context.sp = (uint64)(t->stack + STACK_SIZE);` -> Sets the stack
        pointer to the *top* of the allocated array, as stacks grow downwards
        in memory.

```c
// user/uthread.c - Inside thread_create()
memset(&t->context, 0, sizeof(t->context));

// Set RA to the thread function so thread_switch will "return" to it
t->context.ra = (uint64)func;

// Set SP to the top of the stack (highest address)
t->context.sp = (uint64)(t->stack + STACK_SIZE);
```

* **Cooperative Multitasking**: Because there are no timer interrupts in user
    space to force a switch, threads must voluntarily hand over the CPU by
    calling `thread_yield()`.

```c
// user/uthread.c - Inside thread_schedule()
if (current_thread != next_thread) {
  next_thread->state = RUNNING;
  t = current_thread;
  current_thread = next_thread;
  // Voluntarily trigger the assembly context switch
  thread_switch(&t->context, &current_thread->context);
}
```

## 6. Debugging Threads with GDB (Transcript Analysis)
Your GDB session provides a perfect trace of how to verify that your user-level
threading library is correctly creating and switching contexts. Here is a
breakdown of exactly what is happening in the trace:

### Step 1: Loading Symbols and Setting Breakpoints
```gdb
(gdb) file user/_uthread_test
Reading symbols from user/_uthread_test...
(gdb) b uthread.c:80
Breakpoint 1 at 0xb5e: file user/uthread.c, line 80.
```
Because user programs run in isolated memory, GDB needs to load the symbol
table for `_uthread_test` to map memory addresses to your C code. You then
set a breakpoint right where the context switch happens in `thread_schedule()`.

### Step 2: Inspecting the Thread Context
After continuing (`c`) and hitting the breakpoint, you inspected the target
thread's structure:
```gdb
(gdb) p/x *next_thread
$1 = {stack = {0x0 <repeats 8192 times>}, state = 0x1,
      context = {ra = 0xa4, sp = 0x8130, s0 = 0x0, s1 = 0x0, ... s11 = 0x0}}
```
This is the most critical verification step. It proves `thread_create()` worked:
1.  **`state = 0x1`**: The thread is marked as `RUNNING` (0x1).
2.  **`ra = 0xa4`**: The Return Address is set to `0xa4`, which is the memory
    address of the thread's entry function (e.g., `thread_a`). When `ret` is
    called in `thread_switch`, it will jump here.
3.  **`sp = 0x8130`**: The Stack Pointer correctly points to the *top* of the
    allocated 8192-byte stack array.
4.  **`s0` through `s11` = `0x0`**: The callee-saved registers were properly
    zeroed out during initialization.

### Step 3: Inspecting Stack Memory
```gdb
(gdb) x/20x next_thread->stack
0x6130 <all_thread+16624>:  0x00000000  0x00000000 ...
```
The `x/20x` command examines 20 hexadecimal words of memory at the base of the
thread's stack. By repeatedly checking this on different threads (like the
subsequent `0x81a8`), you can verify that each thread is operating in its own
distinct memory region within the `all_thread` array.

### Step 4: Assembly-Level Stepping
```gdb
(gdb) b thread_switch
Breakpoint 2 at 0xc3e
(gdb) c
Thread 1 hit Breakpoint 2, 0x0000000000000c3e in thread_switch ()
=> 0x0000000000000c3e <thread_switch+0>:  00153023  sd  ra,0(a0)

(gdb) si
0x0000000000000c42 in thread_switch ()
=> 0x0000000000000c42 <thread_switch+4>:  00253423  sd  sp,8(a0)
```
You set a breakpoint directly on the assembly function `thread_switch`.
Using `si` (step instruction), you execute exactly one CPU instruction at a
time.
* **`sd ra,0(a0)`**: This stores the Return Address (`ra`) into the memory
    address held in register `a0`. In RISC-V C calling conventions, `a0` holds
    the first argument passed to the function (which is `&t->context`, the
    pointer to the old thread's context).
* **`sd sp,8(a0)`**: This stores the Stack Pointer (`sp`) at an 8-byte
    offset from `a0` (because `uint64` variables are 8 bytes long).

