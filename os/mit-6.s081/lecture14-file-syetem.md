# Lecture14：File Syetem

文件系统是一个位于磁盘的数结构，我们带着关于设计文件系统的两个重要目的和实现操作系统的四个挑战来看看这个文件系统是怎么实现的。

两个重要目的：

* **组织和存储数据**：文件系统将磁盘上的数据以结构化的方式存储，并提供高效的方式来读取和写入数据。
* **数据共享与持久化**：允许多个用户和应用程序共享数据，同时确保即使发生重启或崩溃，数据依然能够持久保存。

四个挑战：

* **磁盘块的管理**： 文件系统需要在磁盘上使用数据结构来表示目录和文件的树形结构，记录每个文件内容所在的磁盘块的标识符，以及记录哪些磁盘区域是空闲的。
* **崩溃恢复**：如果在文件系统操作过程中发生崩溃（如电源故障），文件系统必须能够恢复到一致的状态，避免数据损坏。
* **并发访问**：多个进程可能会同时访问文件系统，因此必须保证文件系统的一致性和数据的正确性。
* **缓存优化**：磁盘访问速度远低于内存，因此文件系统需要利用内存缓存机制来提升性能。

### 常见的 关于 File 的系统调用 <a href="#yqzbn" id="yqzbn"></a>

我们先来从常用的文件操作的命令行来了解 File

```c
fd = open("x/y",-);
write(fd,"abc",3)
link("x/y","x/z")
```

上面的命令是系统调用创建文件，并返回文件描述符给调用者。调用者会对文件描述符调用 `write`。

我们可以看到，接口中的路径名字是可读的字符串形式。`write`中并没有 offset，也就是说**写入到文件的哪个位置是由文件系统决定。**

我们也可以调用link系统调用，为之前创建的文件“x/y”创建另一个名字“x/z”，所以**文件系统内部需要以某种方式跟踪指向同一个文件的多个文件名**。

另外我们也可以删除更新文件的命名空间，比如使用 `unlink`系统调用，但是文件还是可以正常接收数据，说明**文件对象不依赖文件名。**

其实，文件系统的目的是实现上面描述的 API。比如，数据库也可以持久化存储数据，但是数据库提供了和文件系统不一样的 API。

### 文件系统层级 <a href="#s3lmu" id="s3lmu"></a>

xv6 文件系统是由七个层次组成的，每一层处理不同的功能。

<figure><img src="../../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

**（1) 磁盘层 (Disk Layer)**

磁盘层直接与磁盘硬件交互，负责读写磁盘上的数据。xv6 使用的是 **virtio 硬盘**，它将数据划分为固定大小的磁盘块（通常是 512 字节一个块）

**(2) 缓冲区缓存层 (Buffer Cache Layer)**

缓冲区缓存层的任务是：

* **同步磁盘块的访问**：确保同一个磁盘块在内存中只有一个副本，且只有一个内核线程能在任何时候修改该副本。
* **缓存热点数据块**：为了减少对磁盘的访问频率，缓存常用的磁盘块。

在 xv6 中，内存中的磁盘块被缓存为 **buffer**（缓冲区）。这些缓冲区的管理依赖于锁机制，确保多个进程不会同时访问同一缓冲区。每次访问磁盘块时，缓冲区缓存层会通过 `bread` 或 `bwrite` 函数进行操作。`bread` 用于读取磁盘块并将其缓存到内存中，`bwrite` 用于将修改过的缓存块写回磁盘。

**(3) 日志层 (Logging Layer)**

日志层是为了实现 **崩溃恢复**，确保文件系统在发生崩溃后能够恢复一致的状态。每次文件系统操作（例如修改文件内容）不会直接写入磁盘，而是首先记录在日志中。日志记录了操作的所有写入信息。

日志分为两个步骤：

* **写日志**：文件系统操作首先将所有修改记录到日志中。
* **提交日志**：操作完成后，系统会在日志中写入一个特殊的提交记录，表示这次操作已完成。

崩溃恢复的流程是：

1. 如果崩溃发生在操作提交之前，恢复时会忽略这部分日志，确保文件系统恢复到崩溃前的状态。
2. 如果崩溃发生在操作提交之后，恢复时会重新执行日志中的操作，确保文件系统数据一致。

**(4) Inode 层 (Inode Layer)**

