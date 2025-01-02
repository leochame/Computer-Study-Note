# Lec 13：sleep Andwake up

## 锁的限制条件 <a href="#v48ah" id="v48ah"></a>

在执行`switch`函数的过程中，XV6对锁有一些严格的限制，尤其是禁止在执行`switch`函数时持有任何除进程自身锁（`p->lock`）以外的锁。这是为了防止死锁的发生。

**死锁问题示例：**

假设有两个进程，进程P1和进程P2，它们分别持有不同的资源（如磁盘、UART或console的锁），然后都尝试切换线程。如果在切换过程中，P1持有了某个资源的锁并且试图调用`switch`，但在`switch`调用之前又没有释放该锁，那么P2如果需要该锁时会进入自旋等待。当P1无法获取到CPU时，P2永远也无法出让CPU，因为P2在自旋等待锁，P1又无法继续执行并释放锁，形成死锁。

为了防止这种死锁，XV6要求进程在调用`switch`时，**不能持有除自身进程锁（**`p->lock`**）外的任何其他锁**。

**那么为什么不可以使用中断机制，中断自旋锁呢？**

* 所有进程切换发生在内核中，所有的 acquire，switch，release 都是内核代码。如果XV6正在执行内核代码时发生了定时器中断，中断处理程序会调用yield函数并出让CPU。但是acquire函数在等待锁之前会关闭中断，否则的话可能会引起死锁。

***

## Sleep & Wakeup 协调机制 <a href="#y7rxs" id="y7rxs"></a>

在多线程编程中，经常需要等待某些特定事件的发生，例如：

* 等待 Pipe 缓冲区非空事件。
* 等待读取磁盘块完成事件。
* Unix 进程调用 `wait` 等待子进程退出事件。

***

我们来看看引入 Sleep\&Wakeup 机制和传统方式的区别：

**传统的 Busy-Wait 问题**

* **实现方式**：通过循环不断检查条件是否满足。
* **缺陷**：耗费 CPU 时间，特别是等待时间较长时（如毫秒甚至分钟级），极度浪费资源。

**引入 Sleep & Wakeup**

* Sleep：线程主动放弃 CPU，直到某事件发生。
* Wakeup：事件发生后通知相关线程重新运行。

***

UART（通用异步收发传输器）硬件只能一次接收一个字符，因此需要等待硬件准备好接收下一个字符：

* 每次写入字符后，线程进入睡眠状态.
* UART 中断触发时唤醒线程并准备写入下一个字符。

这是经典的硬件驱动风格。

当 shell 需要输出时会调用 write 系统调用最终走图`uartwrite()` 。UART硬件会在完成传输一个字符后，触发一个中断。所以UART驱动中除了uartwrite函数外，还有名为uartintr的中断处理程序。

中断处理程序（`uartintr`）读取硬件寄存器（memory-mapped register），检查 `LSR_TX_IDLE` 标志位：

* 若标志位为 `1`，说明传输完成，设置 `tx_done = 1`。
* 调用 `wakeup(tx_channel)`，通知等待的线程可以继续操作。

***

这时候我们大概了解了 sleep 和 wakeup 的流程，这里我们简洁的总结一下：、

* 线程在 `uartwrite` 中调用 `sleep(tx_channel, lock)`，进入睡眠等待 `tx_done = 0`。
* 中断处理程序完成传输后，`uartintr` 将标志位 set 为 1， 调用 `wakeup` 唤醒线程。
* 唤醒的线程检查条件（`tx_done == 1`），继续传输下一个字符，并再次进入睡眠。

另外`sleep` 和 `wakeup` 是成对出现的，通过共享的 **sleep channel** 链接，**Sleep Channel** 是一个64位标识，用于关联特定的事件和线程：

* `sleep(channel)`：线程等待某个事件的发生。
* `wakeup(channel)`：唤醒等待特定事件的线程。

## Lost Wakeup <a href="#nrgg7" id="nrgg7"></a>

"Lost wake-up"问题是指，当进程在等待某个条件发生时，它可能被提前唤醒（即在调用`sleep`前已发生了`wakeup`），导致`wakeup`操作失效，进程未被唤醒。这个问题通常出现在多核系统中，特别是涉及到中断的情况。

