---
layout: post
title: 从xv6 OS看文件系统的实现
---

* contents
{:toc}


文件系统是一个操作系统中的重要组成部分，提供了数据持久化存储到磁盘上的方法。与内存管理模块类似，文件系统提供了管理磁盘空间的一组方法与数据结构，基本上有四方面要素：1）要有地方存metadata；2）要有地方记录block使用情况；3）要有地方记录文件信息；4）要有地方记录文件索引信息。当然不同的文件系统实现对这四方面也有不同的理解，UNIX世界文件系统基本是按照这种模型实现的，superblock（metadata），block map（block使用情况），inodes（文件信息），directories（文件索引）。

本文将根据xv6 OS（一个开源教学操作系统，以下称xv6）实现的文件系统来看下文件系统在操作系统中的作用及其简单实现。



## 概述

xv6的文件系统是一种类UNIX的文件系统（以下称xv6fs），主要实现了这几个目标：

- 提供文件和目录的命名机制
- 提供系统崩溃数据恢复功能
- 多进程并发访问
- 磁盘数据缓存

xv6fs逻辑上分为七层，如 图1 所示，这也是很多系统的做法。并且为了方便实现这些逻辑层，文件系统为磁盘划分了许多区域，如 图2 所示。0号boot块为系统启动扇区，1号为superblock，存放文件系统元信息，包括inode数量，磁盘信息等；从block 2开始是log区，存放文件系统日志；之后是inode区，存放所有文件的inode；bit map记录所有block使用情况；最后是data区，存放实际数据。