在 xv6 中，每个文件都有一个唯一的标识符，叫做 **i-number**，这个标识符在文件系统中是唯一的。每个文件的元数据（如大小、创建时间、块指针等）存储在 **inode** 中。文件的数据内容（如文件的字节）存储在磁盘的 **数据块** 中。

Inode 是 xv6 文件系统的核心，它代表了一个文件或者目录。每个 inode 由多个数据块组成，inode 的内容包括文件的属性、权限、指向数据块的指针等。

实现了 read 和 write。

inode 有一个 link count 来跟踪指向这个 inode 的文件名的数量，还有一个 openfd count 代表当前打开文件的文件的文件描述符计数。一个文件只有两者都为 0 才会删除。

**(5) 目录层 (Directory Layer)**

目录在 xv6 中也是一种特殊的 inode。每个目录包含多个目录项，每个目录项都包含**文件的名称和对应的 inode 编号**。通过目录，用户可以根据文件名来查找文件对应的 inode。

目录通过递归的方式实现路径解析，例如 `/usr/rtm/xv6/fs.c`，文件系统会逐级解析目录，最终找到对应的 inode。

**(6) 路径名层 (Pathname Layer)**

路径名层负责处理文件的路径，支持文件路径的解析和查找。路径名解析是递归的，系统会从根目录开始，逐级访问每个目录，直到找到目标文件。

**(7) 文件描述符层 (File Descriptor Layer)**

文件描述符层是操作系统用来抽象文件的接口，通过文件描述符来管理不同类型的资源（例如文件、管道、设备等）。在 xv6 中，文件描述符为进程提供了对文件的抽象，应用程序可以通过文件描述符进行文件的读取、写入等操作。

### 文件系统的磁盘布局 <a href="#lxlcc" id="lxlcc"></a>

磁盘硬件传统上将磁盘上的数据表示为 512 字节块（也称为扇区，sectors）的编号序列：扇区 0 是前 512 字节，扇区 1 是下一个字节，依此类推。操作系统用于其文件系统的块大小可能与磁盘使用的扇区大小不同，但通常块大小是扇区大小的倍数。

有时候人们也会把 sectors 称为 block，但 blocks 是操作系统中的视角，一般是 1024 字节，所以这里的术语也不是很准确。

Xv6 在 struct buf 类型的对象中保存已读入内存的块的副本。

此结构中存储的数据有时与磁盘不同步：它可能尚未从磁盘读取（磁盘正在处理它，但尚未返回扇区的内容），或者它可能已被更新软件但尚未写入磁盘。

文件系统必须计划在磁盘上存储 inode 和内容块的位置。为此，xv6 将磁盘分为几个部分：

<figure><img src="../../.gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

* **第 0 块**：引导扇区（boot sector），包含启动信息。主要用作启动操作系统
* **第 1 块**：超级块（superblock），包含文件系统的元数据，例如文件系统的大小、数据块数、inode 数等。构造出大部分文件系统的信息。
* **第 2 块-31 块**：日志（log），用于记录文件系统操作的事务日志。
* **32 块到 45 块**：存储 inode，每个 inode 存储一个文件或目录的元数据。64 字节的数据结构。
* **位图块（bitmap blocks）**：用来标记哪些数据块是空闲的，哪些是已经分配的。
* **数据块**：存储文件和目录的内容。

<figure><img src="../../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

#### 磁盘上存储 inode 到底是什么？ <a href="#qhwl9" id="qhwl9"></a>

**File type：**&#x8868;明 node 是文件还是目录’

**nlink:**&#x8DDF;踪多少文件名指向 inode

**size:**&#x8868;明文件数据多少字节

在 xv6 中，每个 `inode` 包含 12 个直接块编号（Direct Block Numbers）。这些编号直接指向磁盘上的数据块位置。如果文件的大小超过了 12 个块，`inode` 会使用一个 **间接块编号（Indirect Block Number）**，该间接块会指向另外一个块，这个块中保存了更多数据块的编号。

<figure><img src="../../.gitbook/assets/image (15).png" alt="" width="375"><figcaption><p>The representation of file on disk</p></figcaption></figure>

总之， inode中的信息完全足够用来实现read/write系统调用，至少可以找到哪个disk block需要用来执行read/write系统调用。

#### Directory 是怎么实现的？ <a href="#i9anf" id="i9anf"></a>

**目录本质**：目录本质上是文件加上文件系统可以理解的结构，每一个目录包含 **目录条目（directory entries）**。 每一个 entry 都有固定的结构，在 xv6 中使用 16 个字节实现：