例如，在UART驱动中，`uartwrite`函数调用`sleep`时，如果在`sleep`和`wakeup`之间发生了中断（且该中断处理程序也涉及`wakeup`操作），就有可能导致`wakeup`在进程还未完全进入`SLEEPING`状态之前就已经执行，这样进程的状态不会被正确更新，导致“lost wake-up”。

### 1. lost wakeup 问题的产生 <a href="#fjr0c" id="fjr0c"></a>

**核心问题：**

* **线程A** 在 `uartwrite` 中调用了 `sleep` 函数，并且期望在它等待过程中，如果 UART 传输完成，**线程B** 会通过 `wakeup` 唤醒它。
* 但由于没有锁保护，**线程B** 可能在 **线程A** 进入 `sleep` 之前就已经调用了 `wakeup`，并且唤醒了不该唤醒的线程。
* 这意味着线程A在 `sleep` 期间没有机会进入休眠状态，导致 `wakeup` 的唤醒被“丢失”，这就是 **lost wakeup**。

**为什么会发生这种情况：**

* `wakeup` 和 `sleep` 依赖于共享数据（如 `done` 标志位）来决定是否唤醒等待的进程。
* 如果在 `sleep` 执行期间没有正确加锁，`wakeup` 可能会提前运行，进而影响 `sleep` 的正确行为，导致在调用 `sleep` 时没有相应的进程可以被唤醒。

### 2. 锁的作用和加锁位置 <a href="#ejj8u" id="ejj8u"></a>

共享资源（如 `done` 标志位或 `tx_channel`）会被多个线程同时访问时，所以必须加锁来保护这些共享数据，确保没有竞争条件。

**那么如何加锁呢？**

`uartwrite` 需要遍历 `buffer` 并发送每个字符。如果我们在每次发送字符时都加锁，会导致一个问题：`uartwrite` 持有锁时，中断处理程序（`uartintr`）无法获取锁，因此无法修改 `done` 标志位。如果 `done` 标志位不能及时被设置为 1，`uartwrite` 就会一直等待（`while not done`），造成死锁。

我们不能在发送每个字符时都持有锁，而是要在**发送字符的开始时**加锁，**在进入 `sleep` 函数之前释放锁**。这样可以允许中断处理程序在适当的时候获取锁，修改 `done` 标志位并唤醒 `uartwrite`。所以我们有了如下方案：

* **在** `uartwrite` **函数中加锁：** 需要保护共享变量 `done`，同时防止多个线程同时访问 UART 硬件。这样可以确保 `uartwrite` 中的每个字符写入过程不会受到中断的影响。
* **在** `sleep` **函数中释放锁：** 由于 `sleep` 可能会导致线程进入休眠，因此在调用 `sleep` 前释放锁，允许中断处理程序执行并唤醒相关进程。然后，在 `sleep` 返回后重新获取锁。这样一来，`uartwrite` 在发送字符时释放了锁，允许中断处理程序执行，并确保 `wakeup` 发生时，进程已经处于正确的状态。

```c
// 在uartwrite中加锁
acquire(uart_tx_lock);

// 发送字符之前释放锁，让中断可以运行
while (!done) {
    release(uart_tx_lock);  // 释放锁
    broken_sleep(tx_channel);  // 进入休眠
    acquire(uart_tx_lock);  // 重新获取锁
}

// 继续执行发送字符
```

这样，`uartwrite` 在每次发送字符时，都会在等待过程中释放锁，允许中断处理程序运行并可能唤醒进程，避免 **lost wakeup** 问题。

### 消除 Lost Wakeup <a href="#pvppx" id="pvppx"></a>

我们必须要释放uart\_tx\_lock锁，因为中断需要获取这个锁，但是我们又不能在释放锁和进程将自己标记为SLEEPING之间留有窗口。

为了避免这种情况，需要将这两个操作（释放锁和设置 `SLEEPING` 状态）合并成一个原子操作。这样可以确保当锁被释放时，进程已经进入 `SLEEPING` 状态，`wakeup` 才有可能成功唤醒它。

