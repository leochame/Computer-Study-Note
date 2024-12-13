# Lec 05 Trap



## 5.0 Trap

**用户空间和内核空间的切换通常被称为trap。**&#x56E0;为很多应用程序，要么因为系统调用，要么因为page fault，都会频繁的切换到内核中。所以，trap机制要尽可能的简单，这一点非常重要。

Trap是CPU中断正常指令流的总称，涵盖了1.系统调用、2.异常和3.设备中断(for example when the disk hardware finishes a read or write request)三种情况。以上情况统称trap。每种情况触发的条件不同，但处理逻辑都类似：切换到内核模式处理事件。

透明性确保了被中断的代码不需要知道中断发生的细节，从而能够在中断完成后继续执行原任务。

尽管这三类Trap在处理流程上具有共性，内核可以用一条通用的代码路径来处理所有Trap，但将用户态Trap与内核态Trap分开处理会更方便。处理Trap的代码（无论是汇编还是C语言）通常被称为**处理程序（handler）**；而通常最初的一些处理程序指令用汇编语言编写，这些汇编指令有时也被称为**向量（vector）**。



## 5.1 RiSC-V Trap 机制

每个 RISC-V CPU 都有一组控制寄存器，内核通过写入这些寄存器来指示 CPU 如何处理Trap；同时，内核也可以通过读取这些寄存器来了解已经发生的Trap况。

