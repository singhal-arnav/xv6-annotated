# User Space: The First Process

We've come a long way! We've seen how xv6 boots, manages memory with paging, handles processes and scheduling, deals with traps and system calls, interacts with devices, and implements a complete file system. But there's one crucial piece we haven't fully explored yet: how does the system actually transition from kernel space to user space and start running user programs?

This is where it all comes together. In this post, we'll trace through the creation and execution of the very first user process. We'll see how xv6 sets up this process in a clever way that lets it reuse all the normal process management code, and we'll watch as it boots up the init program, which in turn starts the shell and brings the system to life.

This is a beautiful piece of systems design, so let's dive in!

## The Big Picture

Here's the sequence of events after the kernel initializes:

1. **Kernel initialization**: `main()` in main.c initializes all the devices and subsystems.
2. **Creating the first process**: `main()` calls `userinit()` to create the very first process.
3. **The initcode program**: This tiny assembly program makes one system call: `exec("/init")`.
4. **The init program**: A C program that sets up file descriptors and starts the shell.
5. **The shell**: Finally, we have a command prompt where users can interact with the system!

Let's trace through each step in detail.

## Kernel Initialization: main()

We covered `main()` back in the early posts, but let's refresh our memory of the relevant parts (from main.c):

```c
int
main(void)
{
  kinit1(end, P2V(4*1024*1024)); // phys page allocator
  kvmalloc();      // kernel page table
  mpinit();        // detect other processors
  lapicinit();     // interrupt controller
  seginit();       // segment descriptors
  picinit();       // another interrupt controller
  ioapicinit();    // yet another interrupt controller
  consoleinit();   // console I/O
  uartinit();      // serial port
  pinit();         // process table
  tvinit();        // trap vectors
  binit();         // buffer cache
  fileinit();      // file table
  ideinit();       // disk
  startothers();   // start other processors
  kinit2(P2V(4*1024*1024), P2V(PHYSTOP)); // must come after startothers()
  userinit();      // first user process
  mpmain();        // finish this processor's setup
}
```

After initializing absolutely everything (memory allocator, paging, interrupts, console, disk, file system, etc.), `main()` calls `userinit()` to create the first user process. Then it calls `mpmain()`, which eventually calls `scheduler()` to start running processes.

The key insight here is that `userinit()` doesn't actually *run* the first process -- it just creates it and marks it as RUNNABLE. The scheduler will pick it up and run it just like any other process!

## Creating the First Process: userinit()

Let's look at `userinit()` in proc.c:

```c
// Set up first user process.
void
userinit(void)
{
  struct proc *p;
  extern char _binary_initcode_start[], _binary_initcode_size[];

  p = allocproc();
  
  initproc = p;
  if((p->pgdir = setupkvm()) == 0)
    panic("userinit: out of memory?");
  inituvm(p->pgdir, _binary_initcode_start, (int)_binary_initcode_size);
  p->sz = PGSIZE;
  memset(p->tf, 0, sizeof(*p->tf));
  p->tf->cs = (SEG_UCODE << 3) | DPL_USER;
  p->tf->ds = (SEG_UDATA << 3) | DPL_USER;
  p->tf->es = p->tf->ds;
  p->tf->ss = p->tf->ds;
  p->tf->eflags = FL_IF;
  p->tf->esp = PGSIZE;
  p->tf->eip = 0;  // beginning of initcode.S

  safestrcpy(p->name, "initcode", sizeof(p->name));
  p->cwd = namei("/");

  // this assignment to p->state lets other cores
  // run this process. the acquire forces the above
  // writes to be visible, and the lock is also needed
  // because the assignment might not be atomic.
  acquire(&ptable.lock);
  p->state = RUNNABLE;
  release(&ptable.lock);
}
```

This function is doing a lot, so let's break it down step by step:

### Allocating a Process

```c
p = allocproc();
initproc = p;
```

First, we call `allocproc()` to allocate a proc structure and set up its kernel stack. We covered this back in the processes post -- `allocproc()` finds a free slot in the process table, allocates a kernel stack, and sets up the context so that when the process is first scheduled, it will start running in `forkret()`, which will eventually return from a trap.

