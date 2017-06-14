---
layout: post
title:  "Linux 基础笔记"
date:   2016-01-20 15:15:54
categories: Linux-Application
excerpt: linux
---

* content
{:toc}

记录开发中实际涉及的Linux应用编程知识

---

## unix中的时间值

unix系统中使用两种不同的时间值，日历时间和进程时间。

> * 日历时间

日历时间很好理解，其实就是和程序没有任何关系的自然时间。该值是自1970年01.01.00:00:00以来的国际标准时间(UTC)。见到最多的地方应该是我们执行`ls -l`时，文件和文件夹后面跟着的时间值。这是因为unix系统就是使用这些时间值来记录文件最近一此被修改的时间等。

系统基本数据类型`time_t`用于保存这种时间值。

> * 进程时间

该时间主要是用来度量进程使用cpu资源情况的，所以又被称之为cpu时间。系统使用基本数据架构`clock_t`来保存这种时间值。




---

## 存储映射I/O(mmap)

### 基本概念

存储映射I/O使一个磁盘文件与存储空间的一个缓冲区相映射，这样就不用执行read和write
来读文件或者写缓存区。

为了使用这种功能，应首先告诉内核将一个给定的文件映射到一个存储区域中。这是由mmap函数实现的。
<pre><code>#include "sys/mman.h"
void *mmap(void *addr, size_t len, int prot, int flag, int filedes, off_t off);
</code></pre>

addr 参数用于指定映射存储区的起始地址，通常设置为0,这样表示由系统选择该映射区的起始地址。此函数的返回地址是该映射区的起始地址。

**prot　参数说明对映射区的保护要求:**

prot	|说明
---		|---
PROT_READ	|映射区可读
PROT_WRITE	|映射区可写
PROT_EXEC	|映射区可执行
PROT_NONE	|映射区不可访问

对指定映射存储区的保护要求不能超过文件open模式访问权限。

**flag　参数影响映射存储区的多种属性:**

flag	|说明
---		|---
MAP_FIXED	|返回值必须等于addr.如果未指定此标志，而且addr非0,则内核只把addr视为在何处设置映射区的一种建议。
MAP_SHARED	|此标志指定存储操作修改映射文M件，也就是说，存储操作相当于对该文件的write。
MAP_PRIVATE	|此标志说明，对映射区的存储操作导致创建该映射文件的一个私有副本。
MAP_XXX		|各个不同系统定义自己支持的flag,具体参考mmap(2)

off 和 addr的值通常应该是系统虚存页长度的倍数。虚存页的长可用带参数`_SC_PAGESIZE`或者`_SC_PAGE_SIZE`的sysconf函数得到。

与映射存储区相关的信号有SIGSEGV和SIGBUS。前者通常用于指示进程试图访问对它不可用的存储区。另外，如果进程企图存储数据到mmap指定为只读的映射存储区，
那么也产生此信号。

如果访问映射区的某个部分，而在访问时这一部分实际上已不存在，则产生SIGBUS信号。例如，某个进程截短了另一个进程映射的文件，此时如果这个进程访问已截去的映射区，
则会收到SIGBUS信号。

进程终止时，或者调用了munmap之后，存储映射区就被自动解除映射。但是关闭文件描述符并不解除映射区。

下面是mmap与使用read/write(缓冲区长度为8196)进行文件复制性能的比较，被复制文件的长度是300MB:

**read/write 与　mmap/memcpy比较的时间结果**

操	|Linux2.4.22(intel x86)	
	|---	|---
作	|用户	|系统	|时钟
read/wirte	|0.04	|1.02	|39.76
mmap/memcpy	|0.64	|1.31	|24.26


如果考虑到时钟时间，那么mmap和memcpy方式比read/write方式要快。

### /dev/zero的存储映射



### 匿名存储映射

在调用mmap时指定`MAP_ANON`标志，并将文件描述符指定为-1,就可以得到一个匿名的区域(因为它并不通过一个文件描述符与一个路径相结合)，并且创建一个可与后代进程共享的存储区。具体的调用方法如下:

<pre><code>area = mmap(0,SIZE,PROT_READ | PROT_WRITE, MAP_ANON | MAP_SHARED, -1, 0);
</code></pre>





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

### 启动相关细节

1.由于模块放置在磁盘的根目录内，因此在启动的过程中内核必须要挂载根目录系统，而且为了担心影响到磁盘内的文件系统，因此启动过程中根目录是以只读方式来挂载的。

2.虚拟文件系统(InitialRAM Disk)一般使用的文件名为/boot/initrd，这个文件的特色是，它也能够通过boot loader来加载到内存中，然后这个文件会解压到内存中仿真成一个根目录，且该仿真文件系统能够提供一个可执行程序来加载启动过程中最需要的内核模块。

3.需要initrd最重要的原因是，当启动时无法挂载根目录的情况下，此时就一定要initrd。如果你的linux是安装在IDE接口的磁盘上，并且使用默认的ext2/3文件系统时，那么没有initrd也能顺利启动linux。

4.在内核加载完成后，你的主机应该开始正确运行了，接下来，就是要开始执行系统的第一个程序:/sbin/init。

5.pid为1的/sbin/init程序通过/etc/inittab配置文件来规划，/etc/inittab文件内有一个很重要的设置选项，那就是默认的run level(启动执行等级)。

6.inittab中的语法是:[设置选项]:[run levle]:[init的操作行为]:[命令选项].

**init的操作行为说明如下表:**

inittab设置值	|意义说明
------	|------
initdefault	|代表默认的runlevel设置值
sysinit	|代表系统初始化的操作选项
wait	|代表后面字段设置的命令项目必须要执行完毕才能继续下面其他的操作
respawn	|代表后面的字段设置的命令可以重新启动

**启动系统服务与相关启动配置文件**

1.服务的启动是/etc/rc.d/rc文件以/etc/init.d/xxx{start,stop}来启动与关闭的。

2./etc/rc.d/ 中的文件都是K和S开头的文件，后面接着的数字是表示该文件被执行的顺序，数字越大表示越后执行。

3.最后一个被执行的选项是S99local,即是/etc/rc.d/rc.local这个文件。

**用户自定义启动程序(/etc/rc.d/rc.local)**

我们有任何想要在启动时就可以进行的工作，直接将它写入/etc/rc.d/rc.local就会在启动时候被自动加载。而
不必等我们登陆系统去启动它。

**启动过程中会用到的主要配置文件**

1.启动过程中用到的配置文件大多数放置在/etc/sysconfig目录下。

2.某些条件下我们得对模块进行一些参数的规划，此时要使用/etc/modprobe.conf。



---

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











