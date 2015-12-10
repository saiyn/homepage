---
layout: post
title:  "常用Shell技能"
date:   2015-10-20 15:15:54
categories: Linux
excerpt: 只记录平时开发中会长用的，太全太详细的shell命令没有必要。
---

* content
{:toc}

本文只记录一下平时开发经常会用到的Shell命令，一些细节记录下来方便以后查找。

---

##重定向

**期望每个程序的输出都是其他程序的输入，即使是未知的程序。**这是Unix哲学里让我感受最深的一条。在[从LinkedIn,Apache Kafka到Unix哲学](http://www.jointforce.com/jfperiodical/article/1036?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)一文中，对此有更加鲜明的说明。
这里我们将主要关注两个符合: **>**和**|**。

---

###重定向符: **>**

ls命令默认将结果输出到屏幕，使用“>”重定向符可以将所得结果输出到指定文件中。
<pre><code># ls -l /usr/bin > out.txt
</code></pre>

重复运行上面的命令，发现out.txt中的文件一直不变，这是因为当我们使用">"来重定向输出结果时，目标文件总是从开头被重写。如果想把重定向结果追加到文件内容后面，就使用">>"重定向符。
<pre><code># ls -l /usr/bin >> out.txt
</code></pre>

###管道符: **|**

---

##tar
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



---
