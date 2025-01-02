# Lec10：Lock

## UART 中 Lock 的运用 <a href="#es3eq" id="es3eq"></a>

对于 UART 模块也需要保证并发安全问题。

当UART硬件完成传输，会产生一个中断。usrtsatrt 的调用者会获得锁，确保不会多个进程同时像 THR 寄存器写数据。但是 UART 本身也可能与调用 printf 的进程并行。

如果一个进程调用了printf，它运行在CPU0上，会调用 uartstart 函数；CPU1处理了UART中断，那么CPU1也会调用uartstart 函数。

驱动的bottom部分（注，也就是中断处理程序）和驱动的up部分（注，uartputc函数）可以完全的并行运行，所以中断处理程序也需要获取锁。

## Spin Lock <a href="#ptboz" id="ptboz"></a>

xv6 用结构体 `struct spinlock`（1401）。结构体中的临界区用 `locked` 表示。这是一个字，在锁可以被获得时值为0，而当锁已经被获得时值为非零。

```
void
acquire(struct spinlock *lk)
{
    for(;;) {
        if(!lk->locked) {
            lk->locked = 1;
            break;
        }
    }
}
```

然而这段代码在现代处理器上并不能保证互斥。有可能两个（或多个）CPU 接连执行到第25行，发现 `lk->locked` 为0，然后都执行第26、27行拿到了锁。这时，两个不同的 CPU 持有锁，违反了互斥。这段代码不仅不能帮我们避免竞争条件，它本身就存在竞争。这里的问题主要出在第25、26行是分开执行的。若要保证代码的正确，就必须让第25、26行是原子操作的。

为了让这两行变为原子操作， xv6 采用了386硬件上的一条特殊指令 `xchg`（0569）。在这个原子操作中，`xchg` 交换了内存中的一个字和一个寄存器的值。

这个指令接收3个参数，分别是address，寄存器r1，寄存器r2。

通过这里的加锁，可以确保address中的数据存放于r2，而r1中的数据存放于address中，并且这一系列的指令打包具备原子性。

### 如果 acquire 不关闭中断怎么办？

uartputc函数会acquire锁，UART本质上就是传输字符，当UART完成了字符传输它会做什么？是的，它会产生一个中断之后会运行uartintr函数，在uartintr函数中，会获取同一把锁，但是这把锁正在被uartputc持有。如果这里只有一个CPU的话，那这里就是死锁。中断处理程序uartintr函数会一直等待锁释放，但是CPU不出让给uartputc执行的话锁又不会释放。在XV6中，这样的场景会触发panic，因为同一个CPU会再次尝试acquire同一个锁。

所以spinlock需要处理两类并发，一类是不同CPU之间的并发，一类是相同CPU上中断和普通程序之间的并发。针对后一种情况，我们需要在acquire中关闭中断。中断会在release的结束位置再次打开，因为在这个位置才能再次安全的接收中断。

对于并发执行，很明显这将会是一个灾难。如果我们将critical section与加锁解锁放在不同的CPU执行，将会得到完全错误的结果。所以指令重新排序在并发场景是错误的。为了禁止，或者说为了告诉编译器和硬件不要这样做，我们需要使用memory fence或者叫做synchronize指令，来确定指令的移动范围。对于synchronize指令，任何在它之前的load/store指令，都不能移动到它之后。锁的acquire和release函数都包含了synchronize指令。
