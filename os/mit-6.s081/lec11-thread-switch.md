# Lec11：Thread Switch

任何操作系统都会出现，进程多于操作系统的时候。

我们就需要给进程提供一种错觉，让它用自己的虚拟 CPU 复用硬件 CPU。

***

xv6 CPU切换进程实现 Multiplexing 一般是两种情况。一个是 sleep 和 wake up 会在 block（需要等待时间，例如 read，wait 和 sleep）时候切换进程。另一个就是，定期的强制切换；这个是发生在 CPU 计算密集型，也就是长时间计算的时候。

这时候 Multiplexing 就创造了一种每个进程都有 CPU 的错觉。

***

## 一、 Context Switch <a href="#wuugi" id="wuugi"></a>

<figure><img src="../../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

Figure7.1 涵盖了从一个用户进程切换到另一个的过程：通过 trap 从 shell 的用户进程到 old process's kernel thread，一个 context switch 到当前 CPU 的 sheduler thread，一个 contex switch 到 new thread 的 kernel thread，然后再经过 trap 返回用户空间。

**调度线程**是由操作系统专门为每个 CPU 创建的线程，负责调度不同进程的执行 。 因为调度程序在任何进程的内核栈上执行是不安全的：其他 CPU 可能会唤醒进程并运行这个进程，如果在两个不同的 CPU 上使用相同的栈，那将会造成灾难。所以，为了处理这种情况，每个 CPU 都会有一个独立的调度线程。

Switching 包含了保存原有线程的 CPU Register 和恢复 新线程之前保存的 CPU Register（第一次执行的话，创建这个进程的时候会有存储），

函数 swtch 保存和恢复 kernel 的寄存器，它本身并不知道线程，只是保存或恢复了一组 RISC-V 的寄存器，也称为 context。

当一个进程需要放弃 CPU 时，进程的内核线程会调用 `swtch` 来保存自身的上下文并恢复调度程序的上下文。每个上下文都包含在一个 `struct context` 结构体中，该结构体本身包含在进程的 `struct proc` 或 CPU 的 `struct cpu` 中。

```c
// Saved registers for kernel context switches.
struct context {
    uint64 ra;
    uint64 sp;

    // callee-saved
    uint64 s0;
    uint64 s1;	
    uint64 s2;
    uint64 s3;
    uint64 s4;
    uint64 s5;
    uint64 s6;
    uint64 s7;
    uint64 s8;
    uint64 s9;
    uint64 s10;
    uint64 s11;
};
```

`swtch` 接受两个参数：`struct context *old` 和 `struct context *new`。它会将当前的寄存器保存到 `old` 中，然后从 `new` 中加载寄存器，并返回。

让我们跟随一个进程通过 `swtch` 进入调度程序。我们知道，中断结束时的一种可能性是 ：

* `usertrap` 调用 `yield`。
* `yield` 又调用 `sched`。
* `sched` 调用 `swtch` 来保存当前的上下文到 `p->context` 并切换到先前保存在 `cpu->context` 中的调度程序上下文。

```c
// Switch to scheduler.  Must hold only p->lock
// and have changed proc->state. Saves and restores
// intena because intena is a property of this
// kernel thread, not this CPU. It should
// be proc->intena and proc->noff, but that would
// break in the few places where a lock is held but
// there's no process.
void
sched(void)
{
  int intena;
  struct proc *p = myproc();

  if(!holding(&p->lock))
    panic("sched p->lock");
  if(mycpu()->noff != 1)
    panic("sched locks");
  if(p->state == RUNNING)
    panic("sched running");
  if(intr_get())
    panic("sched interruptible");

  intena = mycpu()->intena;
  swtch(&p->context, &mycpu()->context);
  mycpu()->intena = intena;
}
```

swtch 不保存程序计数器，而是保存 `ra` 寄存器，它保存了从哪里调用 `swtch` 的返回地址。

接下来，`swtch` 从新的上下文中恢复寄存器，新的上下文包含先前 `swtch` 调用时保存的寄存器值。当 `swtch` 返回时，它返回到由恢复的 `ra` 寄存器指向的指令，也就是新线程之前调用 `swtch` 的地方。

此外，它会在新线程的栈上返回，因为恢复的 `sp` 指向的是新线程的栈。

`sched` 调用 `swtch` 切换到 `cpu->context`，即每个 CPU 的调度程序上下文。当我们跟踪的 `swtch` 返回时，它不会返回到 `sched`，而是返回到 `scheduler`，并且栈指针指向当前 CPU 的调度栈。

## 二、Scheduler <a href="#tpxsg" id="tpxsg"></a>

每个 CPU 都会运行着调度函数，scheduler 负责选择下一个负责运行的进程。 一个希望放弃 CPU 的进程必须先获得它自己的进程锁 `p->lock`，释放它持有的其他锁，更新它的状态 (`p->state`)，然后调用 `sched`。你可以在 `yield`、`sleep` 和 `exit` 中看到这一序列。

```c
// Give up the CPU for one scheduling round.
void
yield(void)
{
  struct proc *p = myproc();
  acquire(&p->lock);
  p->state = RUNNABLE;
  sched();
  release(&p->lock);
}
```

`sched` 会再次检查上面这些要求。另外，因为有锁被持有，应该禁用中断。最后，`sched` 调用 `swtch` 来保存当前的上下文到 `p->context`，并切换到保存在 `cpu->context` 中的调度程序上下文。`swtch` 在调度程序的栈上返回，就像调度程序的 `swtch` 已经返回了一样。调度程序继续其 `for` 循环，找到一个要运行的进程，切换到它，然后这个周期再次开始。

