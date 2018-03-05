---
layout: post
title:  "Linux内存笔记"
date:   2017-08-11 15:15:54
categories: Linux
excerpt:  linux memory
---

* content
{:toc}

内存无小事，性能优化的起点可能就是内存优化。Linux下的内存管理还是比较复杂的，如何甄别一个系统的内存状态是否健康、如何在系统发生内存问题时快速定位
到问题点，这些都非易事，需要积累。

---

<br />

# 了解内存以及内存的状态

<br />

在现代的操作系统中，每个进程都活在它自己的内存分配空间(memory allocation space)中。操作系统作为硬件抽象层为每一个进程提供一个独立的相同的虚拟地址空间(virtual memory space)。为了实现这一点，操作系统使用per-process translation table来为每一个进程进行物理地址和虚拟地址的映射。

使用虚拟地址的好处如下:



<br />

## Memory Types

<br />

从两个不同维度可以将虚拟内存分为4种类型，其中一个维度是，这个内存是进程私有的(specific to that process)还是多个进程可以共享的(shared)；另一个维度是，这个内存是匿名的(anonymous)还是绑定文件的(file-backed)。4中不同类型的内存如下图所示:



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

当系统出现内存紧张时，内核会尝试将RAM中的一些数据移到硬盘上去。如果这些内存是file-backed and shared,那么这种搬移很简单。Since the file is the source of the data, it is just removed from RAM,then 下次需要的时候再从文件中加载。但是当内核尝试将RAM中的anonymous或者private内存搬移到硬盘上时，内核得将这些数据写到硬盘上的特殊位置。这个过程叫做swap。




<br />

---

## /proc/meminfo

<br />

/proc/meminfo文件是了解Linux系统内存使用状况的主要途径，常用的`free`、`vmstat`等命令都是通过它获取数据的。下面分析其中各项统计的含义。

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

在Linux的内存管理中，这里的buffers指Linux内存的: `Buffer cache`；这里的cache值Linux内存中的: `Page cache`。
在当前的内核中，page cache顾名思义就是针对内存页的缓存，也就是说，如果内存中有以page进行分配管理的，都可以使用page cache作为其缓存管理使用。
除了页，内存中存在使用块(block)进行管理的，这部分内存如果要使用cache功能的话，就使用buffer cache。

cached = page cache + shmem

根据上面shmem章节，可以进一步展开得到：

cached = page cache + tmpfs + "share memory based IPC" + SHARED mmap + GEM objects


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
* 


---

### cached中的内存是否都可以回收

<br />