* **前 2 字节**：存储文件或子目录的 `inode` 编号。
* **后 14 字节**：存储文件或子目录名。

***

那么路径名查找过程是怎样的呢？

以路径 `/y/x` 为例：

1. 从路径名看出**从根目录开始查找**：

*
  * 根目录的 `inode` 编号通常固定为 1（在 XV6 中）。
  * 根据 `inode` 编号定位到根目录对应的磁盘块。XV6 的 inode 块是从 block32 开始，每个占用 64 字节，所以 编号为 `1` 的 inode 在 **block 32 的 64 到 128 字节** 之间。

2. **查找目录条目**：

*
  * 线性扫描根目录的数据块，找到名称为 `y` 的条目，并获取其 `inode` 编号。也就是读取所有的direct block number和indirect block number。

3. **递归查找子目录**：

*
  * 使用 `inode` 编号读取子目录 `y` 的数据块，再次扫描找到名称为 `x` 的条目。

4. **返回结果**：

*
  * 找到 `x` 的 `inode` 编号后，返回结果供进一步操作（如读取文件数据）。

如何判断 `inode` 是文件还是目录？

`inode` 结构中包含一个 `type` 字段，会表示是文件还是目录类型。

在 XV6 中，目录查找是线性扫描，效率较低。这种简单实现的优点是易于理解，但在实际应用中存在显著性能问题：

* 使用更高效的数据结构，如哈希表、红黑树或 B 树，以加速目录条目查找。
* 现代文件系统（如 EXT4）使用 B 树等复杂结构优化目录查找性能。

### File System 是怎么工作的 <a href="#ies3j" id="ies3j"></a>

启动XV6的过程中，调用了makefs指令，makefs创建了一个全新的磁盘镜像，在这个磁盘镜像中包含了我们在指令中传入的一些文件。makefs为你创建了一个包含这些文件的新的文件系统。

在 XV6 的文件系统中，磁盘被划分为多个块（blocks），这些块分成了不同的类型：

1. **Meta Blocks**（元数据块）：包括启动块（boot block）、超级块（super block）、日志块（log blocks）、inode 块、bitmap 块。
2. **Data Blocks**（数据块）：用来存储文件的实际数据。

在这里教授为了方便我们理解，会打印出关于 block 的日志：

我们通过echo “hi” > x，来创建一个文件x，并写入字符“hi”。

这里会有几个阶段

1. 第一阶段是创建文件
2. 第二阶段将“hi”写入文件
3. 第三阶段将“\n”换行符写入到文件

<figure><img src="../../.gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

结合磁盘分布图来看讲解：

**第一阶段：**&#x547D;令 `echo "hi" > x` 的第一步是创建一个名为 `x` 的文件 。

过程如下：

* **分配一个空闲 inode，并且分两次写入 ：【** Block 33】
  * 第一次写入：修改该 inode 的 type 字段（从空闲变为文件）。
  * 第二次写入：写入 inode 的详细信息（如文件类型、初始链接数、大小为 0 等）。
* **更新根目录的内容**：【 Block46】
  * 向根目录的第一个 block 添加一个目录项，包含文件名 `x` 和分配给 `x` 的 inode 编号。
* **更新根目录的 inode：【Block 32**】
  * 根目录的 inode 的大小发生变化，因为增加了 16 字节的目录项（包含文件名和 inode 编号）。
* **再次更新文件 `x` 的 inode：【Block 33**】
  * 为文件 `x` 的 inode 做了初步的设置，但此时文件还没有任何数据。

**第二阶段：写入“hi”到文件 : 在这一阶段，文件系统将字符 "hi" 写入文件 `x` 中**

* **更新 bitmap:** 【Block 45】
  * 在 bitmap 中找到一个空闲数据块，设置对应 bit 为 1，表示该数据块已被占用。
* **写入数据到文件数据块：【**&#x42;lock59&#x35;**】**
  * 将字符 `h` 和 `i` 分别写入数据块 595。
* **更新文件 inode：【**&#x42;lock 33】
  * 更新文件 `x` 的 inode 中的 `size` 字段（从 0 更新为 2，因为增加了三个新的字符）。
  * 设置文件数据块（direct block）的地址为 595。

<figure><img src="../../.gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure>

### Code：XV6 创建 inode <a href="#od7cr" id="od7cr"></a>

分配 inode 发生在 `sys_open`函数中，这个函数负责创建文件。

