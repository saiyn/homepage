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

##fork 函数
子进程获得父进程数据空间、堆和栈的副本，父、子进程并不共享这些存储空间部分。
另外，fork的一个特性是父进程的所有打开描述符都被复制到子进程中，父、子进程的每个相同的打开描述符共享一个文件表现。
通过下面两个例子来演示一下：

<pre><code>#include "apue.h"
int glob = 6;
char buf[] = "a write to stdout\n";
int main(void)
{
  int var;
  pid_t pid;
  
  var = 88;
  if(write(STDOUT_FILENO, buf, sizeof(buf)-1) != sizeof(buf) - 1)
	err_sys("write error");
  prnitf("before fork\n");
  
  if((pid = fork()) < 0){
	err_sys("fork error");
  }else if(pid == 0){
	glob++;
	var++;
  }else{
	sleep(2); /*parent*/
  }
  
  printf("pid = %d, glob = %d, var = %d\n", getpid(). glob, var);
  exit(0);
}
</code></pre>

执行此程序则得到：

> $ ./a.out
> a write to stdout
> before fork
> pid = 430, glob = 7, var = 89 子进程的变量值改变了
> pid = 429, glob = 6, var = 88 父进程的变量值没有改变
> $ ./a.out > temp.out
> $cat temp.out
> a write to stdout
> before fork
> pid = 432, glob = 7, var = 89
> before fork 
> pid = 431, glob = 6, var = 88

除了注意上面执行结果的注释部分说明了子进程复制父进程的存储空间并且子进程对变量所作的
改变并不影响父进程中的该变量的值以外，还要注意`fork`与I/O函数之间的交互关系。
write函数是不带缓冲的，但是，标准I/O库是带缓冲的。如果标准输出连接到终端设备，则它是行缓冲的，
否则它是全缓冲的。当以交互方式运行该程序时，只得到该printf输出的行一次，其原因是标准输出
缓冲区由换行符冲洗。但是当标准输出重定向到一个文件时，却得到printf输出两次。
其原因是，在fork之前调用了printf一次，但当调用fork时，该行数据仍在缓冲区中，然后在将父进程数据空间
复制到子进程时，该缓冲区也被复制到子进程中。当每个进程终止时，最终会冲洗其缓冲区中的副本。

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








