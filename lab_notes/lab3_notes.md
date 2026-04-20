# Lab 3: Synchronization

## 1. Synchronization Primitives in xv6
To prevent data inconsistency when multiple processes access shared resources,
xv6 provides two primary synchronization mechanisms.

* **Spinlocks (`spinlock.c`)**: Busy-waiting locks that burn CPU cycles
    looping until a lock is acquired.
    * *Tradeoff*: Efficient for very short waits (e.g., 10 CPU cycles)
        because it avoids the heavy overhead of context switching.
* **Sleep and Wakeup (`proc.c`)**: Yields the CPU. The process enters a
    `SLEEPING` state and is added to a wait channel.
    * *Tradeoff*: Essential for long waits (e.g., 10,000 cycles, like waiting
        for disk I/O) where spinning would waste massive amounts of CPU time.

## 2. Spinlock Internals & Hardware Mechanics (Race Conditions)
Spinlocks rely on special hardware instructions to guarantee mutual exclusion.

* **The Non-Atomic Swap Danger**: If `acquire` is implemented using standard
    C reads and writes, two CPUs could encounter a race condition. Both CPUs
    might simultaneously read `lk->locked == 0`. Both would then write `1` to
    it, and both would enter the critical section. This violates mutual
    exclusion.
* **Consequence of Violation**: If mutual exclusion fails, critical kernel
    invariants are destroyed. For example, the kernel free list (`kalloc.c`)
    could be completely corrupted if two CPUs try to allocate or free memory
    at the exact same time.
* **Memory Barriers (`__sync_synchronize`)**: CPUs and compilers often
    reorder instructions for performance. A memory barrier (like RISC-V's
    `fence` instruction) is required in `acquire` and `release`. Without it,
    load and store operations inside the critical section might not complete
    before the lock is released. This results in incorrect program execution
    because the next CPU to acquire the lock might see stale data instead of
    the updated data.

## 3. Implementing a Blocking Semaphore
You implemented a blocking semaphore library (`kernel/semaphore.c`) using
xv6's spinlocks and sleep/wakeup primitives.

### Lock Hierarchy
* `sem_table_lock`: A global lock protecting the allocation array
    (preventing two processes from allocating the same semaphore ID).
* `sem->lock`: A per-semaphore lock protecting its specific `value`.

### `sem_wait` (Decrement & Block)
* **Crucial Detail**: We use a `while(sem->value == 0)` loop instead of an
    `if` statement to handle "spurious wakeups" or the "thundering herd"
    problem (where multiple processes wake up, but only one gets the resource).
* **Sleep Atomicity**: `sleep` requires you to hold `sem->lock`. It atomically
    puts the process to sleep and releases the lock, preventing a "lost
    wakeup" race condition.

```c
// kernel/semaphore.c
int sem_wait(int semid) {
  if(semid < 0 || semid >= MAX_SEMAPHORES || !semaphores[semid].in_use)
    return -1;

  struct semaphore *sem = &semaphores[semid];
  acquire(&sem->lock);

  // Sleep while semaphore value is zero.
  // Must be a while loop to re-check condition upon waking.
  while(sem->value == 0) {
    sleep(&sem->value, &sem->lock);
  }

  // Decrement semaphore value atomically
  sem->value--;
  release(&sem->lock);
  return 0;
}
```

### `sem_signal` (Increment & Wake)
* `wakeup(&sem->value)` wakes all processes sleeping on this specific channel.
    Because they wake up and try to reacquire `sem->lock`, they are serialized.

```c
// kernel/semaphore.c
int sem_signal(int semid) {
  if(semid < 0 || semid >= MAX_SEMAPHORES || !semaphores[semid].in_use)
    return -1;

  struct semaphore *sem = &semaphores[semid];
  acquire(&sem->lock);

  // Increment semaphore value atomically and notify sleeping processes
  sem->value++;
  wakeup(&sem->value);

  release(&sem->lock);
  return 0;
}
```

