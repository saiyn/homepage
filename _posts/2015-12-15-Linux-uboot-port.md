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
 在解决这个问题前有必要说一下uImage是如何生成的，因为出现这样的问题肯定与uImage的正确性有密切关系。
 分析uImage是如何生成的，肯定是去分析Makefile了。
> * 看一下顶层中的Makefile，其中：
<pre><code>zImage Image xipImage bootpImage uImage: vmlinux
	$(Q)$(MAKE) $(build)=$(boot) MACHINE=$(MACHINE) $(boot)/$@
</code></pre>
 当我们执行`make uImage`时，运行其中的命令行，展开其中的变量，得到：
<pre><code>make -f scripts/Makefile.build obj=arch/arm/boot MACHINE=arch/arm/mach-s3c2410/ arch/arm/boot/uImage
</code></pre>
 上面是展开变量得到的结果，还有一种简单的方法就是利用神器`grep`，执行：
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
 上面其实就是kbuild的工作实例，kbuild使用的典型方式如下：
<pre><code>$(MAKE) $(build)=\<subdir\> [target]
</code></pre>

> * 下面简单地阐述一下make的工作原理。

通过`-f`选项，指定Makefile为scripts目录下的Makefile.build。
而当make解释执行Makefile.build时，再将子目录中的Makefile包含到Makefile.include中来，
这就动态的组成子目录的真正的Makefile。
make始终工作于顶层目录下，所以需要跟踪编译所在的子目录，为此，kbuild定义了两个变量：src和obj。
其中，src始终指向需要构建的目录，obj指向构建的目标存放的目录。所以在Makefile.build的
一开头，变量src的值为$(obj):
<pre><code>
/scripts/kbuild.include:
src := $(obj)
PHONY := __build
__build:
...
</code></pre>

---

让我们继续回到uImage如何生成的问题上来
<pre><code>make -f scripts/Makefile.build obj=arch/arm/boot MACHINE=arch/arm/mach-s3c2410/ arch/arm/boot/uImage
</code></pre>
上面这句命令的效果就是去执行`arch/arm/boot`中的Makefile, 目标是`arch/arm/boot/uImage`,

> * 现在分析`arch/arm/boot`中的Makefile(Linux2.6.32)：
<pre><code>
$(obj)/uImage:	$(obj)/zImage FORCE
	$(call if_changed,uimage)
	@echo '  Image $@ is ready'
</code></pre>
因为目标是`arch/arm/boot/uImage`，所以首先执行的是上面部分，
uImage依赖zImage 和 FORCE，其中	FORCE是一个伪目标，作用就是不管zIamge是否最新，都要执行底下的命令行。
要弄懂`$(call if_changed,uimage)`这句的意思，得清楚`call`函数(见[Makefile 学习笔记](http://saiyn.github.io/homepage/2015/11/17/makefile-learn/))，以及`if_changed`函数。
`if_changed`定义在scripts/Kbuild.include文件中：
<pre><code>
if_changed = $(if $(strip $(any-prereq) $(arg-check)),                       \
	@set -e;                                                             \
	$(echo-cmd) $(cmd_$(1));                                             \
	printf '%s\n' 'cmd_$@ := $(make-cmd)' > $(dot-target).cmd)
</code></pre>
上面的命令比较复杂，不过看到`$(cmd_$(1))`这句我们大概知道就是要执行`cmd_uimage`, 看一下Makefile, 果然有这么一段：
<pre><code>
quiet_cmd_uimage = UIMAGE  $@
      cmd_uimage = $(CONFIG_SHELL) $(MKIMAGE) -A arm -O linux -T kernel \
		   -C none -a $(LOADADDR) -e $(STARTADDR) \
		   -n 'Linux-$(KERNELRELEASE)' -d $< $@
</code></pre>
Makefile的一开始就定义了`MKIMAGE`变量，`MKIMAGE         := $(srctree)/scripts/mkuboot.sh`,
`mkuboot.sh`脚本会去寻找'mkimage'程序，`mkimage`一般在U-Boot的`tools`目录下。所以我们得将其拷贝到我们的`/usr/local/bin`目录下。所以我们得将其拷贝到我们的`/usr/local/bin`目录
后面的`-A arm -O linux -T kernel -C none -a $(LOADADDR) -e $(STARTADDR) -n 'Linux-$(KERNELRELEASE)' -d $< $@`
都是传递给`mkimage`的参数，其中最重要的就是`$(LOADADDR)`和`$(STARTADDR)`，可以猜到，出现`undefined instruction`错误
就是这两个参数定义出了问题。

> * 分析`$(LOADADDR)`和`$(STARTADDR)`这两个参数的定义

`$(LOADADDR)`是定义U-Boot该从哪个地址加载Linux内核镜像，
<pre><code>
ifeq ($(CONFIG_ZBOOT_ROM),y)
$(obj)/uImage: LOADADDR=$(CONFIG_ZBOOT_ROM_TEXT)
else
$(obj)/uImage: LOADADDR=$(ZRELADDR)
endif
</code></pre>
因为没有定义`CONFIG_ZBOOT_ROM`，所以LOADADDR=$(ZRELADDR)
<pre><code>ZRELADDR    := $(zreladdr-y)

ifneq ($(MACHINE),)
include $(srctree)/$(MACHINE)/Makefile.boot
endif
</code></pre>
ZRELADDR依赖zreladdr-y，而zreladdr-y就定义在`$(srctree)/$(MACHINE)/Makefile.boot`中，即~/arch/arm/mach-s3c2410/Makefile.boot
<pre><code>root@ubuntu:~/work/linux-2.6.32.69#cat arch/arm/mach-s3c2410/Makefile.boot
   zreladdr-y	:= 0x30008000
params_phys-y	:= 0x30000100
</code></pre>
这样可以清楚的知道`-a $(LOADADDR)`，其实就是`-a 0x30008000`，即在0x30008000处加载内核。
清楚了$(LOADADDR)，那么$(STARTADDR)就清楚了。
<pre><code>
ifeq ($(CONFIG_THUMB2_KERNEL),y)
# Set bit 0 to 1 so that "mov pc, rx" switches to Thumb-2 mode
$(obj)/uImage: STARTADDR=$(shell echo $(LOADADDR) | sed -e "s/.$$/1/")
else
$(obj)/uImage: STARTADDR=$(LOADADDR)
endif
</code></pre>
因为没有定义CONFIG_THUMB2_KERNEL，所以STARTADDR=$(LOADADDR)，即加载地址等于入口地址，这显然是不对的，因为加载地址后的40个字节放的是
u-boot传入的参数，至此终于搞清楚为什么会出现`undefined instruction`错误了。



