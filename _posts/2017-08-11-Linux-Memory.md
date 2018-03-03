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

# 了解内存的分布和动态

<br />

清楚了解当前系统内存状态是其他一切的前提。

<br />

## /proc/meminfo

<br />

/proc/meminfo文件是了解Linux系统内存使用状况的主要途径，常用的`free`、`vmstat`等命令都是通过它获取数据的。
使用free命令时，其输出结果中的`buffers/cache`统计应该是最费解的。

<br />

**什么是buffers/cache**

在Linux的内存管理中，这里的buffers指Linux内存的: `Buffer cache`；这里的cache值Linux内存中的: `Page cache`。
在当前的内核中，page cache顾名思义就是针对内存页的缓存，也就是说，如果内存中有以page进行分配管理的，都可以使用page cache作为其缓存管理使用。
除了页，内存中存在使用块(block)进行管理的，这部分内存如果要使用cache功能的话，就使用buffer cache。

所以,free命令输出中的buffers表示块设备所占用的缓存页，包括: 直接读写块设备、以及文件系统元数据(metadata)；
cached 表示普通文件数据所占用的缓存页。





