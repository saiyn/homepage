---
layout: post
title:  "Linux内存笔记"
date:   2017-08-11 15:15:54
categories: Linux-Kernel
excerpt:  linux memory
---

* content
{:toc}

内存无小事，性能优化的起点可能就是内存优化。Linux下的内存管理还是比较复杂的，如何甄别一个系统的内存状态是否健康、如何在系统发生内存问题时快速定位
到问题点，这些都非易事，需要积累。

---

<br />

# 实战技术

<br />

## 列出系统中top10占用内存(RSS)最多的进程

<br />

    ps -e -orss,pid=,user=,args=, | sort -b -k1,1n | pr -TW$COLUMNS| tail -10


第一列为RSS大小，单位KB, 第二列为PID

<br />

# 了解内存以及内存的状态

<br />

在现代的操作系统中，每个进程都活在它自己的内存分配空间(memory allocation space)中。操作系统作为硬件抽象层为每一个进程提供一个独立的相同的虚拟地址空间(virtual memory space)。为了实现这一点，操作系统使用per-process translation table来为每一个进程进行物理地址和虚拟地址的映射。

使用虚拟地址的好处如下:



<br />

## Memory Types

<br />

从两个不同维度可以将虚拟内存分为4种类型，其中一个维度是，这个内存是进程私有的(specific to that process)还是多个进程可以共享的(shared)；另一个维度是，这个内存是匿名的(anonymous)还是绑定文件的(file-backed)。4中不同类型的内存如下图所示:

![lmem_0](http://omp8s6jms.bkt.clouddn.com/image/git/lmem_0.png)

<br />

**Private Memory**

所谓的private memory就是这块内存是一个进程内部私有的，其他进程无法访问。In case of file-backed private memory, the changs made by the process
are not written back to the underlying file, however changs made to the file may or may not be made available to the process.

<br />

**Shared Memory**

<br />

与private对应的自然是shared, it can only be created by explicitly requesting it using the right mmap() call or a dedicated call (shm*)。

如果这块shared内存是file-backed, 那么any process mapping the file will see the changes in the file.

<br />

**Anonymous Memory**

Anonymous memory is purely in RAM. 但是在这块虚拟内存没有被实现写之前，内核是不会将其映射到实际的物理内存的。因为虚拟内存的这个特性，我们可以实现很多有意思功能，具体见这篇文章[Virtual Memory Tricks](http://ourmachinery.com/post/virtual-memory-tricks/)

<br />

**File-backed and Swap**

When a memory map is file-backed, the data is loaded from the disk. 大部分情况下，这些数据是按需加载的，但是你也可以给内核一些hints,让内核可以
在访问前进行预取(prefetch)。

当系统出现内存紧张时，内核会尝试将RAM中的一些数据移到硬盘上去。如果这些内存是file-backed and shared,那么这种搬移很简单。Since the file is the source of the data, it is just removed from RAM,then 下次需要的时候再从文件中加载，这个过程称为page-out,不需要用到交换区(swap)。但是当内核尝试将RAM中的anonymous或者private内存搬移到硬盘上时，内核得将这些数据写到硬盘上的特殊位置。这个过程叫做swap-out。




<br />

---

## /proc/buddyinfo

<br />

buddyinfo记录的信息是Linux系统下Buddy memory allocation机制下内存分配的状态，网上大部分文章只是说明了其中字段的含义，也顺便说了句该文件的信息
主要用来分析linux系统内存碎片的情况，但是很少有文章说明如何从这个文件的信息来判断当前系统内存碎片的情况。

cat /proc/buddyinfo的输出结果大致如下:

![lmem_1](https://raw.githubusercontent.com/saiyn/homepage/gh-pages/images/lmem_1.png)

<br />

要理解上面输出的含义，得先参阅一下wiki上的这篇文章[Buddy memory allocation](https://en.wikipedia.org/wiki/Buddy_memory_allocation)

判断当前系统内存碎片严重程度有个计算公式:

![lmem_2](https://raw.githubusercontent.com/saiyn/homepage/gh-pages/images/lmem_2.png)

下面通过一个实例计算来说明这个公式的使用方法:

![lmem_3](https://raw.githubusercontent.com/saiyn/homepage/gh-pages/images/lmem_3.png)

<br />

---

## /proc/meminfo

<br />

/proc/meminfo文件是了解Linux系统内存使用状况的主要途径，常用的`free`、`vmstat`等命令都是通过它获取数据的。



<br />

### 内核层统计

<br />

kernel的动态内存分配主要通过以下几种接口：

* alloc_pages/__get_free_page: 以页为单位分配。
* vmalloc: 以字节为单位分配虚拟地址连续的内存块。
* slab allocator: 以字节为单位分配物理地址连续的内存块。

内核所用内存的静态部分，比如内核代码、页面描述符等数据在引导阶段就分配掉了，并不计入MemTotal里，而是算作Reserved。而内核所用内存的动态部分，
是通过上文提到的几个接口申请的，其中通过alloc_pages申请的内存有可能未纳入统计，就像**黑洞**一样。下面讨论的都是/proc/mmeinfo中统计到的部分。

<br />

**SLAB**

通过slab分配的内存被统计在以下三个值中:

* SReclaimable: slab中可回收的部分。在调用kmem_getpages()时加上SLAB_RECLAIM_ACCOUNT标记，表明是可回收的，计入SReclaimable，否则计入SUnreclaim。

* SUnreclaim: slab中不可回收的部分。

* Slab: slab中所有的内存，等于以上两者之和。

<br />

---

### 用户层统计

<br />

**shmem**

/proc/meminfo中的shmem统计的内容包含:

* tmpfs
* IPC API call, 例如shmget、shmat
* mmap with the SHARED flags
* DRM GEM objects

所以 shmem = tmpfs + "share memory based IPC" + SHARED mmap + GEM objects

tmpfs就是临时文件系统，它将内存的一部分空间拿来当做文件系统使用，使内存空间可以当做目录文件来用。

<br />

**buffers/cached**

buffers是对原始磁盘块的临时存储，也就是用来缓存**磁盘的数据**，通常不会太大(200M左右)。这样，内核就可以把分散的写集中起来，统一优化磁盘的写入。

cached是从磁盘读取文件的页缓存，也就是用来**缓存从文件读取的数据**。这样，下次访问这些文件数据时，就可以直接从内存中快速获取，而不需要再次访问缓慢的磁盘。

cached = page cache + shmem

根据上面shmem章节，可以进一步展开得到：

cached = page cache + tmpfs + "share memory based IPC" + SHARED mmap + GEM objects

<br />

**LRU**

<br />

LRU(Least Recently Used)是kernel的页面回收算法(Page Frame Reclaiming)使用的数据结构。Page cache和所有用户进程的内存(kernel stack、huge pages除外)都在LRU lists上。

LRU lists包含如下几种，在/proc/meminfo中都要对应的统计值:

LRU_INACTIVE_ANON - 对应Inactive(anon)

LRU_ACTIVE_ANON   - 对应Active(anon)

LRU_INACTIVE_FILE - 对应Inactive(file)

LRU_ACTIVE_FILE   - 对应Active(file)

LRU_UNEVICTABLE   - 对应Unevictable

另外，

* Inactive list里的是长时间未被访问过的内存页，Active list里的是最近被访问过的内存页，LRU算法利用Inactive list和Active list可以判断哪些内存页可以被优先回收。
* 括号中的anon表示匿名页(anonymous pages)。用户进程的内存页分为两种：file-backed pages(与文件对应的内存页)、anonymous pages(匿名页)。进程的代码、映射的文件都是file-backed,而进程的堆、栈都是不与文件相对应的，就属于匿名页。
* Unevictable LRU list上的是不能page-out/swap-out的内存页，包括VM_LOCKED的内存页、SHM_LOCK共享内存页(被统计在`Mlocked`项中)和ramfs。在unevictable list出现之前，这些内存页都在active/inactive lists上，vmscan每次都要扫过它们，但是又不能把它们page-out/swap-out，这在打内存的系统上严重影响性能，设计unevictable list的初衷就是避免这种情况。


<br />

**Mapped**

cached中的page cache包含了文件的缓存页，其中有些文件当前已经不在使用，page cache仍然可能保留着它们的缓存页面；而另外一些正在被用户进程关联的文件，
比如shared libraries、可执行文件、mmap的文件等，这些文件的缓存页就被称为`mapped`。这里所说的内存即为上面章节Memory Types图中的3,4。

进程所占的内存页分为annoymous pages和file-backed pages,理论上应该有:

【所有进程的PSS之和】 == 【Mapped + AnonPages】

但是执行命令`$ grep Pss /proc/[1-9]*/smaps | awk ‘{total+=$2}; END {print total}’`得到的值比Mapped + AnonyPages的值要小，这是因为mapped还包含Memory Types图中的1.2和2的内存部分。即Mapped = file-backed pages + mmap(ANON)

<br />

**AnonPages**

用户进程的内存页分为两种：filed-backed pages 和 anonymous pages。 anonymous pages就记录在/proc/meminfo中的anonpages项中， 它具有如下特性:

* cached中的page cache里面都是file-backed pages, 没有anonymous pages。
* mmap(ANON, PRIVATE)属于AnonPages, 而mmap(ANON, SHARED)属于cached(file-backed pages)，因为shared anonymous mmap也是基于tmpfs实现的。
* Anonymous Pages是与用户进程共存的，一旦进程退出，anonymous pages就被释放掉，不像page cache即使文件与进程不关联了还可以缓存。
* AnonPages统计值中包含了Transparent HugePages(THP)对应的AnonHugePages。

<br />

****


---

### cached中的内存是否都可以回收

<br />










