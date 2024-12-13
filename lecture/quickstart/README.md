---
icon: bullseye-arrow
---

# MIT 6.S081



## Introduction <a href="#tst9g" id="tst9g"></a>

这是本人对 MIT 的 OS 公开课程的笔记，本人英语能力有限，课程视频并未采用最新的 2024，而是去看的是 2022 的视频，结合了网上的诸多开源资料：

MIT 6.S081 Schedule：[6.1810 / Fall 2024 (mit.edu)](https://pdos.csail.mit.edu/6.S081/2024/schedule.html)

樊潇老师的笔记：[MIT 6.S081 Lecture Notes - Xiao Fan's Personal Page (fanxiao.tech)](https://fanxiao.tech/posts/2021-03-02-mit-6s081-notes/#11-processes-and-memory)

视频：[【操作系统工程】精译【MIT 公开课 MIT6.S081】\_哔哩哔哩\_bilibili](https://www.bilibili.com/video/BV1rS4y1n7y1/?spm_id_from=333.337.search-card.all.click\&vd_source=09b781e37940ba73b70deb9e59824d33)

中文文档：[简介 | MIT6.S081 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081)



如何调试：\
首先执行

```
make CPUS=1 qemu-gdb
```

在另一个窗口执行

1\.

```
riscv-64-unknown-elf-gdb
```

```
tui enable
```

berek 打断点

```
b
```

继续运行，continue

```
c
```



得到源码窗口

```
layout sources
```



得到汇编指令

```
layout asm
```

展示c语言代码

```
layout source
```

得到源码和汇编窗口

```
layout split
```







2\.

```
gdb-multiarch kernel/kernel
```

```
target remote localhost:25000
```

```
layout split
```