```c
uint64
sys_open(void)
{
    char path[MAXPATH];
    int fd, omode;
    struct file *f;
    struct inode *ip;
    int n;

    argint(1, &omode);
    if((n = argstr(0, path, MAXPATH)) < 0)
        return -1;

    begin_op();

    if(omode & O_CREATE){
        ip = create(path, T_FILE, 0, 0);
        if(ip == 0){
            end_op();
            return -1;
        }
    } 
}
```

我们可以看到 `sys_open`调用了 `create`函数。在 `create` 函数中会先解析路径名并找到最后一个目录，如果文件存在就会调用`ialloc`函数。

```c
static struct inode*
create(char *path, short type, short major, short minor)
{
  struct inode *ip, *dp;
  char name[DIRSIZ];

  if((dp = nameiparent(path, name)) == 0)
    return 0;

  ilock(dp);

  if((ip = dirlookup(dp, name, 0)) != 0){
    iunlockput(dp);
    ilock(ip);
    if(type == T_FILE && (ip->type == T_FILE || ip->type == T_DEVICE))
      return ip;
    iunlockput(ip);
    return 0;
  }

  if((ip = ialloc(dp->dev, type)) == 0){
    iunlockput(dp);
    return 0;
  }

  ilock(ip);
  ip->major = major;
  ip->minor = minor;
  ip->nlink = 1;
  iupdate(ip);

  if(type == T_DIR){  // Create . and .. entries.
    // No ip->nlink++ for ".": avoid cyclic ref count.
    if(dirlink(ip, ".", ip->inum) < 0 || dirlink(ip, "..", dp->inum) < 0)
      goto fail;
  }

  if(dirlink(dp, name, ip->inum) < 0)
    goto fail;

  if(type == T_DIR){
    // now that success is guaranteed:
    dp->nlink++;  // for ".."
    iupdate(dp);
  }

  iunlockput(dp);

  return ip;

 fail:
  // something went wrong. de-allocate ip.
  ip->nlink = 0;
  iupdate(ip);
  iunlockput(ip);
  iunlockput(dp);
  return 0;
}
```

`ialloc`函数会为文件 x 分配 inode。它通过遍历所有的 inode 编号，扎到 inode 对应的编号，查看 type 字段是否是 free，是的话就设置为文件，这就等于标记上已经分配了

```c
// Allocate an inode on device dev.
// Mark it as allocated by  giving it type type.
// Returns an unlocked but allocated and referenced inode,
// or NULL if there is no free inode.
struct inode*
ialloc(uint dev, short type)
{
  int inum;
  struct buf *bp;
  struct dinode *dip;

  for(inum = 1; inum < sb.ninodes; inum++){
    bp = bread(dev, IBLOCK(inum, sb));
    dip = (struct dinode*)bp->data + inum%IPB;
    if(dip->type == 0){  // a free inode
      memset(dip, 0, sizeof(*dip));
      dip->type = type;
      log_write(bp);   // mark it allocated on the disk
      brelse(bp);
      return iget(dev, inum);
    }
    brelse(bp);
  }
  printf("ialloc: no inodes\n");
  return 0;
}
```

如果多个进程调用 `ialloc`函数会怎样？

我们知道 `ialloc`会调用 `bread`，`bread`会调用`bget`函数，`bget`会为我们从 buffer cache 中找到 block 缓存。

<figure><img src="../../.gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

`bget` 函数负责在缓冲区缓存中查找指定设备（`dev`）的特定块（`blockno`）。

* 获取缓存锁
* 如果找到目标块：
  * 增加该缓冲区的引用计数（`refcnt++`），表示该缓冲区正被使用。
  * 释放全局缓存锁（`bcache.lock`），允许其他线程访问缓存。
  * 获取缓冲区的睡眠锁（`acquiresleep(&b->lock)`，本质上来说它获取block 33 cache的锁），保证该缓冲区在使用期间不被其他线程修改。
  * 返回锁定的缓冲区指针。
* 未命中缓存：回收最近最少使用（LRU）的缓冲区
  * 如果目标块不在缓存中，函数会尝试回收一个未被使用的缓冲区（`refcnt == 0`）。
  * 遍历链表时从最近最少使用的缓冲区（链表尾部）开始，这是一种 **LRU 替换策略**。
  * 找到一个未使用的缓冲区，也会进行上述的类似操作，这里就不再赘述。 但这里说明下`valid = 0`是表示数据尚未从磁盘读取。

