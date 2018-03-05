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

---

### cached中的内存是否都可以回收

<br />










