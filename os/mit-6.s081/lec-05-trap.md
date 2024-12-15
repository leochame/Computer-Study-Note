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

### **Trap执行的核心部件**

**保存寄存器的地方：trapframe**

RISC-V中，<mark style="color:blue;">内核有一个名为</mark><mark style="color:blue;">**trapframe**</mark><mark style="color:blue;">的结构体，它负责保存用户寄存器的值。当</mark><mark style="color:blue;">`ecall`</mark><mark style="color:blue;">触发时，内核通过一个特定的内存区域（</mark><mark style="color:blue;">**trapframe page**</mark><mark style="color:blue;">）保存所有寄存器的值。这个区域通常会被映射到用户进程的</mark><mark style="color:blue;">**page table**</mark><mark style="color:blue;">中。</mark>

每个进程在其用户页表中都有一页专门存储寄存器信息。这个页面的虚拟地址是`0x3fffffe000`，它被用来存储寄存器值。内核的任务就是保存这些寄存器的值，以便后续恢复。用户寄存器的保存通过`trapframe`结构体完成，这个结构体包括了每个寄存器的保存槽位。内核事先设置好这个映射，让寄存器能够被保存和恢复。

#### **`sscratch`寄存器的作用**

RISC-V中有一个特殊的寄存器&#x53EB;**`sscratch`**  **，它是内核用来保存一个临时值的。内核在切换到用户空间之前，会将**trapframe page的地址**保存到`sscratch`中。这样，当`ecall`指令触发时，内核可以通过交换`a0`寄存器和`sscratch`寄存器的值，快速获得**trapframe page\*\*的虚拟地址，从而开始保存寄存器的内容。

#### **trampoline页的特殊性**

trampoline页是一个特殊的页，它在用户空间和内核空间中都有相同的映射。这意味着，当内核切换页表时，**trampoline页**的虚拟地址在两个页表中是相同的。因此，内核可以继续执行`trampoline.S`中的代码，而不会因为页表切换而导致崩溃。

***



### **`第一步：uservec` 的处理过程**