![](https://ws3.sinaimg.cn/large/006tNc79gy1fh4d7ve6wvj307l085q38.jpg)图 1



![](https://ws2.sinaimg.cn/large/006tNc79gy1fh4djxiolmj30gf0290sy.jpg)图 2



接下来分别介绍这几层逻辑实现，Disk是与硬件驱动相关的一层，这里将不讨论。



## Buffer Cache

作为文件系统最低一层逻辑，buffer cache主要有两项功能**1）提供磁盘block的同步访问；2）作为磁盘block在内存中的缓存。**与Buffer Cache相关的源文件有bio.c，buf.h，ide.c。

该层的关键数据结构有：

```c
struct buf {
  int flags;
  uint dev;
  uint blockno;
  struct buf *prev; // LRU cache list
  struct buf *next;
  struct buf *qnext; // disk queue
  uchar data[BSIZE];
};
#define B_BUSY  0x1  // buffer is locked by some process
#define B_VALID 0x2  // buffer has been read from disk
#define B_DIRTY 0x4  // buffer needs to be written to disk

struct {
  struct spinlock lock;
  struct buf buf[NBUF];

  // Linked list of all buffers, through prev/next.
  // head.next is most recently used.
  struct buf head;
} bcache;
```



在xv6初始化的时候会创建固定数量的buf存放在bcache中，通过bread()和bwrite()实现block读写操作。

buf有三种标志位记录当前的状态信息，`B_BUSY` 表示该block正在进行读写操作，`B_VALID` 表示该buffer含有有效的磁盘数据，`B_DIRTY`  表示该buffer中的数据已被修改需要被写到磁盘中。xv6fs通过这三种标记位，以及bcache中的自旋锁，保证磁盘数据一致性、数据同步。

bget()从bcache中寻找相关dev第一个空闲，即非 `B_BUSY` 的buf，标记为 `B_BYSY` 并返回。如果没有找到则遍历寻找所有空闲buffer并且buffer为“CLEAN”即非 `B_DIRTY` 的buf，做相应标记后返回。

bread()调用bget()获取有效buf后调用驱动程序将buf的数据区填满并返回，bwrite()将buf的数据刷到磁盘，并清空 `B_DIRTY` 位。所有调用bread()或bwrite()之后都须调用brelse()将该buf添加到bcache队头调整使用率，同时唤醒其它等待该buf的进程。因为其它进程在bget()没有找到自己所需的block时会将自己休眠并让出CPU。



## Logging

Log主要是用来实现系统崩溃后的重启数据恢复，数据先写到log区，commit后复制到数据区，然后从log区中删除，任一阶段系统崩溃重启后，会先检查log区，将存在的完整的commit复制到数据区然后从log区删除。这样保证了数据的完整一致性。log层代码在log.c中。

该层关键数据结构有：

```c
struct logheader {
  int n;
  int block[LOGSIZE];
};

struct log {
  struct spinlock lock;
  int start;
  int size;
  int outstanding; // how many FS sys calls are executing.
  int committing;  // in commit(), please wait.
  int dev;
  struct logheader lh;
};
```



xv6fs采用固定大小的log区，这就导致文件系统要检查每个写操作的大小会不会超过log区大小，并且需要将大文件的写入切分为小块，分批次commit（问题：如果批次commit过程中宕机了怎么办）。

有了log系统，磁盘数据的写操作应遵循一下的步骤：

```c
begin_op();
...
bp = bread(...);
bp->data[...] = ...;
log_write(bp);
brelse(bp);
end_op();
```

begin_op()会等待正在commit的操作完成，以及log区剩余空间满足所有正准备写入的数据量。之后将log的outstanding加一，表示有系统调用正在写日志。

log_write()可以看作是对bwrite()的封装。但是只是将block加入logheader，修改block标志位为 `B_DIRTY` 从而保证不会别的进程占用，并不写数据。在此过程中，log如果发现当前修改的block之前有重复，则会合并两次修改，减少磁盘请求。

end_op()，将log的outstanding操作数减1，如果减为0则进行commit操作，否则唤醒其它等待begin_op()的进程，因为此次end_op()减少了outstanding数，也就是可能减少了占用的log空间。

提交阶段分四步，1）write_log()将log数据结构中记录的buf数据写入log区；2）write_head()将log数据结构写入log区中，这一步是真正的commit操作；3）将log区的数据拷贝到真正的数据区；4）将log内存结构清空，并将磁盘log区清空。



## Inode

inode可以看作是文件的元信息，对于文件的操作都应通过inode来进行，通常情况下，磁盘上的inode：“dinode”以及内存中的inode都统称为inode，内存中的inode除了是磁盘中inode的拷贝外还会含有一些控制信息等。所有的inode都集中存放在磁盘的几个相邻block中，因此可以根据i-number来随机查找访问inode。inode相关的文件有fs.c，fs.h，file.h。

关键数据结构有：

```c
// in-memory copy of an inode
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count
  int flags;          // I_BUSY, I_VALID

  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+1];
};
#define I_BUSY 0x1
#define I_VALID 0x2

// On-disk inode structure
struct dinode {
  short type;           // File type 0(free), T_DIR, T_FILE, T_DEV
  short major;          // Major device number (T_DEV only)
  short minor;          // Minor device number (T_DEV only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+1];   // Data block addresses
};
```



和bcache类似，xv6持有一份内存inode缓存icache，与inode相关的操作主要有ialloc()，iget()，iput()，ilock()，iunlock()，与inode content相关的主要有writei()，readi()，以及与block相关的bmap()，balloc()，bfree()。

对一个inode的典型操作和log类似：

```c
ip = iget(dev, inum)
ilock(ip)
... examine and modify ip->xxx ...
iunlock(ip)
iput(ip)
```



ialloc()用于在磁盘上找到一空闲inode然后初始化并返回，同时通过调用iget()在icache中缓存一份元信息，通过ref记录cache的引用数。这里是不是可以优化一下，先从icache中找，就不用读磁盘了。

iget()会返回i-number的inode内存信息，同时增加ref引用数，相应的iput()会减少引用数，如果引用和其自身link都为0时，说明该inode对应的文件已被删除，应该回收该inode及其所有blocks。

ilock()，iunlock()通过修改inode的flags标志位保证一个inode只能被一个进程使用。

先说block相关的，bmap()提供了一个有效inode中offset与系统blockno对应关系的计算方法。可以得到偏移量所在的blockno，或者必要时分配新的blockno；balloc()通过修改blockmap block，分配一块空block并返回blockno；bfree()通过修改blockmap删除对应的block。

writei()就是对相应的inode，offset处写入n字节内容，其中涉及到通过bmap()中调用balloc分配空block，并将其关联到该inode中。相应的readi()就是将inode中对应的blockno的数据读到目的地址中。



## Directory

目录层主要提供了inode内容到目录的抽象。目录被看作是一种特殊的文件，文件内容是该目录含有的文件名及其inum，用数据结构 `struct dirent` 来表示。

dirlookup()实现了按照名字在一 `T_DIR` 类型inode中查找，并返回对应的inode。

dirlink()实现了将(name, inum)插入到相应inode中。



## Pathname

这一层实现了inode的命名，可以根据pathname得到相应文件的inode。

随着抽象层越来越高，锁的粒度问题应该的得到重视，在这一层的namx函数中，对于不同目录层级文件名的查找是分开加锁的而不是整个循环一把大锁，这样可以使得不同进程的文件查找得以并行执行。这也得益于iget()和ilock()是分开的而不是像bread()访问和锁是合起来的，同时inode的访问和锁分开防止了一个死锁问题，例如如果是访问当前目录“.”，可以先放锁，在下一次循环再加锁（详见fs.c:610 namx）。



## File Descriptor

UNIX一个重要思想是所有资源都可以看作是文件，File Descriptor是实现这一思想的重要途径。xv6用结构体 `struct file` 来表示文件，在每个进程中记录有自己打开的文件指针。

```c
struct file {
  enum { FD_NONE, FD_PIPE, FD_INODE } type;
  int ref; // reference count
  char readable;
  char writable;
  struct pipe *pipe;
  struct inode *ip;
  uint off;
};
```



## System Call

最上一层就是系统调用，read()，write()之类，以及相关的实现。

图3 所示的open()系统调用图大致上展示了所有的xv6fs函数调用。

![](https://ws3.sinaimg.cn/large/006tKfTcly1fh4q5fpqy2j30mj0rb0tw.jpg)图 3



## 总结

至此我们从底到上看到了一个简单文件系统的全貌，有分层实现逻辑的概念，cache的使用，锁的使用，资源的引用计数，日志系统的使用等等。虽然实现很简单，但遵循了基本的UNIX文件系统理念，提供了对磁盘的基本管理，尤其是逻辑分层部分确实能学到不少东西。同时也感受到了一些进程切换，内核的锁，和用户空间内核空间边界等部分的概念。

### 与Linux 文件系统比较

Linux也是一种类UNIX操作系统，在文件系统概念上也是比较相近，Linux的VFS系统也是将文件系统分为多层模型，提供了superblock object，inode object，file object，dentry object集中抽象接口，具体的文件系统将要实现这些接口中的相关操作函数，这种设计也为支持多种文件系统提供了方便。同时Linux加入了page cache概念，也就是将block cache扩大到了page cache，从而提供了更好的性能。

### 后续

接下来是时候对Linux文件系统下手了。。。



## 参考文献

1. [https://pdos.csail.mit.edu/6.828/2016/xv6/book-rev9.pdf](https://pdos.csail.mit.edu/6.828/2016/xv6/book-rev9.pdf)
2. [https://github.com/mit-pdos/xv6-public/tree/xv6-rev9](https://github.com/mit-pdos/xv6-public/tree/xv6-rev9)
3. Daniel P. Bovet / Marco Cesati. Understanding the Linux Kernel, Third Edition