为了实现上述目标，`sleep` 函数需要知道保护进程 `SLEEPING` 状态和条件的锁（例如 `uart_tx_lock`），确保中断可以获得锁并进行唤醒 。所以代码流程图下

* `sleep` 函数的第一个操作是释放传入的锁（这里写作`lk`，从而允许其他中断或线程访问该资源。
* `sleep` 会获取当前进程的锁，设置进程为 `SLEEPING` 状态，并记录`sleep channel`。
* 之后，调用 `sched()` 函数切换进程 。

但是实际代码中，我们会先获取当前进程的锁， 这样可以确保 `wakeup` 函数无法在进程状态设置为 `SLEEPING` 之前执行（因为 wakeup 执行前会先获取进程锁），然后再释放传入的锁。

```c
// Atomically release lock and sleep on chan.
// Reacquires lock when awakened.
void
sleep(void *chan, struct spinlock *lk)
{
  struct proc *p = myproc();
  
  // Must acquire p->lock in order to
  // change p->state and then call sched.
  // Once we hold p->lock, we can be
  // guaranteed that we won't miss any wakeup
  // (wakeup locks p->lock),
  // so it's okay to release lk.

  acquire(&p->lock);  //DOC: sleeplock1
  release(lk);

  // Go to sleep.
  p->chan = chan;
  p->state = SLEEPING;

  sched();

  // Tidy up.
  p->chan = 0;

  // Reacquire original lock.
  release(&p->lock);
  acquire(lk);
}
```

`wakeup` 函数执行的流程很直观：

1. 加锁：它首先获取整个进程表的锁。
2. 检查每个进程的状态，如果进程是 `SLEEPING` 并且等待的条件（ `sleep channel`）匹配，就将该进程的状态设为 `RUNNABLE`。
3. 释放进程锁，唤醒该进程。

```c
// Wake up all processes sleeping on chan.
// Must be called without any p->lock.
void
wakeup(void *chan)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    if(p != myproc()){
      acquire(&p->lock);
      if(p->state == SLEEPING && p->chan == chan) {
        p->state = RUNNABLE;
      }
      release(&p->lock);
    }
  }
}
```

### Question <a href="#qqeh9" id="qqeh9"></a>

1. 为什么要使用 `while(tx_done == 0)` 这样的循环来调用 `sleep`？如果已经知道是由 UART 中断处理程序唤醒的，`tx_done` 是否显得多余？

我们不能简单地假设当 `sleep` 函数返回时，事件一定发生了。特别是在涉及硬件操作（如 UART）时，可能会出现其他进程或线程在我们调用 `sleep` 时进入同一临界区，或者两个进程同时尝试访问 UART，导致资源冲突。在这种情况下，虽然有一个进程被唤醒，但并不意味着它能立即进行操作，因为另一个进程可能已经处理了 UART，导致我们需要再次进入 `sleep` 等待。



## Sleep和具体函数结合

其实我们根据 `uartputc`函数可以看出，最开始先获取 `condition lock`，持有`condition lock`调用 `sleep` 函数。

```c
// add a character to the output buffer and tell the
// UART to start sending if it isn't already.
// blocks if the output buffer is full.
// because it may block, it can't be called
// from interrupts; it's only suitable for use
// by write().
void
uartputc(int c)
{
  acquire(&uart_tx_lock);

  if(panicked){
    for(;;)
      ;
  }
  while(uart_tx_w == uart_tx_r + UART_TX_BUF_SIZE){
    // buffer is full.
    // wait for uartstart() to open up space in the buffer.
    sleep(&uart_tx_r, &uart_tx_lock);
  }
  uart_tx_buf[uart_tx_w % UART_TX_BUF_SIZE] = c;
  uart_tx_w += 1;
  uartstart();
  release(&uart_tx_lock);
}
```

前面这个特定的场景中，sleep等待的condition是发生了中断并且硬件准备好了传输下一个字符。在一些其他场景，内核代码会调用sleep函数并等待其他的线程完成某些事情。其实本质上没有区别。



### Pipe <a href="#iayqp" id="iayqp"></a>

`sleep` 和 `wakeup`还有一个更加复杂的例子，就是关于同步消费者和生产者的实现，比如 pipe。