`uservec` 位于 `trampoline.S` 文件（[`kernel/trampoline.S:22`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trampoline.S#L22)）。它的任务如下：

**首先，保存用户寄存器**：

> 当 `uservec` 开始执行时，所有 32 个寄存器的值均属于被中断的用户代码。为了返回到用户空间后可以恢复，这些寄存器需要保存在内存中。
>
> 存储寄存器的值需要使用一个寄存器作为地址，而此时没有可用的通用寄存器。RISC-V 提供`sscratch` 寄存器来解决这个问题。

* `uservec` 开始时通过 `csrw` 指令将 `a0` 的值保存到 `sscratch` 中。这样，`uservec` 就有了一个可用的寄存器 `a0`。

**接下来，就是保存32位用户寄存器到 tarpframe**：

> 操作系统为每个进程都有一页内存用于存储一个 `trapframe` 结构（[`kernel/proc.h:43`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.h#L43)）。`trapframe` 包括保存 32 个用户寄存器的空间。
>
> 每个用户进程都有自己的一页，用来存储它的状态。操作系统已经在进程的页表里“悄悄”安排好这个页面了，每个进程都能通过固定的虚拟地址（`0x3ffffffe000`）找到它。
>
> 因为此时的 `satp` 仍然指向用户页表，`trapframe` 必须映射到用户地址空间。Xv6 将每个进程的 `trapframe` 映射到 `TRAPFRAME` 虚拟地址（位于 `TRAMPOLINE` 之下）。

`uservec` 将 `TRAPFRAME` 的地址加载到 `a0` 寄存器，并将每个用户寄存器的内容保存到这个位置，包括从 `sscratch` 中读取回来的 `a0` 值。



Trapframe 中还包含了当前进程的内核栈地址、当前 CPU 的 `hartid（CPU编号）`、`usertrap` 函数的地址，以及内核页表的地址。`uservec` 会读取这些值，将 `satp` 切换到内核页表，并跳转到 `usertrap。`这时内核开始执行实际的内核代码，处理系统调用、进程调度等任务。

***

### **`第二步：usertrap` 的处理过程**

`usertrap` 的任务是分析Trap原因并处理（[`kernel/trap.c:37`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L37)）。它的主要步骤包括：

1. **首先更新STEVC寄存器**：
   * `usertrap` 首先修改 `stvec`，使得在内核中发生陷阱时，应该由 `kernelvec` 处理，而不是 `uservec`。
2.  **获取当前进程信息：**

    程序会调用myproc函数获取当前正在执行的进程。myproc通过hartid（CPU的核心ID）来索引当前CPU核心上正在运行的进程。
3. **保存用户状态**：

> 用户程序执行时，CPU 会维护一个叫 `SEPC`（程序计数器）的寄存器，用来保存当前指令的地址。以便于后面恢复程序的执行。

* 在 `usertrap` 函数中，程序会将 `SEPC` 寄存器中的地址保存在 `trapframe` 结构中，这样可以避免在处理过程中丢失信息。

4\. **检查触发 `trap` 的原因**

* 通过检查 `SCAUSE` 寄存器，程序可以知道是何种类型的异常或中断触发了 `trap`。
* 如果Trap是系统调用，usertrap调用 `syscall` 处理；如果是设备中断，调用 `devintr`；否则认为是异常，内核终止发生错误的进程。

**5.更新用户程序计数器**：

* 对于系统调用的情况，`sepc` 的值增加 4，因为 RISC-V 的系统调用时会将 `pc` 指向`ecall` 指令处，而用户代码需要从接下来的指令继续执行。

**6.检查进程状态**：

* 在处理完这些任务后，`usertrap` 会检查进程是否被终止或是否需要让出 CPU（例如定时器中断）。

`最后，usertrap` 调用 `usertrapret` 函数，真正执行恢复工作。`usertrapret` 会将保存的 `SEPC`（程序计数器）值恢复到 CPU 中，并且恢复其他寄存器的值，确保用户程序能够从中断或系统调用中正确地恢复到原来被中断的位置。

***

### **`第三步：usertrapret` 和返回用户空间**

这是usertrapret的主要执行内容：

* **关闭中断**，防止在更新 `STVEC` 时出现意外中断。
* **更新 `STVEC`**，确保返回到用户空间时，能够跳转到正确的代码。
* **填充 `trapframe`**，确保进程的上下文在返回时能够恢复。
* **设置控制寄存器 `SSTATUS`**，确保 `sret` 指令返回到用户模式，并恢复中断。
* **切换页表**，确保用户空间使用正确的虚拟内存映射。
* **跳转到 `trampoline` 代码**，最终通过 `sret` 指令返回到用户空间。



0\. **关闭中断**

在返回用户空间之前，我们首先关闭中断，确保在更新 `STVEC` 寄存器时，不会发生意外的中断。

> 在返回到用户空间之前，内核需要更新 `STVEC` 寄存器（用来存储中断处理函数的地址）。如果此时发生了中断，可能会导致程序错误，因为它会跳转到用户空间的中断处理代码，而实际上我们仍然在内核中执行。

1. **更新STVEC:**&#x75;sertrapret会设置 `stvec` 为 更新为指向 `trampoline` 代码的地址。

> （`Trampoline` 是一段特殊的汇编代码，负责完成用户空间和内核空间的切换工作。）确保当 `sret` 指令执行时，能够顺利跳转到用户空间。

2. **填充 `trapframe` 结构：**`usertrapret` 在返回之前会对 `trapframe` 结构中的一些字段进行填充，为即将跳转到用户空间做准备。这些操作确保当用户进程从内核返回时，它能够恢复正确的上下文，包括寄存器、内存映射等。

> 主要的填充项包括：
>
> **Kernel page table 的指针**：确保我们在返回到用户空间时能够正确使用用户空间的页表。
>
> **当前进程的内核栈**：这允许用户进程恢复执行时能够访问其栈。
>
> **`usertrap` 函数的指针**：在 `trampoline` 代码执行时，能正确跳转到 `usertrap` 函数。
>
> **CPU 核编号**：`tp` 寄存器保存当前 CPU 核编号，`usertrapret` 会将其存储到 `trapframe` 中，以便恢复。

**3.设置 `SSTATUS` 寄存器:：**

> * **`SPP`**：这个位控制 `sret` 指令的行为。`SPP` 为 0 时，表示下一次执行 `sret` 时要返回到用户模式；为 1 时，表示返回到内核模式。
> * **`SPIE`**：这个位控制在执行完 `sret` 指令后，是否重新打开中断。因为我们希望在返回用户空间后继续响应中断，因此需要将 `SPIE` 设置为 1。

4. **设置指令计数器**

用于返回到用户模式。执行 `sret` 后，CPU 会使用 `SEPC` 寄存器中的地址作为程序计数器（PC）的值，并跳转回用户代码。`usertrapret` 通过设置 `SEPC` 来确保程序跳转到用户空间时，指令计数器指向正确的位置。

5. **切换页表**

在返回到用户空间之前，操作系统还需要切换页表。这是因为用户空间和内核空间通常有不同的虚拟地址空间。`usertrapret` 在这里准备了用户页表的指针，并将它作为参数传递给汇编代码中的 `trampoline`。在 `trampoline` 中，`userret` 函数会执行页表切换，确保用户空间能够使用正确的页表。



`最后，usertrapret` 计算出跳转到 `trampoline` 中的 `userret` 函数的地址。`userret` 函数负责执行用户空间的恢复工作，最终会通过 `sret` 指令将控制权交给用户程序。

`usertrapret` 会通过函数指针调用 `userret` 函数，传递两个参数：`userret` 函数的地址和页表指针。这些参数存储在 `a0` 和 `a1` 寄存器中。



***

### 第四步：userret执行用户空间恢复工作

* **切换页表**：切换到用户页表，确保用户空间能够正确访问内存。
* **恢复寄存器状态**：通过交换和加载寄存器值，恢复用户程序的上下文。
* **执行 `sret`**：通过 `sret` 指令恢复用户程序的状态，重新启用中断并返回用户模式。
* **返回用户空间**：最终跳转回用户程序，继续执行系统调用后续的操作。



1. 切换用户页表

内核通过 `csrw satp, a1` 指令将用户页表的指针存入 `SATP` 寄存器，`a1` 中存储的就是用户页表的地址。这个操作切换了页表，确保接下来的内存访问会按照用户空间的映射进行，而不是内核空间的映射。幸运的是，`trampoline` 页表本身也映射到用户空间的地址，这样即便页表切换了，`trampoline` 代码仍然能继续执行。

2\. **交换`SSCRATCH` 和 `a0` 寄存器**

* **`SSCRATCH` 寄存器**：用来保存内核使用的临时数据。
* **`a0` 寄存器**：在 `trapframe` 中保存了某些数据，特别是系统调用的返回值。

在 `uservec` 中，操作系统通过交换 `SSCRATCH` 和 `a0` 的值，恢复了原本保存在 `trapframe` 中的**所有**寄存器值到对应的寄存器中。

接下来的操作是将 `trapframe` 中保存的寄存器值恢复到相应的寄存器中。这些寄存器包括用户程序的栈指针、寄存器值等。至此，内核的操作基本完成，用户的上下文状态已经恢复，离返回用户空间只差一步。

#### 4. **`sret` 指令**

当所有的寄存器值恢复完毕后，最后一步是执行 `sret` 指令。`sret` 是一个特殊的指令，它执行以下操作：

* **恢复程序计数器（PC）**：`sret` 会将 `SEPC` 寄存器中的值（即用户程序上次运行的地址）复制到程序计数器（PC）寄存器中。这意味着程序会从之前中断或系统调用的位置继续执行。
* **切换到用户模式**：执行 `sret` 时，CPU 会根据 `SSTATUS` 寄存器的设置，将执行模式切换到用户模式（`SSTATUS` 的 `SPP` 位决定了是否切换到用户模式）。
* **重新打开中断**：在返回用户空间时，我们希望能够继续响应中断。`sret` 会重新启用中断。

通过 `sret` 指令的执行，程序从内核模式切换回用户模式，并继续执行原来的用户代码。

#### 5. **恢复用户程序**

程序现在已经返回到用户空间。由于 `SEPC` 中保存的是 `write` 函数的返回地址，执行 `ret` 指令时，程序将跳回到用户程序中，从 `write` 系统调用返回到 `Shell` 中，或者从触发系统调用的库函数返回到 `Shell` 中。

#### 6. **中断处理**

`professor` 还解释了在 `sret` 中断的特殊处理：

* 在 `sret` 执行时，操作系统会重新启用中断。返回用户模式后，程序可能会遇到一些硬件中断，例如磁盘中断、时钟中断等，操作系统必须能够处理这些中断。
* 重新打开中断意味着在用户空间执行时，程序可以处理这些外部事件。





1. **调用 `userret`**：
   * `usertrapret` 调用位于 trampoline 页中的 `userret`（[`kernel/trampoline.S:101`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c#L37)）。这时候，会将进程的用户页表指针作为参数传递给 `a0`
   * `userret` 将 `satp` 切换回进程用户页表。**用户页表映射了 trampoline page 和 trapframe，而内核的内容不在其中。**
2. **userret切换到用户页表后，开始恢复用户寄存器和trapframe并返回**：
   * `userret` 使用 `TRAPFRAME` 地址恢复用户寄存器。会加载 `TRAPFRAME` 的地址到 `a0`，并从中恢复用户寄存器的内容，包括之前保存的 `a0`。
   * 最后执行 `sret` 指令返回到用户空间。

***

**关键机制**：<mark style="color:blue;">Trampoline 页和</mark> <mark style="color:blue;"></mark><mark style="color:blue;">`trapframe`</mark> <mark style="color:blue;"></mark><mark style="color:blue;">是 Xv6 处理用户空间Trao的核心设计，它们解决了硬件设计上的约束（如页表未切换）。</mark>

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