* **stvec(**&#x53;upervisor Trap Vector Base Address Registe&#x72;**)**：内核将其处理程序的地址写入此寄存器；当发生Trap时，RISC-V 跳转到 `stvec` 中指定的地址处理Trap。
* **sepc**：发生Trap时，RISC-V 将程序计数器（`pc`）保存到 `sepc` 中（因为此时 `pc` 被 `stvec` 的值覆盖）。`sret`（从Trap返回的指令）会将 `sepc` 的值复制回 `pc`。内核可以写入 `sepc` 以控制 `sret` 的返回地址。**在Trap过程中保存程序计数器的值。**
* **scause**：RISC-V 在此寄存器中记录描述Trap原因的编号。
* **sscratch**：Trap处理程序使用 `sscratch` 帮助避免在保存用户寄存器之前覆盖它们。
* **sstatus**：
  * `SIE` 位控制设备中断是否启用。如果内核清除了 `SIE` 位，RISC-V 会延迟设备中断，直到内核重新设置 `SIE`。
  * `SPP` 位指示Trap来自用户模式还是监督模式，并决定 `sret` 返回到哪个模式。

上述寄存器仅与在监督模式下处理的Trap相关，用户模式下无法读取或写入这些寄存器。

在多核芯片上，每个 CPU 都有自己独立的一组寄存器，多个 CPU 可能同时处理Trap。



**注意：**

**CPU最小化工作：**&#x43;PU 在发生Trap时不会切换到内核页表，不会切换到内核堆栈，也不会保存除 `pc` 之外的任何寄存器。这些任务需要内核软件完成。CPU最小化给了软件最大灵活性和选择。

**确保 CPU 切换到内核指定的指令地址（即 `stvec`）至关重要**：如果 CPU 不切换程序计数器（`pc`），则用户空间的Trap可能在保持用户指令运行的同时切换到监督模式。这可能会破坏用户/内核隔离，例如通过修改 `satp` 寄存器指向一个允许访问所有物理内存的页表。





## 5.2 Trap from user space

Xv6 在处理Trap时，根据Trap发生在内核代码还是用户代码中采取不同的方式

接下来描述了当用户程序发生某些特殊情况（比如调用系统功能、非法操作或者设备中断）时，系统如何接管控制权、处理这些情况，并最终将控制权交还给用户程序的过程。

***



当用户程序执行系统调用（`ecall` 指令）、发生非法操作，或者设备中断时，Trap可能发生。用户空间的陷阱处理流程是这样的：

1. 进入 **uservec** （位于 `kernel/trampoline.S:22`）。
2. 然后执行 **usertrap** （位于 `kernel/trap.c:37`）。
3. 返回时调用 **usertrapret** （位于 `kernel/trap.c:90`），最后由 **userret** （位于 `kernel/trampoline.S:101`）恢复到用户空间。

***

### 设计限制

一个重要的设计约束是：RISC-V 硬件在触发Trap时不会切换页表。这带来了两个要求：

1. `stvec` 中的指向Trap处理程序地址，必须在当前user page table中必须有效。因为在Trap处理代码开始执行的时候，仍然是用户页表生效。
2. 操作系统需要切换到 **内核页表**，以便能继续执行。而在那时，内核页表也需要映射 `stvec` 指向的Trap处理程序。



为了满足这些条件，Xv6 使用了 **trampoline page**：

* **Trampoline page** 包含 `uservec` 的代码（即 `stvec` 指向的Trap处理程序。），且被映射到每个进程的页表中虚拟地址 `TRAMPOLINE` 处（位于虚拟地址空间顶部）。
* **trampoline page** 被映射到每个进程的页表中的一个固定地址（`TRAMPOLINE`，位于虚拟地址空间的顶部，这样就不会与用户程序使用的内存地址重叠）。同样，**trampoline page** 也在内核页表中映射。

这样，在用户空间发生tarp时，tarp可以在监督模式下开始执行，因为 `trampoline page` 被映射到用户页表中。而当切换到内核页表后，trap处理程序可以继续执行。



***

**`uservec` 的处理过程**

`uservec` 位于 `trampoline.S` 文件（[`kernel/trampoline.S:22`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trampoline.S#L22)）。它的任务如下：

1. **保存用户寄存器**：
   * 当 `uservec` 开始执行时，所有 32 个寄存器的值均属于被中断的用户代码。为了返回到用户空间后可以恢复，这些寄存器需要保存在内存中。
   * 存储寄存器的值需要使用一个寄存器作为地址，而此时没有可用的通用寄存器。RISC-V 提供了 `sscratch` 寄存器来解决这个问题。
   * `uservec` 开始时通过 `csrw` 指令将 `a0` 的值保存到 `sscratch` 中。这样，`uservec` 就有了一个可用的寄存器 `a0`。
2. **接下来就是保存32位用户寄存器到** ：
   * 操作系统为每个进程都有一页内存用于存储一个 `trapframe` 结构（[`kernel/proc.h:43`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.h#L43)）。`trapframe` 包括保存 32 个用户寄存器的空间。
   * 每个用户进程都有自己的一页，用来存储它的状态。操作系统已经在进程的页表里“悄悄”安排好这个页面了，每个进程都能通过固定的虚拟地址（`0x3ffffffe000`）找到它。
   * 因为此时的 `satp` 仍然指向用户页表，`trapframe` 必须映射到用户地址空间。Xv6 将每个进程的 `trapframe` 映射到 `TRAPFRAME` 虚拟地址（位于 `TRAMPOLINE` 之下）。
3. **切换到内核页表**：
   * `uservec` 从 `trapframe` 中加载内核页表的地址，并切换 `satp` 到内核页表。
   * 接着，`uservec` 跳转到 `usertrap`。

***

**`usertrap` 的处理过程**

`usertrap` 的任务是分析Trap原因并处理（[`kernel/trap.c:37`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L37)）。它的主要步骤包括：

1. **调整Trap处理程序**：
   * 将 `stvec` 设置为 `kernelvec`，以便处理后续内核中的Trap。
2. **保存状态**：
   * 保存 `sepc`（用户程序计数器）以备后续返回。
   * 如果Trap是系统调用，调用 `syscall` 处理；如果是设备中断，调用 `devintr`；否则认为是异常，内核终止发生错误的进程。
3. **更新用户程序计数器**：
   * 对于系统调用的情况，`sepc` 的值增加 4，因为 RISC-V 的系统调用将 `pc` 留在 `ecall` 指令处，而用户代码需要从下一条指令继续执行。
4. **检查进程状态**：
   * 在返回前检查进程是否被终止或者是否需要让出 CPU（比如处理定时器中断时）。

***

**`usertrapret` 和返回用户空间**

1. **准备控制寄存器**：
   * 设置 `stvec` 为 `uservec`，以处理未来从用户空间发生的陷阱。
   * 设置 `sepc` 为之前保存的用户程序计数器。
2. **调用 `userret`**：
   * `usertrapret` 调用位于 trampoline 页中的 `userret`（[`kernel/trampoline.S:101`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L37)）。
   * `userret` 将 `satp` 切换回用户页表。
3. **恢复用户寄存器并返回**：
   * `userret` 使用 `TRAPFRAME` 地址恢复用户寄存器。
   * 最后执行 `sret` 指令返回到用户空间。

***

**关键机制**：Trampoline 页和 `trapframe` 是 Xv6 处理用户空间陷阱的核心设计，它们解决了硬件设计上的约束（如页表未切换）。

**模块化流程**：

1. `uservec` 负责从用户代码切换到内核代码。
2. `usertrap` 负责实际的Trap处理。&#x20;
3. `usertrapret` 和 `userret` 负责从内核返回用户空间。



## 5.3 code：系统调用执行过程

user调用`exec`执行system call的过程：

* 把给`exec`使用的参数放到a0和a1寄存器中，把system call的代码(SYS\_exec)放到a7寄存器中，执行ecall指令进入内核，切换到特权模式：
  * **`uservec`**：保存用户寄存器到进程的 `trapframe`。
  * **`usertrap`**：检测到这是系统调用后，调用 `syscall` 处理。
  * **`syscall`**：
    * 从 `trapframe` 中读取系统调用号（`a7` 寄存器的值）。
    * 根据系统调用号，在内核的 `syscalls` 函数指针表中找到对应的系统调用实现（如 `sys_exec`）。
    * 执行 `sys_exec` 函数。

返回值的处理

* 系统调用实现（如 `sys_exec`）返回值存储在 `p->trapframe->a0` 中。
* 返回值会传递给用户空间调用的返回值：
  * 若返回负数表示出错。
  * 若返回零或正数表示成功。

如果 `a7` 的值不在 `syscalls` 表中，`syscall` 函数会打印错误日志，并返回 `-1` 作为错误值。

**系统调用参数的解析方式**

内核需要从用户传入的寄存器中提取系统调用的参数。xv6 使用如下方法完成这一操作：

1. **参数的初始位置**\
   根据 RISC-V C 调用约定，用户传递的参数最初存储在寄存器中（如 `a0`、`a1` 等）。
2. **参数保存在 `trapframe`**\
   trap后，用户寄存器的值会保存在当前进程的 `trapframe` 中，内核可以从 `trapframe` 中读取这些参数。
3. **参数提取函数**\
   xv6 提供以下函数来从 `trapframe` 提取参数：
   * **`argint`**：提取整数类型参数。
   * **`argaddr`**：提取指针类型参数。
   * **`argfd`**：提取文件描述符类型参数。 这些函数的底层实现是 **`argraw`**，直接访问 `trapframe` 中保存的用户寄存器值。

## 5.4 Trap from kernel space

***

**1. 内核trap的入口：`usertrap` 和 `kernelvec`**

* `usertrap` 是处理用户空间到内核空间切换的入口函数。
* 在内核执行期间，trap向量寄存器 `stvec` 指向汇编代码 `kernelvec`，专门用于处理内核中的trap事件。
* 由于 `kernelvec` 只在内核中执行，它可以直接使用内核页表和内核栈。

***

**2. `kernelvec` 的作用**

* **保存寄存器状态**：将当前线程的所有32个寄存器压入内核栈，保护线程的执行状态。寄存器的value属于该线程。
* **准备后续处理**：保存完寄存器后，它会跳转到 C 函数 `kerneltrap`，进行具体的trap处理。

***

**3. `kerneltrap` 的分类处理**

`kerneltrap` 的任务是识别并处理trap的来源：

1. **设备中断**：调用 `devintr` 检查并处理硬件中断（如计时器中断）。
2. **异常**：如果trap不是设备中断，则可能是异常。在xv6中，异常（如非法操作）被认为是致命错误，直接调用 `panic` 停止内核。

***

**4. 计时器中断与线程调度**

* 如果计时器中断发生在内核线程执行期间，`kerneltrap` 会调用 `yield` 主动让出 CPU。
* 被让出的线程在稍后会被调度回来，继续执行其 `kerneltrap` 的剩余逻辑。

***

**5. 恢复到中断前的状态**

* `kerneltrap` 完成处理后，回到被tap中断的代码继续执行。由于 `yield` 可能会扰乱 `sepc` 和 `sstatus` 中保存的状态，`kerneltrap` 会在开始时保存这些状态，并在结束时恢复这些控制寄存器，然后返回到 `kernelvec`。
* `kernelvec` 从栈中弹出之前保存寄存器，最后通过 `sret` 将`sepc` 复制到pc，将控制权交还到被中断的代码位置，继续执行。

***

**6. 时间窗口问题**

* 从用户态切换到内核态时，存在一个短暂窗口，期间 `stvec` 尚未切换到 `kernelvec，仍然指向uservec`。
* 如果此时发生设备中断，可能导致问题。但 RISC-V 确保在处理trap时候中断，避免了这种风险。







