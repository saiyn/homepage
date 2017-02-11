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

##	Linux启动

### 启动具体流程

> 1. 加载BIOS执行硬件自检，依据设置取得第一个可启动设备。   	
> 2. 读取并执行第一个启动设备内MBR的boot loader。MBR位于第一个扇区，也就是最前面512字节。主引导记录有三部分组成:
>	* 第1-446字节:调用操作系统的机器码
>	* 第447-510字节:分区表(Partition table)
>		
>		分区表只有64个字节，里面分成四项，形成四个主分区，每项16个字节，由6个部分组成:
>
>			* 第一个字节:如果为0x80,就表示该主分区是激活分区，控制权要转交给这个分区。
>			* 第2-4个字节:主分区第一个扇区的物理地址。
>			* 第5个字节: 主分区类型。
>			* 第6-8个字节:主分区最后一个扇区的物理地址。
>			* 第9-12字节:该主分区第一个扇区的逻辑地址。
>			* 第13-16字节:主分区的扇区总数。
>
>	* 第511-512字节:主引导记录签名(0x55,0xAA)
> 3. 加载内核镜像。
> 4. kernel主动调用init进程，而init进程会取得run-level信息。
> 5. init执行/etc/rc.d/rc.sysinit文件来准备软件执行的操作环境。
> 6. init执行run-level的各个服务启动(script方式)。
> 7. init执行/etc/rc.d/rc.local文件。
> 8. init执行终端模拟程序mingetty来启动login进程。




## fork 函数

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

> $./a.out
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

---

## exec 函数

当进程调用一种exec函数时，该进程执行的程序完全替换为新程序，而新程序则从其main函数开始执行。
因为调用exec并不创建新进程，所以前后的进程ID并未改变。exec只是用一个全新的程序替换了当前进程的正文、数据、堆和栈段。
`exec`加上`v`, `l`, `e`, `p`这4个字母的两两组合或者单独一个总共有6种不同的函数可供使用。

字母	|词意	|说明
v		|vector	|`v`表示传递参数时应先构造一个指向各参数的指针数组，然后将该数组地址作为参数传递。
l		|list	|`l`与上面的`v`是向对应的，表示要求将新程序的每个命令行参数都说明为一个单独的测试，并且这种参数表以空指针结尾。
e		|environment	|`e`表示在向新程序传递环境表时，可以传递一个指向环境字符串指针数组的指针。
p		|path	|`p`表示可以通过传递文件名来定位新程序文件的位置，但是是如果传递的文件名中如果包含`/`，则会忽略'p'的作用，将其视为路径名。否则就按PATH环境变量去搜索。

> *值得注意的是：如果`execlp`或者`execvp`使用PATH搜索到一个可执行文件，但是该文件不是由连接器产生的机器可执行文件，则认为该文件时应该`shell`脚本，于是试着调用`/bin/sh`，并以该`filename`作为`shell`的输入。

<pre><code>#include /<unistd.h/>

int execl(const char *pathname, const char *arg0, .../* (char *)0 */);
int execv(const char *pathname, char *const argv[]);
int execle(const char *pathname, const char *arg0, .../*(char *)0*/, char *const envp[]);
int execve(const char *pathname, char *const argv[], char *const envp[]);
int execlp(const char *filename, const char *arg0, .../* (char *)0 */);
int execvp(const char *filename, char *const argv[]);
</code></pre>

---

## system函数

<pre><code>#include /<stdlib.h/>
int system(const char *cmdstring);
</code></pre>

> * 如果cmdstring是一个空指针，则仅当命令处理程序可用时，system返回非0值，这一特征可以确定在一个给定的操作系统上是否支持system
> 函数。
> * 因为system在其实现中调用了fork、exec和waitpid，因此有三种返回值：
> 1).如果fork失败或者waitpid返回除EINTR之外的出错，则system返回-1，而且errno中设置了错误类型。
> 2).如果exec失败，则其返回值如同shell执行了exit(127)一样。
> 3).如果3个函数都执行成功，返回值是shell的终止状态。

---

## exit函数

在大多数UNIX系统实现中，exit(3)是标准C库中的一个函数，而_exit(2)则是一个系统调用。
进程要么是正常终止返回退出状态(exit status)，要么异常终止返回终止状态(terminatiom status)。
在任意一种情况下，该终止进程的父进程都能够使用wait或者waitpid函数取得其终止状态。

> 值得注意的是，在最后调用_exit时，内核将退出状态转换成终止状态。

有四个互斥的宏可用来取得进程终止的原因，它们的名字以WIF开始：

宏		|说明
----		|----
WIFEXITED(status)		|若为正常终止子进程返回的状态。则为真。对于这种情况可执行WEXITSTATUS(status)，取子进程传给exit、_exit或_Exit参数的低8位。
WIFSIGNALED(status)		|若为异常终止子进程返回的状态，则为真。执行WTERMSIG(status)取使子进程终止的信号编号。
WIFSTOPPED(status)		|若为当前暂停子进程的返回的返回状态，则为真。执行WSTOPSIG(status)取使子进程暂停的信号编号
WIFCONTINUED(status)	|若在作业控制暂停后已经继续的子进程返回了状态，则为真。仅用于waitpid。

对于waitpid函数中pid参数的作用解释如下：

pid值		|说明
----		|----
-1		|等待任一子进程。就这一方面而言，waitpid与wait等效。
>0		|等待其进程ID与pid相等的子进程。
==0		|等待其组ID等于调用进程组ID的任一子进程。
<-1		|等待其组ID等于pid绝对值的任一子进程。

---

## 守护进程

> 编写守护进程时的基本规则：
>
> 1. 首先要做的是调用umask将文件模式创建屏蔽字设置为0.
> 2. 调用fork，然后使父进程退出。这样做实现了2点：1).如果该守护进程是由shell命令启动的
> 那么父进程终止使得shell认为这条命令已经执行完毕。2).子进程继承了父进程的进程组ID，但具有一个新的进程ID，这就保证了子进程不是一个进程组的组长进程。
> 这对于下面就要做的setsid调用是必要的前提条件。
> 3. 调用setsid以创建一个新会话。之后该进程：1).成为新会话的首进程。2).成为一个新进程组的组长进程。c).没有控制终端。
> 4. 将当前工作目录更改为根目录。
> 5. 关闭不再需要的文件描述符。
> 6. 某些守护进程打开/dev/null使其具有描述符0，1和2.

## 标准I/O库

### 打开流

下面3个函数打开一个标准I/O流。
<pre><code>FILE *fopen(const char *restrict pathname, const char *restrict type);
FILE *freopen(conat char *restrict pathname, const char *restrict type, FILE *restrict fp);
FILE *fdopen(int fileds, const char *type);
</code></pre>
这三个函数的区别是：
(1)fopen打开一个指定的文件。
(2)freopen在一个指定的流上打开一个指定的文件，











