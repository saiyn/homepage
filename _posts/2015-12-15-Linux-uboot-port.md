---
layout: post
title:  "Linux u-boot 移植学习笔记"
date:   2015-12-15 15:15:54
categories: Linux
excerpt: linux u-boot port
---

* content
{:toc}

记录移植U-boot，Linux过程中的知识点和技巧。 

---

##启动内核时出现`undefined instruction`错误
启动内核时会出现各种错误，我遇到的是如下情况：
<pre><code>......
Image Name:linux-2.6.32.2
......
Load Address:30008000
Entry Point:30008000
Verifying Checksum...OK
XIP Kernel Image...OK
OK
Starting kernel...
undefined instruction
...
Resetting CPU...
resetting...
</code></pre>
> 在解决这个问题前有必要说一下uImage是如何生成的，因为出现这样的问题肯定与uImage的正确性有密切关系。
> 分析uImage是如何生成的，肯定是去分析Makefile了。
> 看一下顶层中的Makefile，其中：
<pre><code>zImage Image xipImage bootpImage uImage: vmlinux
	$(Q)$(MAKE) $(build)=$(boot) MACHINE=$(MACHINE) $(boot)/$@
</code></pre>
> 当我们执行`make uImage`时，运行其中的命令行，展开其中的变量，得到：
<pre><code>make -f scripts/Makefile.build obj=arch/arm/boot MACHINE=arch/arm/mach-s3c2410/ arch/arm/boot/uImage
</code></pre>
> 上面是展开变量得到的结果，还有一种简单的方法就是利用神器`grep`，执行：
<pre><code>#make uImage -n | grep "make -f"
...
make -f scripts/Makefile.build obj=net/sunrpc
make -f scripts/Makefile.build obj=net/unix
make -f scripts/Makefile.build obj=net/wireless
make -f scripts/Makefile.build obj=arch/arm/lib
make -f scripts/Makefile.build obj=lib
make -f scripts/Makefile.build obj=lib/zlib_deflate
make -f scripts/Makefile.build obj=lib/zlib_inflate
make -f /home/saiyn/work/linux-2.6.32.69/scripts/Makefile.modpost vmlinux.o
echo '  GEN     .version'; set -e; if [ ! -r .version ]; then rm -f .version; echo 1 >.version; else mv .version .old_version; expr 0$(cat .old_version) + 1 >.version; fi; make -f scripts/Makefile.build obj=init
make -f scripts/Makefile.build obj=arch/arm/boot MACHINE=arch/arm/mach-s3c2410/ arch/arm/boot/uImage
make -f scripts/Makefile.build obj=arch/arm/boot/compressed arch/arm/boot/compressed/vmlinux
</code></pre>
> 上面其实就是kbuild的工作实例，kbuild使用的典型方式如下：
<pre><code>$(MAKE) $(build)=\<subdir\> [target]
</code></pre>

> * 下面简单地阐述一下make的工作原理。
> 通过`-f`选项，指定Makefile为scripts目录下的Makefile.build。
> 而当make解释执行Makefile.build时，再将子目录中的Makefile包含到Makefile.include中来，
> 这就动态的组成子目录的真正的Makefile。
> make始终工作于顶层目录下，所以需要跟踪编译所在的子目录，为此，kbuild定义了两个变量：src和obj。
> 其中，src始终指向需要构建的目录，obj指向构建的目标存放的目录。所以在Makefile.build的
> 一开头，变量src的值为$(obj):
<pre><code>
/scripts/kbuild.include:
src := $(obj)
PHONY := __build
__build:
...
</code></pre>





