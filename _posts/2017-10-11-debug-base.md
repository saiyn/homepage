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
# DWARF

DWARF的全称是"Debug With Attributed Record Formats",现在已经有dwarf1,dwarf2,dwarf三个版本，关于DWARF格式的详细说明
参见[这篇资料](http://dwarfstd.org/doc/dwarf-2.0.0.pdf)

dwarf3.0于2006年推出，现在已经是一种独立的标准，支持c,c++,java,fortran等语言。

使用readelf工具可以分析查看dwarf存放在elf文件中的各个段，只要编译时加上-g参数，elf文件中都包含.debug_info,.debug_line,.debug_str等段。
这些段的内容就是dwarf协议的具体表现。执行`readelf -S cycbuf`，结果如下:

![dwarf_1]()

要查看上面几个相关的debug段的具体内容，使用readelf -w*命令，*是需要查看段名的第一个字母，比如-wi就是查看.debug_info段的内容，结果如下:

![dwarf_2]()

执行readelf -wl cycbuf的结果如下：

![dwarf_3]()
