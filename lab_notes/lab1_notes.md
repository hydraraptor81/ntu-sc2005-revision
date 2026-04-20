# Lab 1: System Calls & Processes

## 1. xv6 Basics and Process Execution
xv6 is a Unix-like teaching operating system that uses the RISC-V architecture.
The kernel provides services to user programs through a strict boundary.

* **User Space vs. Kernel Space**: Programs run in user space and alternate to
    kernel space when they invoke kernel services (system calls) using the
    `ecall` instruction.
* **Process Introspection**: Pressing `Ctrl-p` in the xv6 console prints a
    list of all running processes, their PIDs, states, and names.
* **Getting PIDs**: User programs can fetch their own Process ID using the
    `getpid()` system call (e.g., `printf("%d\n", getpid());`).

## 2. Process States & Zombies
A process goes through various states, including a "zombie" state upon exiting.

* **The Zombie State**: When a child process terminates, it cannot fully
    remove itself from the process table immediately. It remains in a `ZOMBIE`
    state until its parent acknowledges its death by calling `wait()`.
* **Zombie Demonstration (`zombie.c`)**: If a parent process (e.g., PID 15)
    `fork()`s a child (PID 16) and then `sleep()`s, the child will exit first.
    Because the parent is asleep and not calling `wait()`, the terminated child
    lingers in the process table as a zombie until the parent wakes up.
```sh
1 sleep  init
2 sleep  sh
15 sleep  zombie
16 zombie zombie
```

## 3. The `kill` Command
The `kill` command in xv6 changes a process's state to "killed", meaning it
will terminate the next time it returns to user space.

* `kill 0`: Does nothing. PID 0 is the kernel setup process and is not a
    standard running user process.
* `kill 1`: Kills the `init` process. Since `init` is the ancestor of all
    user processes and handles reaping orphaned zombies, killing it causes
    the system to crash (kernel panic).
* `kill 2`, `kill 3`: Kills standard user processes like the `sh` (shell)
    depending on the boot order.

## 4. Implementing a New System Call: `getproccount()`
To track active processes, we implemented a custom system call that iterates
through the kernel's process table. This requires modifying both user-space
and kernel-space code.

### Step 1: User-Space Declarations
1.  **`user/user.h`**: Add the prototype `int getproccount(void);` so user
    programs can compile against it.
2.  **`user/usys.pl`**: Add `entry("getproccount");`. This Perl script generates
    `usys.S`, which contains the assembly to load the system call number and
    invoke the `ecall` instruction to trap into the kernel.
    ```asm
    .global mysyscall
    mysyscall:
     li a7, SYS_mysyscall
     ecall
     ret
    ```

### Step 2: Kernel-Space Routing
1.  **`kernel/syscall.h`**: Define the unique system call number:
    `#define SYS_getproccount 22`.
2.  **`kernel/syscall.c`**:
    * Add the `extern uint64 sys_getproccount(void);` prototype.
    * Add `[SYS_getproccount] sys_getproccount` to the `syscalls` array,
        which maps the system call number to the actual kernel function.

### Step 3: Kernel-Space Implementation
1.  **`kernel/sysproc.c`**: Implement the actual logic. The kernel stores all
    processes in an array of `struct proc`. We must acquire a lock before
    checking a process's state to prevent race conditions.

    ```c
    uint64 sys_getproccount(void) {
      struct proc *p;
      int count = 0;
      for (p = proc; p < &proc[NPROC]; p++) {
        acquire(&p->lock);
        if(p->state != UNUSED) {
          count++;
        }
        release(&p->lock);
      }
      return count;
    }
    ```

### Step 4: User-Space Testing
1.  Create `user/proccount.c` to call `getproccount()` and print the result.
2.  Create `user/testproccount.c` to test the count dynamically (e.g., printing
    the count before a `fork()`, inside the child, and after the child exits).
3.  Add `$U/_proccount` and `$U/_testproccount` to the `UPROGS` list in the
    `Makefile` to ensure they are compiled into the xv6 file system.

## 5. Inter-Process Communication (Pipes)
Pipes are a fundamental IPC mechanism that provide a synchronized,
unidirectional data channel between processes using an in-memory buffer
managed by the kernel (`kernel/pipe.c`).

* **File Descriptors**: The `pipe(int p[2])` system call populates an array
    with two file descriptors. `p[0]` is the read end, and `p[1]` is the
    write end.
* **Sharing Across Processes**: When a process calls `fork()`, the child
    inherits copies of the parent's file descriptors, allowing both processes
    to interact with the exact same pipe.
* **Blocking I/O**: Reading from an empty pipe blocks the process until data
    is written. Writing to a full pipe blocks until space is freed by reading.

### Case Study: `pingpong.c`
In your `pingpong.c` implementation, communication flows one-way from the
child process to the parent process:

1.  **Setup**: The parent creates a pipe `p` and calls `fork()`.
2.  **Child (Producer)**: Loops 5 times, writing 4-byte strings ("ping" or
    "pong") into the write end `p[1]`. It then exits.
3.  **Parent (Consumer)**: Loops 5 times, calling `read(p[0], buf, 4)`. If
    the child hasn't written data yet, the parent pauses (blocks) here. Once
    data arrives, it prints the received message along with its own PID.
4.  **Cleanup**: After the loop, the parent cleans up by calling `close()`
    on both pipe ends and `wait(0)` to reap the child zombie process.

*Note: It is a best practice to close pipe file descriptors when they are no
longer needed. If write ends are left open unnecessarily, a reading process
might block forever waiting for data that will never come.*

```c
  int p[2];
  char buf[5];
  pipe(p);
  if(fork() == 0){
    // Child
    char* buf;
    for (int j=0; j<5; j++) {
      if ((j % 2) == 0) {
        buf = "ping";
        write(p[1], buf, 4);
      } else {
        buf = "pong";
        write(p[1], buf, 4);
      }
    }
    exit(0);
  } else {
    // Parent
    for (int j=0; j<5; j++) {
      read(p[0], buf, 4);
      printf("%d: received %s\n", getpid(), buf);
    }

    close(p[0]);
    close(p[1]);
    wait(0);
    exit(0);
```


## 6. Miscellaneous Trivia
* **Directory Size (`DIRSIZ`)**: The classic v6 file system has a 14-byte
    filename limit. A directory entry is 16 bytes total: 2 bytes for the
    inode number + 14 bytes for the filename.