```c
// Look through buffer cache for block on device dev.
// If not found, allocate a buffer.
// In either case, return locked buffer.
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;

  acquire(&bcache.lock);

  // Is the block already cached?
  for(b = bcache.head.next; b != &bcache.head; b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // Not cached.
  // Recycle the least recently used (LRU) unused buffer.
  for(b = bcache.head.prev; b != &bcache.head; b = b->prev){
    if(b->refcnt == 0) {
      b->dev = dev;
      b->blockno = blockno;
      b->valid = 0;
      b->refcnt = 1;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }
  panic("bget: no buffers");
}
```

### SleepLock <a href="#epunr" id="epunr"></a>

```c
void acquiresleep(struct sleeplock *lk)
{
  acquire(&lk->lk);                // 1. 获取普通的spinlock
  while (lk->locked) {             // 2. 如果sleeplock已被锁定
    sleep(lk, &lk->lk);            // 3. 休眠并释放spinlock，等待被唤醒
  }
  lk->locked = 1;                  // 4. 设置sleeplock为已锁定状态
  lk->pid = myproc()->pid;         // 5. 记录持锁进程的PID
  release(&lk->lk);                // 6. 释放spinlock
}
```

`acquiresleep`函数是获取`sleep lock`的核心函数。它的工作流程如下：

1. **获取普通的`spinlock`**：首先，通过`spinlock`来确保`sleep lock`的互斥性。
2. **检查`sleep lock`的状态**：如果`sleep lock`已经被持有，当前进程就会进入休眠状态，等待被唤醒。
3. **释放当前CPU的控制权**：在休眠状态下，当前进程的CPU控制权会被释放，其他进程（或CPU核）可以继续执行。

**为什么使用`sleeplock`而非`spinlock`？**

* **磁盘操作的时间消耗**：磁盘操作通常非常耗时，尤其是在单核系统中。对于`spinlock`，如果一个线程在进行磁盘操作时持有锁，它会一直占用CPU，而不会释放CPU给其他进程。这在单核系统中会导致一个问题：即使另一个进程（比如读取磁盘数据的中断处理程序）可以获得CPU，当前线程也无法释放CPU，因此无法继续读取数据。
* **`spinlock`的限制**：`spinlock`通常需要关闭中断，防止中断处理程序（比如磁盘I/O）在锁被持有时改变锁的状态。由于磁盘I/O操作可能需要较长时间，如果线程在持有`spinlock`的同时关闭了中断，它就不能接收来自磁盘的数据。与此同时，系统中的其他CPU核如果需要处理其他工作，它们也不能获得CPU时间。因此，`spinlock`在I/O密集型任务中使用不当，会造成资源浪费。
* **`sleep lock`的优势**：与`spinlock`不同，`sleep lock`允许我们在等待锁的过程中不关闭中断。也就是说，在磁盘操作期间，当前进程仍然可以等待磁盘数据的到来，同时也允许其他进程（或CPU核）获得CPU执行。进程持有`sleep lock`时可以进入休眠状态，从而释放CPU时间，并避免“空转”（spin）浪费计算资源。

#### brelse <a href="#vyhde" id="vyhde"></a>

```c
// Release a locked buffer.
// Move to the head of the most-recently-used list.
void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  acquire(&bcache.lock);
  b->refcnt--;
  if (b->refcnt == 0) {
    // no one is waiting for it.
    b->next->prev = b->prev;
    b->prev->next = b->next;
    b->next = bcache.head.next;
    b->prev = &bcache.head;
    bcache.head.next->prev = b;
    bcache.head.next = b;
  }
  
  release(&bcache.lock);
}
```

首先，`brelease`函数释放`sleeplock`，即解除当前线程对某个block的锁定。允许其他线程访问该`block cache`，获取锁并进行相应操作。这样，其他进程或线程能够在持有`sleeplock`的线程进入休眠后访问同一`block cache`。

接下来，函数获取`bcache`的锁。`bcache`我们之前提到过是一个全局的缓存结构，保存了多个`block cache`的引用。

`b->refcnt--`当一个进程或线程不再使用某个`block cache`时，它会减少引用计数。当引用计数减为0时，意味着没有任何进程或线程正在使用该`block cache`，此时可以考虑将其从缓存中移除或者进行其他操作。

紧接着就是进行 LRU 操作

最后，函数释放`bcache.lock`，这样其他线程可以继续操作`bcache`。
