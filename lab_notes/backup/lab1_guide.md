# Study Guide For Quiz 1
Created at 2026-02-25 16:02:28 UTC

</details>

Based on your implementations, here is your personalized **Quiz 1 Study Guide**:

---

## 1. System Call Implementation Flow (`getproccount`)

**The 5-Step Mechanism** (critical for quiz):
1. **User Stub** (`user/usys.pl`): Generates assembly that loads syscall number into `a7` and executes `ecall`
2. **Trap Entry**: `ecall` triggers kernel trap handler, saves user registers to `trapframe`
3. **Dispatch** (`kernel/syscall.c`): Reads `p->trapframe->a7` (your case: `22`), indexes into `syscalls[]` array
4. **Handler Execution** (`kernel/sysproc.c`):
   ```c
   // Your implementation pattern:
   extern struct proc proc[NPROC];  // Process table declaration
   for (p = proc; p < &proc[NPROC]; p++) {
     acquire(&p->lock);             // MUST hold lock to read state
     if(p->state != UNUSED) count++; // Check active processes
     release(&p->lock);
   }
   ```
5. **Return**: Value stored in `p->trapframe->a0`, returns to user space

**Exam Traps:**
- **Locking**: You must `acquire(&p->lock)` before reading `p->state` (race conditions otherwise)
- **Process States**: `UNUSED` vs `USED` vs `ZOMBIE`—your code counts all non-UNUSED states
- **Return Type**: `uint64` for kernel handlers, `int` for user declarations

---

## 2. Process States & Life Cycle (from `kernel/proc.h`)

```c
enum procstate { UNUSED, USED, SLEEPING, RUNNABLE, RUNNING, ZOMBIE };
```

**Key Transitions:**
- `UNUSED` → `USED`: `allocproc()` finds free slot
- `USED` → `RUNNABLE`: `fork()` or `userinit()`
- `RUNNABLE` → `RUNNING`: `scheduler()` selects process
- `RUNNING` → `SLEEPING`: `sleep()` (waiting for I/O or `wait()`)
- `RUNNING` → `ZOMBIE`: `exit()` (awaits parent `wait()`)
- `ZOMBIE` → `UNUSED`: `wait()` cleans up via `freeproc()`

**Quiz Focus:**
- `initproc`: First user process (runs `/init`)
- Zombie processes exist until parent calls `wait()` (try `Ctrl-p` to see them)
- `kill(pid)` sets `p->killed=1`, but process dies only when returning to user space

---

## 3. Pipes & Inter-Process Communication

**Unidirectional** (`pingpong.c`):
- `pipe(p)` creates `p[0]` (read) and `p[1]` (write)
- After `fork()`, both processes have copies of file descriptors
- **Rule**: Each process closes the end it doesn't use (prevent deadlock/hangs)

**Bidirectional** (`two_pipes.c`):
- Requires **two pipes** (pipes are unidirectional by design)
- `p1`: Parent writes → Child reads
- `p2`: Child writes → Parent reads
- **Critical**: Close unused ends immediately after fork:
  ```c
  // Child
  close(p1[1]);  // Won't write to p1
  close(p2[0]);  // Won't read from p2
  ```

**File Descriptor Inheritance:**
- `fork()` copies `ofile[]` array (`filedup()` increments reference counts)
- `exec()` keeps file descriptors open (unless marked `O_CLOEXEC`, which xv6 doesn't implement)

**Pipe Closure Behavior (Exam Favorite):**
- If write-end closes: `read()` returns 0 (EOF)
- If read-end closes: `write()` triggers `SIGPIPE` (process termination in xv6)

---

## 4. Key Implementation Details from Your Code

**Process Table Traversal Pattern:**
```c
extern struct proc proc[NPROC];  // In sysproc.c
for (p = proc; p < &proc[NPROC]; p++) { ... }
```

**Syscall Registration Checklist:**
1. `#define SYS_name NUM` in `syscall.h` (yours: 22)
2. `extern uint64 sys_name(void)` in `syscall.c`
3. Add to `syscalls[]` array: `[SYS_name] sys_name`
4. Implementation in `sysproc.c` or `sysfile.c`
5. `entry("name")` in `usys.pl`
6. `int name(...)` prototype in `user.h`

**Argument Passing:**
- `argint(0, &n)`: Fetch integer from `a0`
- `argaddr(0, &p)`: Fetch pointer from `a0`
- System call number is in `a7` (handled by framework)

---

## 5. Common Quiz Questions (Predicted)

**Q: What happens if `getproccount()` didn't acquire `p->lock` before checking `p->state`?**
A: Race condition—another CPU could be modifying the state concurrently (e.g., `allocproc()` setting state to `USED`), leading to incorrect counts or torn reads.

**Q: Why does `two_pipes.c` need two pipes instead of one?**
A: A single pipe is unidirectional (one writer, one reader). Bidirectional communication requires separate channels to avoid deadlock and mixing data streams.

**Q: What is the output of `kill 1` in xv6?**
A: Panic or system halt—PID 1 is `init`, and `exit()` in init calls `panic("init exiting")` (see `kernel/proc.c`).

**Q: In `syscall()`, what happens if `num` is invalid?**
A: Prints error message to console (`unknown sys call %d`) and sets return value to `-1` in `a0`.

**Q: What is the maximum number of processes in xv6?**
A: `NPROC` (defined in `param.h`, typically 64). Your `getproccount()` iterates through this fixed-size array.

**Q: What does `fork()` return in parent vs child?**
A: Parent: child's PID; Child: 0 (set via `np->trapframe->a0 = 0` in `proc.c`)

**Q: Why must you close unused pipe ends?**
A: `read()` blocks until EOF; if the write-end remains open in the same process, EOF never occurs → deadlock.

---

## 6. File Architecture Map

| Directory | Purpose | Your Work |
|-----------|---------|-----------|
| `kernel/syscall.c` | Dispatch table & argument fetching | Added `sys_getproccount` to array |
| `kernel/sysproc.c` | Process-related syscall implementations | `sys_getproccount()` implementation |
| `kernel/proc.c` | Process management (fork, exit, wait, scheduler) | Understand `allocproc`, `exit`, `wait` |
| `kernel/proc.h` | Process structure definitions | `enum procstate`, `struct proc` |
| `user/usys.pl` | Perl script generating syscall stubs | Added `entry("getproccount")` |
| `user/user.h` | User-space function prototypes | Added `int getproccount(void)` |
| `user/proccount.c` | Your test program | - |
| `user/two_pipes.c` | Bidirectional IPC implementation | Close patterns, fork logic |

**Build System Note:** `usys.pl` generates `usys.S` during compilation. Do not edit `usys.S` directly—it is auto-generated.

---

**Final Tip:** Be prepared to trace through `fork()` → `allocproc()` → state transitions, and explain why your `getproccount()` acquires/releases locks around each process check rather than holding one global lock.


---

_Generated by [Kagi Assistant](https://kagi.com/assistant)_