当`read`系统调用最终调用到`piperead`函数时，`pi->lock`会用来保护`pipe`，这就是`sleep`函数对应的`condition lock`。`piperead`需要等待的condition是pipe中有数据，而这个condition就`是pi->nwrite`大于`pi->nread`。条件不满足，调用 `sleep`函数。

```c
int
piperead(struct pipe *pi, uint64 addr, int n)
{
  int i;
  struct proc *pr = myproc();
  char ch;

  acquire(&pi->lock);
  while(pi->nread == pi->nwrite && pi->writeopen){  //DOC: pipe-empty
    if(killed(pr)){
      release(&pi->lock);
      return -1;
    }
    sleep(&pi->nread, &pi->lock); //DOC: piperead-sleep
  }
  for(i = 0; i < n; i++){  //DOC: piperead-copy
    if(pi->nread == pi->nwrite)
      break;
    ch = pi->data[pi->nread++ % PIPESIZE];
    if(copyout(pr->pagetable, addr + i, &ch, 1) == -1)
      break;
  }
  wakeup(&pi->nwrite);  //DOC: piperead-wakeup
  release(&pi->lock);
  return i;
}
```

下面是 `pipewrite` 的代码:

```c
int
pipewrite(struct pipe *pi, uint64 addr, int n)
{
    int i = 0;
    struct proc *pr = myproc();

    acquire(&pi->lock);
    while(i < n){
        if(pi->readopen == 0 || killed(pr)){
            release(&pi->lock);
            return -1;
        }
        if(pi->nwrite == pi->nread + PIPESIZE){ //DOC: pipewrite-full
            wakeup(&pi->nread);
            sleep(&pi->nwrite, &pi->lock);
        } else {
            char ch;
            if(copyin(pr->pagetable, &ch, addr + i, 1) == -1)
                break;
            pi->data[pi->nwrite++ % PIPESIZE] = ch;
            i++;
        }
    }
    wakeup(&pi->nread);
    release(&pi->lock);

    return i;
}
```



### Exit 系统调用 <a href="#d2zsr" id="d2zsr"></a>

在XV6中，一个进程如果退出的话，我们需要释放用户内存，释放page table，释放trapframe对象，将进程在进程表单中标为REUSABLE，这些都是典型的清理步骤。

这就会带来几个问题，一个是不能直接摧毁另一个线程，另一个是主动退出也要释放关键资源：

* 线程可能正在另一个 CPU 核上运行，使用自己的栈。线程可能持有关键的内核锁，如果直接销毁，可能会导致死锁或资源泄漏。线程可能正在更新复杂的内核数据结构，强行销毁会导致系统状态不一致。
* 即便进程主动调用 `exit`，它在完成退出逻辑之前仍需要一些关键资源（如栈、进程表项）。如果这些资源在进程退出过程中提前释放，会导致运行时异常。

那我们看看如何解决这个问题的：

XV6有两个函数与关闭线程进程相关。第一个是exit，第二个是kill。让我们先来看位于proc.c中的exit函数。

```c
// Exit the current process.  Does not return.
// An exited process remains in the zombie state
// until its parent calls wait().
void
exit(int status)
{
  struct proc *p = myproc();

  if(p == initproc)
    panic("init exiting");

  // Close all open files.
  for(int fd = 0; fd < NOFILE; fd++){
    if(p->ofile[fd]){
      struct file *f = p->ofile[fd];
      fileclose(f);
      p->ofile[fd] = 0;
    }
  }

  begin_op();
  iput(p->cwd);
  end_op();
  p->cwd = 0;

  acquire(&wait_lock);

  // Give any children to init.
  reparent(p);

  // Parent might be sleeping in wait().
  wakeup(p->parent);
  
  acquire(&p->lock);

  p->xstate = status;
  p->state = ZOMBIE;

  release(&wait_lock);

  // Jump into the scheduler, never to return.
  sched();
  panic("zombie exit");
}
```

从上面中我们可以看到 `exit`函数会关闭所有已经打开的文件。进程有一个对于当前目录（共享资源）的记录，这个记录会随着你执行cd指令而改变。在exit过程中也需要将对这个目录的引用释放给文件系统。

