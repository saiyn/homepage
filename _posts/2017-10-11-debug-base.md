---
layout: post
title:  "调试技术之基础知识笔记"
date:   2017-10-11 15:15:54
categories: debug
excerpt: linux debug debuginfo
---

* content
{:toc}

在部署和使用调试工具时，往往因为调试方面的一些基础知识的缺乏，导致遇到很多克服不了的可能，最终影响问题的解决。本文记录使用各种调试工具中遇到的
问题，以及解决这些问题必须要掌握的基础知识。


<br />

---

# 调试工具

## 内存方面

### free

<br />

free结果中比较难以理解的是buffers,cache这两项。

* buffers是内核缓冲区用到的内存，对应的是/proc/meminfo中的Buffers值。

* cache是内核页缓存和slab用到的内存，对应的是/proc/meminfo中的cached与SReclaimable之和。

至于/proc/meminfo中的各项含义，见[这里](http://saiyn.github.io/homepage/2017/08/11/Linux-Memory/#procmeminfo)

<br />

### /proc

各种调试工具如free,top等最终其实都是依赖于`/proc`系统中的各种信息，自己编写脚本去统计/proc系统文件中的信息也是重要的调试方法。

下面是一个可以实时(每隔一秒)打印感兴趣进程实际占用内存的脚本,这里以nmap进程为例:

    #!/bin/sh
    
    while [ true ]; do
      for pid in $(ps -elf | grep nmap | grep -v grep | awk '{print $1}'); do
        if [ -f /proc/$pid/smaps ]; then
          echo "--Pss:"
          cat /proc/$pid/smaps | grep ^Pss | awk '{total+=$2} END {printf "%d kB", total}'
          
        fi
         
       done
        
        sleep 1
     done

---

<br />

# Load Average

<br />

top命令或者uptime命令都会输出系统平均负载(Load Average)情况，那么这个指标是什么意思，代表着什么，通过这个指标怎么如何判断系统性能情况？

**平均负载的定义**

简单的说，Load Average是指单位时间内，系统处于**可运行状态**和**不可中断状态**的平均进程数，也就是平均活跃进程数。因为其中包含了不可中断的进程，所以说这个指标和CPU使用率没有直接关系。

不可中断状态的进程的典型的是等待硬件设备I/O响应的进程，也就是PS命令中D状态(Uninterruptible Sleep)。

<br />

**平均负载多少时合理**

当平均负载高于CPU数量70%的时候，就应该分析排查负载高的问题了。因为一旦负载过高，就可能导致进程响应变慢。

---

<br />


# DWARF

DWARF的全称是"Debug With Attributed Record Formats",现在已经有dwarf1,dwarf2,dwarf三个版本，关于DWARF格式的详细说明
参见[这篇资料](http://dwarfstd.org/doc/dwarf-2.0.0.pdf)

dwarf3.0于2006年推出，现在已经是一种独立的标准，支持c,c++,java,fortran等语言。

使用readelf工具可以分析查看dwarf存放在elf文件中的各个段，只要编译时加上-g参数，elf文件中都包含.debug_info,.debug_line,.debug_str等段。
这些段的内容就是dwarf协议的具体表现。执行`readelf -S cycbuf`，结果如下:

![dwarf_1](http://omp8s6jms.bkt.clouddn.com/image/git/dwarf_1.png)

要查看上面几个相关的debug段的具体内容，使用readelf -w*命令，*是需要查看段名的第一个字母，比如-wi就是查看.debug_info段的内容，结果如下:

![dwarf_2](http://omp8s6jms.bkt.clouddn.com/image/git/dwarf_2.png)

执行readelf -wl cycbuf的结果如下：

![dwarf_3](http://omp8s6jms.bkt.clouddn.com/image/git/dwarf_3.png)
