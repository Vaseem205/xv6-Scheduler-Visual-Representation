# XV6 RISC-V Scheduler Visualization

This repository contains my interpretation and visual representation of the XV6 RISC-V operating system's scheduler implementation. The visualizations aim to make the scheduler's functionality more accessible and easier to understand.


## How the XV6 Scheduler Works

The XV6 scheduler implements a simple round-robin scheduling algorithm. Here's how it operates:

1. **Basic Structure**:
   - Each CPU core runs its own scheduler
   - The scheduler runs in an infinite loop looking for RUNNABLE processes
   - Uses a round-robin algorithm to give each process a fair share of CPU time

2. **Key Components**:
   - Each CPU has its own scheduler thread
   - Processes are represented by `struct proc`
   - Process states include: UNUSED, USED, SLEEPING, RUNNABLE, RUNNING, ZOMBIE
   - Context switching mechanism to save and restore process state

3. **Scheduling Process**:
   - The scheduler continuously loops through the process table
   - When it finds a RUNNABLE process:
     - Acquires the process lock
     - Changes process state to RUNNING
     - Updates CPU's current process
     - Performs context switch
     - Releases the process lock

4. **Process Termination**:
   - Process runs until it either:
     - Completes its time slice
     - Voluntarily yields the CPU
     - Blocks waiting for I/O or other resources
   - Control returns to the scheduler
   - Current process pointer is cleared

## Key Files in XV6 RISC-V

When studying the scheduler, focus on these key files:

1. `kernel/proc.h`:
   - Process structure definitions
   - Process state enums
   - CPU structure definition

2. `kernel/proc.c`:
   - Scheduler implementation
   - Process management functions
   - Context switching code

3. `kernel/swtch.S`:
   - Assembly code for context switching
   - Saves and restores register states

4. `kernel/timer.c`:
   - Timer interrupt handling
   - Time slice implementation

## Implementation Details

1. **Process Structure**:
   ```c
    struct proc {
      struct spinlock lock;
    
      // p->lock must be held when using these:
      enum procstate state;        // Process state
      void *chan;                  // If non-zero, sleeping on chan
      int killed;                  // If non-zero, have been killed
      int xstate;                  // Exit status to be returned to parent's wait
      int pid;                     // Process ID
    
      // wait_lock must be held when using this:
      struct proc *parent;         // Parent process
    
      // these are private to the process, so p->lock need not be held.
      uint64 kstack;               // Virtual address of kernel stack
      uint64 sz;                   // Size of process memory (bytes)
      pagetable_t pagetable;       // User page table
      struct trapframe *trapframe; // data page for trampoline.S
      struct context context;      // swtch() here to run process
      struct file *ofile[NOFILE];  // Open files
      struct inode *cwd;           // Current directory
      char name[16];               // Process name (debugging)
    };
   ```

2. **CPU Structure**:
   ```c
    struct cpu {
      struct proc *proc;          // The process running on this cpu, or null.
      struct context context;     // swtch() here to enter scheduler().
      int noff;                   // Depth of push_off() nesting.
      int intena;                 // Were interrupts enabled before push_off()?
    };
   ```

3. **Scheduler Loop**:
   ```c
    void
    scheduler(void)
    {
      struct proc *p;
      struct cpu *c = mycpu();
    
      c->proc = 0;
      for(;;){
        // The most recent process to run may have had interrupts
        // turned off; enable them to avoid a deadlock if all
        // processes are waiting.
        intr_on();
    
        int found = 0;
        for(p = proc; p < &proc[NPROC]; p++) {
          acquire(&p->lock);
          if(p->state == RUNNABLE) {
            // Switch to chosen process.  It is the process's job
            // to release its lock and then reacquire it
            // before jumping back to us.
            p->state = RUNNING;
            c->proc = p;
            swtch(&c->context, &p->context);
    
            // Process is done running for now.
            // It should have changed its p->state before coming back.
            c->proc = 0;
            found = 1;
          }
          release(&p->lock);
        }
        if(found == 0) {
          // nothing to run; stop running on this core until an interrupt.
          intr_on();
          asm volatile("wfi");
        }
      }
    }
   ```



## References

- [XV6 RISC-V Book](https://pdos.csail.mit.edu/6.828/2020/xv6/book-riscv-rev1.pdf)
- [XV6 RISC-V Source Code](https://github.com/mit-pdos/xv6-riscv)
- [hhp3](https://youtube.com/playlist?list=PLbtzT1TYeoMhTPzyTZboW_j7TPAnjv9XB&si=FN66rIzynQ9d7H6r)