子进程需要有一个有效的父进程。如果父进程已经退出，通过`reparent`将所有子进程的父进程被重定向到 `init`（PID=1）。`init` 是操作系统中的特殊进程，专门负责处理孤儿进程。

之后，我们需要通过调用wakeup函数唤醒当前进程的父进程，当前进程的父进程或许正在等待当前进程退出。

标记状态为 `ZOMBIE`。现在进程还没有完全释放它的资源，所以它还不能被重用。

所谓的进程重用是指，我们期望在最后，进程的所有状态都可以被一些其他无关的fork系统调用复用，但是目前我们还没有到那一步。

调用 `sched` 切换到调度器线程。调度器只运行 `RUNNABLE` 状态的进程，`ZOMBIE` 状态的进程不再执行。由父进程调用 `wait` 系统调用，将其状态设置为 `UNUSED` 并释放所有资源。另外由于进程状态是 ZOMBIE,调度线程不会再运行这个线程。



### Wait 系统调用 <a href="#xirml" id="xirml"></a>

`wait` 的作用是让父进程等待其子进程退出。当一个子进程调用 `exit` 后，它的状态会被设置为 `ZOMBIE`，父进程通过 `wait` 可以获取到子进程的退出信息。

```c
// Wait for a child process to exit and return its pid.
// Return -1 if this process has no children.
int
wait(uint64 addr)
{
  struct proc *pp;
  int havekids, pid;
  struct proc *p = myproc();

  acquire(&wait_lock);

  for(;;){
    // Scan through table looking for exited children.
    havekids = 0;
    for(pp = proc; pp < &proc[NPROC]; pp++){
      if(pp->parent == p){
        // make sure the child isn't still in exit() or swtch().
        acquire(&pp->lock);

        havekids = 1;
        if(pp->state == ZOMBIE){
          // Found one.
          pid = pp->pid;
          if(addr != 0 && copyout(p->pagetable, addr, (char *)&pp->xstate,
                                  sizeof(pp->xstate)) < 0) {
            release(&pp->lock);
            release(&wait_lock);
            return -1;
          }
          freeproc(pp);
          release(&pp->lock);
          release(&wait_lock);
          return pid;
        }
        release(&pp->lock);
      }
    }

    // No point waiting if we don't have any children.
    if(!havekids || killed(p)){
      release(&wait_lock);
      return -1;
    }
    
    // Wait for a child to exit.
    sleep(p, &wait_lock);  //DOC: wait-sleep
  }
}
```

当一个进程调用了wait系统调用，它会扫描进程表单，找到父进程是自己且状态是ZOMBIE的进程。

当找到子进程时，父进程会通过调用 `freeproc` 来释放资源。直到这一点，`ZOMBIE` 状态的进程才会被清理，进程表项才会被标记为 `UNUSED`，可以被重用。

`freeproc`：负责释放 `trapframe`、`page table`和其他与进程相关的资源，如内核栈等。

\
在某些情况下，父进程和子进程可能会几乎同时调用 `exit`。此时子进程可能会试图唤醒父进程，并告知其已经退出，而父进程此时也在退出。

* **在子进程退出时，父进程仍然可能在 `wait` 等待子进程退出**。
* 为了避免父进程和子进程在退出时相互干扰，子进程需要先获取进程锁，并在确保不会发生竞态条件的情况下唤醒父进程。
* **关键设计**：通过在调用 `wakeup` 唤醒父进程之前先获取当前进程的父进程信息，确保在父子进程同时退出的情况下仍然能够正确唤醒父进程，且不会造成资源清理的冲突。

### Kill 系统调用 <a href="#hbgqi" id="hbgqi"></a>

关于 `kill` 系统调用的工作方式和设计，其核心目的并不是直接停止目标进程的执行，而是做如下操作：

* 在目标进程的 `proc` 结构体中设置 `killed` 标志位为 1。
* 如果目标进程当前处于 `SLEEPING` 状态，它会被设置为 `RUNNABLE`，从而可以被调度器重新调度运行。

这里的关键点是：**`kill` 调用不会立刻杀死进程**，而只是为目标进程设置了一个标志，进程会在合适的时机自行退出。

#### Kill 系统调用的具体工作流程 <a href="#o03nl" id="o03nl"></a>

