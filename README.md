# rCore-Learning-Record

## 2023.10.23

使用 ubuntu 22.04 搭建环境。

## 环境配置

安装 qemu 7.0

![image](https://github.com/StageGuard/rCore-Learning-Record/assets/45701251/020d3a0b-967f-48bb-ac0f-e36bc2c5165c)

尝试运行 chapter 1

![image](https://github.com/StageGuard/rCore-Learning-Record/assets/45701251/97eb3eb2-4978-402f-8984-a44013c7c597)

## 学习 chapter 3

在操作系统中，任务的切换是直接修改相关寄存器（sp 和 pc）来实现的，通过修改寄存器实现直接跳转到对应汇编指令。

当然用户程序和内核代码必须隔离运行，并且用户代码的致命异常不会导致内核停止工作，这就需要特权级切换。

用户程序想调用内核的功能必须通过 trap 实现，由内核来处理用户的 trap 信息，trap 流程完成后需要通过 `sret` 汇编指令来返回用户态。

trap 流程需要先保存用户的上下文环境（也就是相关寄存器和栈指针等），以便 `sret` 时可以恢复。

我初次看到 `__alltraps` 和 `__restore` 的时候还感到奇怪，有些地方调用了这个函数函数之后加上了 `panic("unreachable")`，为什么会这样呢？

仔细想了一下，这两个函数都是直接修改 CPU 寄存器来达到跳转的目的，自然不能按照一般编程的程序流思维来理解。

所以按照一般程序流来看，就好像程序流无缘无故跳转到另一个地方一样。

## 2023.10.24

理解了分时任务的实现，启用 `sie` 寄存器后用户态任务执行一段时间就会被 CPU 强制中断，触发 trap SupervisorTimer。

触发后只需要设置下次触发时间，然后切换任务即可。

完成 chapter 3 的作业，比较简单，在 `TaskControlBlock` 里加一个数组来记录就行了，然后在 trap_handler 来增加 syscall 记录。

`sys_task_info` 接受了一个裸指针 `*mut TaskInfo`，他不是胖指针，而且直接修改裸指针的内容也是 unsafe 的。

使用 Rust 写内核难免会涉及大量 unsafe 操作，但是我们只需要保证他是安全的，就不会出现问题。

## 2023.10.25 - 2023.10.27

看了看 chapter 4，感觉理解起来有些困难，特别是多级页表。所以花了比较长时间来一一理解。

首先定义了许多单个元素的 struct，例如 `PhysAddr` 和 `VirtAddr`，目的就是方便辨识转换。

一个任务的虚拟内存通过 `MemorySet` 管理，里面有页表和逻辑段。逻辑段是用户看到的虚拟空间。页表则保存了这些逻辑段映射的对应物理空间。

也就是说用户通过操作逻辑段中的虚拟地址来改变对应物理空间的地址，这一转换过程是由 CPU 实现的，当然在这之前需要通过写 `stap` 寄存器来启用分页特性。

在启用虚拟内存后，trap 流程也需要对应作出修改，目前是把用户的 trap context 保存在用户应用虚拟空间的内核栈中，相比于之前直接保存在内核的内核栈。

## 2023.10.28

完全理解了 rCore 的虚拟存储原理后，完成作业也就比较容易了。

`sys_mmap` 就是我们要在任务的 `MemorySet` 里创建新的逻辑段 `MapArea`，并把逻辑段加入到集合里，检查一下虚拟地址范围冲突即可。

`sys_get_time` 和 `sys_task_info` 传入的裸指针指向的地址现在是任务的虚拟空间地址，不是真实地址，所以我们需要查页表找到对应的物理块再修改。

## 2023.10.29

看了一下进程相关的文档，相比于上一章还是比较好理解的，把任务相关的一些资源封装到 `TaskControlBlock` 里就行了。

对于 `fork` 和 `exec` 的实现，刚开始我还不太懂为什么要这么写，后来仔细查了一下他们的语义才明白。

所以 `spawn` 的实现就是把 `TaskControlBlock.new` 的方法 copy 一份，然后处理一下父子进程关系就行了。

stride 实现也比较简单，在 `TaskControlBlockINner` 里加上 stride 和 priority 属性，然后把 `run_tasks` loop 里对将要被调度的任务的 stride += BIG_STRIDE / priority 就行了。

还需要添加一个 fetch_task_with_min_stride 方法来获取当前任务队列中 stride 最小的任务。

## 2023.10.30

今天比较忙没看代码

### 2023.10.31

今天看了 chapter 6，学习了 easy-fs 文件系统。

他的设计很巧妙，easy-fs 只包含文件系统的实现，提供了驱动接口 trait `BlockDevice` 来让外部程序实现。

也就是说只要是实现了 trait `BlockDevice` 的驱动就可以使用 easy-fs 文件系统，方便移植到别的平台，做到了低耦合。

easy-fs 文件系统也分成了好几层，最底层就是 BlockCache 层，在这一层将直接对 BlockDevice 进行读写和缓存操作。

网上一层就是 efs 层，这一层定义了文件索引，位图索引和磁盘节点数据结构，他们被写入到对应的块缓存里。

再往上就是 vfs 层，这一层把 efs 层的数据结构都统一封装为 Inode，进行读写和扩容操作，同时作业要求的 link 相关也准备在这一层实现。

### 2023.11.1

完成了 chapter 6 作业，实现了创建硬链接，解除硬链接和查询文件信息。

创建硬链接只需要在目录 `DiskInode` 写入链接目标的文件 `DiskInode` 索引即可，不需要创建新的文件。

在创建了文件的硬链接后，就有多个个文件索引指向同一个文件 `DiskInode`。

解除硬链接之前先获取所有指向这个文件 `DiskInode` 的硬链接，如果是只有一个硬链接，还需要在解除后删除文件内容。

解除硬链接的方式就是在目录 `DiskInode` 存储的对应文件索引处覆写 0 即可。

stat 的实现是遍历根目录所有文件找到 block_id 和 block_offset 和当前 Inode 一样的文件，获取到他的 inode_id，然后再用这个 inode_id 来查其他信息。

> 这个实现只是在这个作业中可以这么做，因为我们目前只有一级目录，也就是根目录。
>
> 如果是多级目录的话，那就是要遍历这个文件的父目录所有文件来找到对应的 inode_id 了。
>
> 而且硬链接的数量也可能需要另外一张表来存储，因为可以在不同层级的目录都链接这个文件，遍历查找显然很耗时间。
