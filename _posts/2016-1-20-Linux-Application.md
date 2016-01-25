---
layout: post
title:  "Linux 应用编程笔记"
date:   2016-1-20 15:15:54
categories: Linux-Application
excerpt: linux
---

* content
{:toc}

记录开发中实际涉及的Linux应用编程知识

---

##exec 函数
当进程调用一种exec函数时，该进程执行的程序完全替换为新程序，而新程序则从其main函数开始执行。
因为调用exec并不创建新进程，所以前后的进程ID并未改变。exec只是用一个全新的程序替换了当前进程的正文、数据、堆和栈段。
`exec`加上`v`, `l`, `e`, `p`这4个字母的两两组合或者单独一个总共有6种不同的函数可供使用。

字母	|词意	|说明
v		|vector	|`v`表示传递参数时应先构造一个指向各参数的指针数组，然后将该数组地址作为参数传递。
l		|list	|`l`与上面的`v`是向对应的，表示要求将新程序的每个命令行参数都说明为一个单独的测试，并且这种参数表以空指针结尾。
e		|environment	|`e`表示在向新程序传递环境表时，可以传递一个指向环境字符串指针数组的指针。
p		|path	|`p`表示可以通过传递文件名来定位新程序文件的位置，但是是如果传递的文件名中如果包含`/`，则会忽略'p'的作用，将其视为路径名。否则就按PATH环境变量去搜索。

> *值得注意的是：如果`execlp`或者`execvp`使用PATH搜索到一个可执行文件，但是该文件不是由连接器产生的机器可执行文件，则认为该文件时应该`shell`脚本，于是试着调用`/bin/sh`，并以该`filename`作为`shell`的输入。

<pre><code>#include <unistd.h>

int execl(const char *pathname, const char *arg0, .../* (char *)0 */);
int execv(const char *pathname, char *const argv[]);
int execle(const char *pathname, const char *arg0, .../*(char *)0*/, char *const envp[]);
int execve(const char *pathname, char *const argv[], char *const envp[]);
int execlp(const char *filename, const char *arg0, .../* (char *)0 */);
int execvp(const char *filename, char *const argv[]);
</code></pre>