```c
int
kill(int pid)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++){
    acquire(&p->lock);
    if(p->pid == pid){
      p->killed = 1;
      if(p->state == SLEEPING){
        // Wake process from sleep().
        p->state = RUNNABLE;
      }
      release(&p->lock);
      return 0;
    }
    release(&p->lock);
  }
  return -1;
}
```

当一个进程调用 `kill` 系统调用并尝试杀死另一个进程时：

* **设置 `killed` 标志**：\
  在进程表中，目标进程的 `killed` 标志会被设置为 1。这个标志是一个软中断信号，它告诉进程需要退出。
* **如果进程正在 `SLEEPING`**：\
  如果目标进程当前处于 `SLEEPING` 状态（例如，它在等待某个事件发生，如 I/O 操作），它的状态会被改变为 `RUNNABLE`，也就是说，它可以重新被调度器选中执行。
* **进程检查 `killed` 标志**：\
  当目标进程继续执行时，它会定期检查自己是否被设置为 `killed`。这种检查通常发生在进程执行系统调用时或在返回内核空间时。比如，在 `usertrap` 函数中，进程会在执行系统调用之前和之后检查这个标志。如果它被设置为 `1`，进程就会执行 `exit`，以便清理资源并退出。

```c
void
usertrap(void)
{
    int which_dev = 0;

    if((r_sstatus() & SSTATUS_SPP) != 0)
        panic("usertrap: not from user mode");

    // send interrupts and exceptions to kerneltrap(),
    // since we're now in the kernel.
    w_stvec((uint64)kernelvec);

    struct proc *p = myproc();

    // save user program counter.
    p->trapframe->epc = r_sepc();

    if(r_scause() == 8){
        // system call

        if(killed(p))
            exit(-1);

        // sepc points to the ecall instruction,
        // but we want to return to the next instruction.
        p->trapframe->epc += 4;

        // an interrupt will change sepc, scause, and sstatus,
        // so enable only now that we're done with those registers.
        intr_on();

        syscall();
    } else if((which_dev = devintr()) != 0){
        // ok
    } else {
        printf("usertrap(): unexpected scause 0x%lx pid=%d\n", r_scause(), p->pid);
        printf("            sepc=0x%lx stval=0x%lx\n", r_sepc(), r_stval());
        setkilled(p);
    }
    err:
    if(killed(p))
        exit(-1);

    // give up the CPU if this is a timer interrupt.
    if(which_dev == 2)
        yield();

    usertrapret();
}
```

#### sleeping 线程被 kill 会发生什么？

如果目标进程是SLEEPING状态，kill函数会将其状态设置为RUNNABLE，这意味着，即使进程之前调用了sleep并进入到SLEEPING状态，调度器现在会重新运行进程，并且进程会从sleep中返回。我们以 `piperead` 函数为例：

如果一个进程正在sleep状态等待从pipe中读取数据，然后它被kill了。`kill`函数会将其设置为RUNNABLE，之后进程会从`sleep`中返回，返回到循环的最开始。

pipe中大概率还是没有数据，之后在`piperead`中，会判断进程是否被kill了。如果进程被kill了，那么接下来`piperead`会返回-1，并且返回到`usertrap`函数的`syscall()`位置，因为`piperead`就是一种系统调用的实现。

如在磁盘操作（例如 `virtio_disk.c` 中的磁盘读取）等关键操作中，进程通常不会在等待期间检查 `killed` 标志。这样做是因为这些操作可能涉及多个步骤（如多个磁盘读写），在整个操作完成之前，强行退出进程可能会导致数据不一致或损坏。



### Question <a href="#hyfqo" id="hyfqo"></a>

#### 关于权限：为什么允许一个进程杀死另一个进程？ <a href="#e6f12908" id="e6f12908"></a>

在 `XV6` 这样的教学操作系统中，`kill` 调用没有权限检查，任何进程都可以杀死任何其他进程。这是为了简化教学内容并专注于操作系统的其他重要概念。在 **生产环境的操作系统**（如 Linux）中，`kill` 操作通常会进行权限检查，确保只有相同用户或具有足够权限的进程才能杀死目标进程，以防止恶意操作。