### `sem_getvalue` (Atomic Read)
* Even reading the value requires acquiring the lock to ensure you don't read
    a torn or intermediate state while another CPU is modifying it.

```c
// kernel/semaphore.c
int sem_getvalue(int semid) {
  if(semid < 0 || semid >= MAX_SEMAPHORES || !semaphores[semid].in_use)
    return -1;

  struct semaphore *sem = &semaphores[semid];
  acquire(&sem->lock);
  int val = sem->value;
  release(&sem->lock);
  return val;
}
```

## 4. Addressing a Race Condition
Without synchronization, two concurrent processes running `val = ucnt_get(0);`
and `ucnt_set(0, val + 1);` will overwrite each other, leading to a lost
increment because the read-modify-write cycle is not atomic.

By wrapping the critical section with the semaphore, we enforce mutual exclusion.

```c
// user/race_test.c
void do_work(int pid, int semid) {
    for (int i = 0; i < N; i++) {
        // Enforce mutual exclusion
        sem_wait(semid);

        // Critical Section
        int val = ucnt_get(0);
        ucnt_set(0, val + 1);

        // Release lock
        sem_signal(semid);
    }
}
```

## 5. The Producer-Consumer Problem
This involves synchronizing two distinct behaviors using two semaphores.
* `sem_empty` tracks available space (initialized to 1 for a 1-slot buffer).
* `sem_full` tracks available data (initialized to 0).

The invariant maintained here is: `sem_empty + sem_full == 1` (Buffer Capacity).

```c
// user/race_test.c
struct buf_sem {
    int sem_empty; // track empty slots, initially 1
    int sem_full;  // track full slots, initially 0
};

// CONSUMER: Waits for data, consumes it, signals space is free.
void consumer(struct buf_sem b, int loops, int valid[]) {
    char tmp;
    for(int i = 0; i < loops; i++) {
        sem_wait(b.sem_full);      // Wait until buffer has data
        tmp = ubuf_read();         // Read data
        sem_signal(b.sem_empty);   // Signal that slot is now empty
        T_ASSERT(valid[(unsigned char)tmp]);
    }
}

// PRODUCER: Waits for free space, writes data, signals data is available.
void producer(const char* msg, struct buf_sem b) {
    for (const char* p = msg; *p != '\0'; p++) {
        sem_wait(b.sem_empty);     // Wait until buffer has an empty slot
        ubuf_write(*p);            // Write data
        sem_signal(b.sem_full);    // Signal that slot is now full
    }
}

// SETUP & CLEANUP
void producer_consumer() {
    struct buf_sem b;
    b.sem_empty = sem_init(1); // 1 empty slot
    b.sem_full =  sem_init(0); // 0 full slots

    // ... fork processes and execute ...

    sem_free(b.sem_empty);
    sem_free(b.sem_full);
}
```

## 6. Critical Panic Scenarios (Exam / MCQ Focus)
1.  **Sleeping without a lock:** If you call `sleep(&sem->value, &sem->lock)`
    without actually holding `sem->lock`, xv6 will panic. Sleep explicitly
    requires the lock to be held so it can atomically release it as the
    process goes to sleep, preventing missed wakeups.
2.  **Deadlock in Wakeup:** If you call `wakeup()` while holding a process
    lock (`p->lock`), you risk a deadlock because `wakeup()` needs to iterate
    through the process table and acquire process locks to wake sleepers.

raw notes (OLD)
if non atomic swap occurs in acquire/release,
two cpus could encounter a race condition where both tries to acquire a lock
when lk->locked=0, writing 1 to it, and thus allowing both processes to enter
the critical section thus violating mutex.

kernel free list could be corrupted if two cpus tries to free memory at once
as the invariant

without a memory barrier like sync_synchronize, such that that load and store
operations in the critical section are not complete, resulting in incorrect
program execution as the updated data from the critical section is not seen
by the next instruction dependent on the critical section
