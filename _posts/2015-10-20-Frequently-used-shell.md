---
layout: post
title:  "常用Shell技能"
date:   2015-10-20 15:15:54
categories: Linux
excerpt: linux shell
---

* content
{:toc}

本文只记录一下平时开发经常会用到的Shell命令，一些细节记录下来方便以后查找。

---

## objdump

代码实例是[C学习笔记](http://saiyn.github.io/homepage/2016/08/07/C/)一文中`地址无关代码`小节中的代码。

参数`-h`就是把ELF文件的各个段的基本信息打印出来。

![obi_h](http://omp8s6jms.bkt.clouddn.com/image/git/obj_h.png)

参数`-d`是将所有包含指令的段进行反汇编。

![obj_d](http://omp8s6jms.bkt.clouddn.com/image/git/obj_d.png)





## 重定向

**期望每个程序的输出都是其他程序的输入，即使是未知的程序。**这是Unix哲学里让我感受最深的一条。在[从LinkedIn,Apache Kafka到Unix哲学](http://www.jointforce.com/jfperiodical/article/1036?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)一文中，对此有更加鲜明的说明。
这里我们将主要关注两个符合: **>**和**|**。

---

### 重定向符: **>**

ls命令默认将结果输出到屏幕，使用“>”重定向符可以将所得结果输出到指定文件中。
<pre><code># ls -l /usr/bin > out.txt
</code></pre>

重复运行上面的命令，发现out.txt中的文件一直不变，这是因为当我们使用">"来重定向输出结果时，目标文件总是从开头被重写。如果想把重定向结果追加到文件内容后面，就使用">>"重定向符。
<pre><code># ls -l /usr/bin >> out.txt
</code></pre>

### 管道符: **|**

---

## tar

> * `-c`:压缩文件
> * `-x`:解压文件
> * `-z`:用gzip压缩文件(压缩还是解压依赖是否组合了`-c`或者`-x`)
> * `-j`:用bzip2压缩文件，一般用于`.bz2`结尾的文件。
> * `-v`(verbose):显示解压过程的具体信息
> * `-f`(file):指定要解压的文件,**在f之后要立即接文档名，不能再加其他参数**
<pre><code>#tar xjvf busybox-1.24.1.tar.bz2 -C /home/work
</code></pre>
上面就是一个将以`.bz2`格式的文件解压到指定路径的例子，如果不加`-C`就解压到当前目录中。
<pre><code>#tar xzvf boa-0.94.13.tar.gz
</code></pre>
上面就是将以`.gz`格式的文件解压到当前目录的例子。
<pre><code>#ls
u-boot-2015.10 boa-0.94.13
#tar zcvf u-boot.tar.gz u-boot-2015.10
#ls
#u-boot-2015.10 boa-0.94.13 u-boot.tar.gz
</code></pre>
上面是一个压缩文件的例子，在参数f之后的文档名是自己取的，我们习惯上都用.tar来作为辨识。

---

## grep

grep命令用来搜索文本，或从给定的文件中搜索行内包含了给定字符串或单词的文件。
一般来说，grep显示匹配到的行。

### grep命令的语法

<pre><code>grep [-option] '搜索字符串' filename
cat 文件 | grep '搜索字符串'
</code></pre>

### 在一个文件进行搜索

搜索/etc/passwd文件下的boo用户：
<pre><code>$grep boo /etc/passwd
</code></pre>
可以使用grep去强制忽略大小写。例如，使用-i选项可以匹配boo,Boo,BOO和其他组合：
<pre><code>$grep -i "boo" /etc/passwd
</code></pre>
当你搜索boo时，grep命令将会匹配fooboo,boo123,barfoo35和其他所有包含boo的字符串，可以使用`-w`选项去强制完全匹配搜索
<pre><code>$grep -w "boo" file
</code></pre>
可以通过添加`-c`选项显示匹配到的次数，`-n`选项可以输出匹配到的行号，`-v`选项可以进行反转匹配。


### 在一个目录中递归搜索

加上`-r`或者`-R`选项可以在一个目录中递归搜索所有文件。例如，在文件目录下面搜索所有包含字符串“192.168.1.5”的文件
<pre><code>$grep -r "192.168.1.5" /etc/
</code></pre>

### 管道与grep命令

grep常常与管道一起使用，在这个例子中，显示硬盘设备的名字：
<pre><code>#dmesg | egrep '(s|h)d[a-z]'
</code></pre>
显示CPU型号：
<pre><code>#cat /proc/cpuinfo | grep -i 'Model'
</code></pre>

### 仅仅显示匹配到内容的文件名字

使用`-l`选项去显示那些文件内容中包含main()的文件名：
<pre><code>$grep -l 'main' *.c
</code></pre>

---

## 解压.cpio.gz文件

<pre><code>$gzip -dc file.gz | cpio -div
</code></pre>

---

## 解压ramdisk.gz文件

<pre><code>$gunzip ramdisk.gz
</code></pre>
解压后得到ramdisk镜像文件，该镜像文件会把原有的ramdisk.gz覆盖。
<pre><code>$mkdir mnt
$mount -o loop ramdisk mnt 
</code></pre>
挂载镜像到mnt目录，这样mnt目录里面就是展开后的文件系统目录。

---

## PS命令

Linux上进程的5种状态：

状态		|描述								|PS中的状态码
----		|----								|----
运行		|正在运行或在运行队列中等待			|R(runnable)
中断		|休眠中，受阻，在等待某个条件		|S(sleeping)
不可中断	|信号不可唤醒，必须等到中断发送		|D(uninterruptble sleep)
僵死		|进程已终止，等待父进程回收			|Z(zombie)
停止		|进程收到SIGSTOP,SIGSTP,SIGTIN		|T(traced or stopped)

显示所有进程信息，连同命令行
<pre><code>#ps -ef
</code></pre>

列出目前所有的正在内存当中的程序
<pre><code>#ps aux
</code></pre>

将目前属于自己这次登入的PID与相关信息列出来
<pre><code>#ps -l
</code></pre>














