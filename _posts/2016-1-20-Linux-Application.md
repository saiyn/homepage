---
layout: post
title:  "Linux 应用编程笔记"
date:   2016-1-20 15:15:54
categories: Linux Application
excerpt: linux-application 
---

* content
{:toc}

记录开发中实际涉及的Linux应用编程知识

---

##exec 函数
当进程调用一种exec函数时，该进程执行的程序完全替换为新程序，而新程序则从其main函数开始执行。
因为调用exec并不创建新进程，所以前后的进程ID并未改变。exec只是用一个全新的程序替换了当前进程的正文、数据、堆和栈段。
`exec`加上`v`, `l`, `e`, `p`这4个字母的两两组合或者单独一个总共有6种不同的函数可供使用。

<pre><code>#include <unistd.h>
int execl(const char *pathname, const char *arg0, ...)
int execv(const char *pathname, )
</code><pre>

---