内核线程放弃 CPU 的唯一地方是在 `sched` 中，并且它总是切换到调度程序中的同一位置，调度程序（几乎）总是会切换到一个之前调用了 `sched` 的内核线程 。

通常情况下，`swtch` 返回时会返回到调用 `sched` 的地方。但是对于一个新进程，在它第一次被调度时，一个新进程被分配和初始化时，`allocproc` 函数会给新进程分配一个 `proc` 结构体，并设置进程的状态和上下文。在这时，`allocproc` 会特别地把新进程的上下文 `ra` 寄存器（返回地址寄存器）设置为 `forkret`，这意味着新进程的第一次上下文切换（即第一次执行 `swtch`）将会返回到 `forkret` 函数。

## 三、如何获取 CPU 或进程 <a href="#jao1s" id="jao1s"></a>

在单核处理器上，我们可以通过一个全局变量来获取当前运行的进程 `proc` 结构体的指针。但在多核处理器上，每个 CPU 可能在运行不同的进程，因此不能使用单一的全局变量来指示当前进程，因为不同的 CPU 执行的进程是不同的。

为了解决这个问题，Xv6 利用了每个 CPU 有自己的一组寄存器的特性。具体来说，Xv6 使用 `tp` 寄存器来保存每个 CPU 的 `hartid`，这是一种简单的 CPU 唯一标识符

`mycpu` 函数（在 `kernel/proc.c:74`）使用 `tp` 来索引一个 CPU 结构体数组，进而返回当前 CPU 的结构体（`struct cpu`）。这个结构体包含当前运行的进程（如果有）的 `proc` 指针、保存该 CPU 的调度线程的寄存器、以及用于管理中断禁用的自旋锁计数

```c
// Return this CPU's cpu struct.
// Interrupts must be disabled.
struct cpu*
mycpu(void)
{
  int id = cpuid();
  struct cpu *c = &cpus[id];
  return c;
}
```

### tp 如何保存 CPU 的 `hartid` 呢？ <a href="#zhyxb" id="zhyxb"></a>

* 在 CPU 启动时，`start` 函数会将 `tp` 设置为当前 CPU 的 `hartid`。
* 当进入用户空间时，`usertrapret` 会保存 `tp` 到 trampoline 页，以防用户代码修改它。
* 然后，`uservec` 函数会在从用户空间进入内核时恢复 `tp` 寄存器，确保它始终正确。

### 我们是怎么获取到当前运行的进程的？ <a href="#oxw2r" id="oxw2r"></a>

我们知道了`mycpu` 返回当前 CPU 的 `struct cpu` 指针。那么我们看看 `myproc` 函数怎么 获取当前 CPU 上运行的进程的 `struct proc` 指针。

为了避免中断影响，`myproc` 会禁用中断，调用 `mycpu` 获取当前 CPU 的结构体，再从中提取 `c->proc`（当前进程的指针）。之后它会重新启用中断。

```c
// Return the current struct proc *, or zero if none.
struct proc*
myproc(void)
{
  push_off();
  struct cpu *c = mycpu();
  struct proc *p = c->proc;
  pop_off();
  return p;
}
```

## 四、第一次调用 swtch 发生了什么？ <a href="#l1pf2" id="l1pf2"></a>

当一个进程第一次调用 **`swtch`** 函数时，这个进程并不直接跳到其他代码。它必须先伪造一个“另一个线程”调用 **`swtch`** 的场景。这是因为在进程的第一次 **上下文切换** 时，我们不能直接跳转到其他位置——必须有一个合理的调用返回点和栈。

这时候，**`allocproc`** 函数会为新进程创建一个上下文，其中 `ra`（返回地址寄存器）和 `sp`（栈指针寄存器）尤为重要。`ra` 存储的是进程 **第一次切换后应该返回的位置**，也就是 **`forkret`** 函数的开始位置。

```c
// Look in the process table for an UNUSED proc.
// If found, initialize state required to run in the kernel,
// and return with p->lock held.
// If there are no free procs, or a memory allocation fails, return 0.
static struct proc*
allocproc(void)
{
    .......
        
  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  return p;
}
```

**`forkret`** 是一个特殊的函数，它在新进程创建后被作为 **第一次切换** 后的返回位置。也就是说，当 **`swtch`** 函数首次执行时，进程会被切换到 **`forkret`** 函数的开始位置，就像是 **`forkret`** 调用了 **`swtch`** 函数并返回了一样。

**`forkret`** 函数仅在进程的第一次切换时使用，它的作用仅限于启动新进程时通过 **`swtch`** 来完成首次的上下文切换。

```c
// A fork child's very first scheduling by scheduler()
// will swtch to forkret.
void
forkret(void)
{
  static int first = 1;

  // Still holding p->lock from scheduler.
  release(&myproc()->lock);

  if (first) {
    // File system initialization must be run in the context of a
    // regular process (e.g., because it calls sleep), and thus cannot
    // be run from main().
    fsinit(ROOTDEV);

    first = 0;
    // ensure other cores see first=0.
    __sync_synchronize();
  }

  usertrapret();
}
```

举例：

* 想象你有一个**进程A**，它要切换到**进程B**。这时，进程A的上下文（寄存器、栈等信息）会被保存，CPU会去加载进程B的上下文。
* 但如果进程B是刚刚通过 **`fork`** 创建的，那么它的第一次上下文切换不能直接跳到代码中。相当于它第一次**出场**时需要通过 **`forkret`** 函数来为自己“准备好舞台”。
* 然后，之后的进程切换就会进入正常的调度过程，**`forkret`** 就不再参与其中。