We also save a pointer to this process in the global `initproc` variable. This will be used later (it's the parent of all user processes, so when processes become zombies, they get reparented to initproc).

### Setting Up the Page Table

```c
if((p->pgdir = setupkvm()) == 0)
  panic("userinit: out of memory?");
```

We call `setupkvm()` (from vm.c, which we covered in the paging posts) to create a page table for this process. `setupkvm()` sets up the kernel part of the page table -- the mappings for kernel code and data. All processes share the same kernel mappings.

### Loading the Initcode Program

```c
inituvm(p->pgdir, _binary_initcode_start, (int)_binary_initcode_size);
p->sz = PGSIZE;
```

This is the interesting part! We call `inituvm()` to load a tiny program into the process's address space. But what program? And where does it come from?

The program is called "initcode", and it's defined in initcode.S (an assembly file). During the build process, the Makefile compiles initcode.S into a binary, then uses the linker to embed that binary into the kernel as a blob of data. The symbols `_binary_initcode_start` and `_binary_initcode_size` point to this embedded binary.

So we're literally copying a tiny binary program from the kernel's data section into the first process's address space!

Let's look at `inituvm()` to see how this works:

```c
// Load the initcode into address 0 of pgdir.
// sz must be less than a page.
void
inituvm(pde_t *pgdir, char *init, uint sz)
{
  char *mem;

  if(sz >= PGSIZE)
    panic("inituvm: more than a page");
  mem = kalloc();
  memset(mem, 0, PGSIZE);
  mappages(pgdir, 0, PGSIZE, V2P(mem), PTE_W|PTE_U);
  memmove(mem, init, sz);
}
```

Simple! We allocate a physical page with `kalloc()`, map it into the process's address space at virtual address 0 with `mappages()`, and copy the initcode binary into it with `memmove()`. The PTE_U flag means this page is accessible from user mode.

The process's size is set to PGSIZE (4096 bytes), meaning the process has exactly one page of memory.

### Setting Up the Trap Frame

Now for the clever part. Remember from the traps post that when a process returns from the kernel to user space, the `trapret` code restores all the registers from the trap frame. So if we want the process to start running at a specific location with specific register values, we can just set up the trap frame to have those values!

```c
memset(p->tf, 0, sizeof(*p->tf));
p->tf->cs = (SEG_UCODE << 3) | DPL_USER;
p->tf->ds = (SEG_UDATA << 3) | DPL_USER;
p->tf->es = p->tf->ds;
p->tf->ss = p->tf->ds;
p->tf->eflags = FL_IF;
p->tf->esp = PGSIZE;
p->tf->eip = 0;  // beginning of initcode.S
```

Let's go through each field:

- **cs, ds, es, ss**: These are the segment selectors. We set them to the user code and user data segments, with DPL_USER (privilege level 3) set. This means the process will run in user mode with restricted privileges.
- **eflags**: We set FL_IF (the interrupt flag), which means interrupts are enabled. This is important -- if we didn't set this, the timer interrupt wouldn't fire and the process would never yield the CPU!
- **esp**: The stack pointer. We set it to PGSIZE (4096), which is the top of the process's one page of memory. Remember, stacks grow downward, so we start at the high end.
- **eip**: The instruction pointer. We set it to 0, which is where we loaded the initcode binary. So when the process first runs, it will start executing at address 0.

This is brilliant! We're creating a *fake* trap frame that makes it look like the process was already running in user space, got interrupted, and is now returning from the kernel. But really, this is the first time the process will run!

### Finishing Up

```c
safestrcpy(p->name, "initcode", sizeof(p->name));
p->cwd = namei("/");
```

We set the process name to "initcode" for debugging, and set its current working directory to the root directory.

```c
acquire(&ptable.lock);
p->state = RUNNABLE;
release(&ptable.lock);
```

Finally, we mark the process as RUNNABLE. The comment explains why we need the lock: we want to make sure all the previous writes are visible before we set the state (the lock provides a memory barrier), and we want the state update to be atomic.

And that's it! The first process is now ready to run. When the scheduler picks it up, it will:
1. Call `switchuvm()` to switch to the process's page table.
2. Call `swtch()` to switch to the process's kernel stack.
3. Return from `forkret()`.
4. Return from `trapret`, which restores all the registers from the trap frame.
5. Start executing at address 0 in user mode!

Pretty cool, right? Now let's see what actually happens when the process runs.

## The Initcode Program (initcode.S)

The initcode program is tiny -- it's written in assembly and it does exactly one thing: call `exec("/init")`. Let's look at it (initcode.S):

```asm
# Initial process execs /init.
# This code runs in user space.

#include "syscall.h"
#include "traps.h"


# exec(init, argv)
.globl start
start:
  pushl $argv
  pushl $init
  pushl $0  // where caller pc would be
  movl $SYS_exec, %eax
  int $T_SYSCALL

# for(;;) exit();
exit:
  movl $SYS_exit, %eax
  int $T_SYSCALL
  jmp exit

# char init[] = "/init\0";
init:
  .string "/init\0"

# char *argv[] = { init, 0 };
.p2align 2
argv:
  .long init
  .long 0
```

Let's trace through this assembly code:

### Setting Up the exec System Call

```asm
start:
  pushl $argv
  pushl $init
  pushl $0  // where caller pc would be
  movl $SYS_exec, %eax
  int $T_SYSCALL
```

This is setting up a system call to `exec()`. Remember from the system calls post that system call arguments are passed on the stack. Here's what we're pushing:

1. `$0`: A dummy return address. Normally when you call a function, the call instruction pushes the return address. Since we're making a system call directly with `int`, we have to push a dummy return address ourselves. The kernel will skip over this.
2. `$init`: The first argument to exec -- a pointer to the string "/init".
3. `$argv`: The second argument to exec -- a pointer to the argv array.

Then we put SYS_exec (the system call number) in %eax and execute `int $T_SYSCALL` to trap into the kernel. This is exactly how user programs make system calls!

The kernel will look at %eax, see SYS_exec, and call `sys_exec()`. That function will read the arguments from the stack (using `argstr()` and `argptr()`), load the /init binary from the file system, replace the current process's memory with the new program, and start executing it.

If all goes well, `exec()` never returns -- the process is now running /init!

### Handling Failure

```asm
exit:
  movl $SYS_exit, %eax
  int $T_SYSCALL
  jmp exit
```

But what if `exec()` fails? If it does, it will return -1 and execution will continue. The code falls through to the `exit` label, which makes an exit system call and loops forever. This should never happen (if /init isn't in the file system, something is seriously wrong), but the code handles it gracefully just in case.

### Data

```asm
init:
  .string "/init\0"

.p2align 2
argv:
  .long init
  .long 0
```

The string "/init" is stored right in the code, along with an argv array containing two pointers: one to the init string, and NULL to mark the end of the array.

The `.p2align 2` directive aligns the argv array to a 4-byte boundary (2^2 = 4). This is important on x86 for performance and correctness when accessing integer arrays.

### Why Not Just Load /init Directly?

You might wonder: why bother with this tiny initcode program? Why not just load /init directly into the first process?

The answer is: it's simpler this way! By using initcode to call `exec()`, we avoid having to special-case the first process. The kernel doesn't need to know how to parse ELF binaries or set up user stacks -- that's all handled by the regular `exec()` system call code. Initcode is so tiny (less than a page) that we can just embed it in the kernel as a binary blob, avoiding the need to read it from the file system.

This is a recurring theme in xv6: avoid special cases by reusing existing code paths wherever possible. It makes the code simpler and more robust.

## The Init Program (init.c)

Now let's see what happens when initcode calls `exec("/init")`. The exec system call loads the /init binary from the file system and starts running it. This is a real C program (not assembly), so let's look at it:

```c
// init: The initial user-level program

#include "types.h"
#include "stat.h"
#include "user.h"
#include "fcntl.h"

char *argv[] = { "sh", 0 };

int
main(void)
{
  int pid, wpid;

  if(open("console", O_RDWR) < 0){
    mknod("console", 1, 1);
    open("console", O_RDWR);
  }
  dup(0);  // stdout
  dup(0);  // stderr

  for(;;){
    printf(1, "init: starting sh\n");
    pid = fork();
    if(pid < 0){
      printf(1, "init: fork failed\n");
      exit();
    }
    if(pid == 0){
      exec("sh", argv);
      printf(1, "init: exec sh failed\n");
      exit();
    }
    while((wpid=wait()) >= 0 && wpid != pid)
      printf(1, "zombie!\n");
  }
}
```

This is actually pretty straightforward! Let's break it down:

### Setting Up File Descriptors

```c
if(open("console", O_RDWR) < 0){
  mknod("console", 1, 1);
  open("console", O_RDWR);
}
dup(0);  // stdout
dup(0);  // stderr
```

First, we try to open the console device. If it doesn't exist (which it won't on the first boot), we create it with `mknod()`. The arguments (1, 1) are the major and minor device numbers for the console.

Then we open the console, which gives us file descriptor 0 (stdin). By convention in Unix, file descriptor 0 is stdin, 1 is stdout, and 2 is stderr.

We call `dup(0)` twice to duplicate file descriptor 0, which gives us file descriptors 1 and 2. Now all three standard file descriptors point to the console! Any program that init (or its children) runs will inherit these file descriptors, so they'll all read from and write to the console by default.

This is why you can see output on the console when you run xv6 programs -- init set it up this way!

### Starting the Shell (and Reaping Zombies)

```c
for(;;){
  printf(1, "init: starting sh\n");
  pid = fork();
  if(pid < 0){
    printf(1, "init: fork failed\n");
    exit();
  }
  if(pid == 0){
    exec("sh", argv);
    printf(1, "init: exec sh failed\n");
    exit();
  }
  while((wpid=wait()) >= 0 && wpid != pid)
    printf(1, "zombie!\n");
}
```

Now for the main event! Init enters an infinite loop where it:

1. Prints a message ("init: starting sh").
2. Forks a child process.
3. In the child, execs the shell ("sh").
4. In the parent, waits for children to exit.

Let's trace through this carefully:

The `fork()` call creates a new process. In the child (pid == 0), we call `exec("sh", argv)`. This replaces the child process with the shell program. The `argv` array (defined at the top of the file) contains just "sh" and NULL, which means the shell doesn't get any command-line arguments.

If `exec()` succeeds, it never returns -- the child is now running the shell! The user can now type commands at the shell prompt.

Meanwhile, in the parent (init), we call `wait()` in a loop. This is interesting! The `while` condition is:

```c
while((wpid=wait()) >= 0 && wpid != pid)
  printf(1, "zombie!\n");
```

This waits for any child to exit. But we only care about the shell child (the one with pid `pid`). If `wait()` returns a different pid, it means some *other* child of init exited. Where did that child come from?

Here's the subtle point: when a process exits, if its parent has already exited, the process gets *reparented* to init. This happens in the `exit()` system call (in proc.c), which we covered in the processes post:

```c
// Pass abandoned children to init.
if(curproc->parent == initproc)
  panic("init exiting");
for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
  if(p->parent == curproc){
    p->parent = initproc;
    if(p->state == ZOMBIE)
      wakeup1(initproc);
  }
}
```

So if a process exits and its parent has already exited, that process becomes a child of init. Init's job is to reap these "orphaned" zombie processes by calling `wait()`. That's what the inner while loop is doing!

When the shell exits (which shouldn't normally happen, but could if the user exits the shell), the outer `wait()` loop condition becomes false, and we go back to the top of the outer for loop and start a new shell. So init keeps the shell running forever!

### Why Does Init Loop Forever?

You might wonder why init needs to loop forever. Why not just start the shell and exit?

If init exits, who will reap the orphaned zombies? There needs to be at least one process that never exits to serve as the ultimate parent. In Unix systems, init (process 1) serves this role.

Also, if the shell crashes or exits, the user would be stuck with no way to interact with the system. By restarting the shell automatically, init ensures that there's always a way to interact with xv6.

This is why init is sometimes called the "reaper process" -- it reaps zombie children that no one else wants!

## The Sequence in Action

Let's trace through the complete boot sequence one more time, now that we understand all the pieces:

1. **The kernel boots**: The bootloader loads the kernel, and execution starts at `entry()` in entry.S. This sets up a stack and jumps to `main()`.

2. **Kernel initialization**: `main()` initializes all the devices and subsystems.

3. **Creating initproc**: `main()` calls `userinit()`, which:
   - Allocates a proc structure.
   - Sets up a page table with the kernel mappings.
   - Allocates one page of user memory and loads the initcode binary into it.
   - Sets up a trap frame to make it look like the process is returning from a trap to address 0.
   - Marks the process as RUNNABLE.

4. **Starting the scheduler**: `main()` calls `mpmain()`, which calls `scheduler()`. The scheduler finds the initproc and runs it.

5. **First user code**: The initcode program runs. It makes one system call: `exec("/init")`.

6. **Loading init**: The exec system call loads /init from the file system, replaces the process's memory, and starts executing it.

7. **Init sets up I/O**: The init program creates the console device (if needed), opens it, and dups the file descriptor to set up stdin, stdout, and stderr.

8. **Starting the shell**: Init forks and execs the shell. The parent (init) waits for children.

9. **User interaction**: The shell prints a prompt, reads a command from the console, forks a child, and execs the command. The user can now interact with xv6!

And that's how xv6 boots into user space!

## The Elegance of the Design

There are several beautiful aspects of this design:

**1. Reusing code**: By having initcode call `exec()`, xv6 avoids having to special-case the first process. The same `exec()` code that loads user programs later is used to load the first user program.

**2. Fake trap frame**: By setting up a trap frame and using the normal trap return path, xv6 avoids having to special-case the transition to user space. The same `trapret` code that returns from system calls is used to start the first process.

**3. File descriptor setup**: By having init set up the console as file descriptors 0, 1, and 2, all programs inherit these file descriptors through fork/exec. This is why user programs can just `printf()` to the console without any special setup.

**4. Zombie reaping**: By having init loop forever and wait for children, xv6 ensures that orphaned processes get reaped and don't clog up the process table.

**5. Shell resilience**: By restarting the shell if it exits, init ensures that the system is always usable.

All of these design choices make the code simpler, more robust, and more Unix-like. This is the Unix philosophy in action: simple pieces that work together well.

## A Note on Process IDs

You might have noticed that we never explicitly set a process ID for the first process. What PID does it get?

If you look back at `allocproc()` (in proc.c), you'll see:

```c
static int nextpid = 1;
...
p->pid = nextpid++;
```

So the first process gets PID 1. This is traditional in Unix systems -- init always has PID 1.

Actually, wait! Let me check... in xv6, `userinit()` calls `allocproc()`, which increments nextpid. So the first process (initcode/init) actually gets PID 1. Good!

(In some Unix systems, there's also a PID 0 for the "swapper" or "scheduler" process, but xv6 keeps it simple and starts with 1.)

## What About Other CPUs?

We've been talking about the first process on the boot CPU, but what about other CPUs on a multiprocessor system?

Remember from the boot process that `main()` calls `startothers()` to start the other CPUs. Each of those CPUs ends up calling `mpmain()`, which in turn calls `scheduler()`.

All CPUs run the same scheduler code and compete for runnable processes in the shared process table. So any CPU might end up running the init process, the shell, or any user program. The locks in the process table ensure that only one CPU runs each process at a time.

This is SMP (symmetric multiprocessing) -- all CPUs are equal and can run any process. There's no special "master" CPU after boot completes.

## Wrapping Up

Okay, let's recap what we learned:

1. **userinit()** creates the first process by:
   - Allocating a proc structure and kernel stack.
   - Setting up a page table with kernel mappings.
   - Loading the initcode binary into user memory.
   - Setting up a fake trap frame to start at address 0 in user mode.

2. **initcode.S** is a tiny assembly program that makes one system call: `exec("/init")`.

3. **init.c** is the first real user program. It:
   - Creates the console device (if needed).
   - Sets up file descriptors 0, 1, and 2 to point to the console.
   - Forks and execs the shell.
   - Loops forever, waiting for and reaping zombie children.

4. **The shell** gives users a command prompt where they can run programs.

The beauty of this design is that it reuses existing code paths (exec, trap return) and avoids special cases. The first process is created and run using the exact same mechanisms as all other processes!

In the next post, we'll dive into the shell itself and see how it parses commands, handles pipes and redirection, and generally provides the command-line interface that makes Unix systems so powerful. See you there!