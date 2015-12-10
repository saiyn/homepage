---
layout: post
title:  "Makefile 笔记"
date:   2015-11-17 15:15:54
categories: Linux
excerpt: linux makefile
---

* content
{:toc}

记录开发中实际涉及的Makefile知识

---

##if 函数
> 之前一直混淆`if`函数和条件语句`ifeq`.
> if函数的语法是：
 : `$(if <condition>, <then-part>)`
> 或者是：
 : `$(if <condition>, <then-part>, <else-part>)`
>
> 如果\<condition\>为真则返回\<then-part\>否则返回\<else-part\>.

---

##伪目标
> * 没有依赖的伪目标
> 我们经常会写出一下的makefile内容：
<pre><code>clean:
	rm *.o temp
</code></pre>
> `clean`就是一个伪目标。伪目标并不是一个文件，只是一个标签。
> 由于“伪目标”不是文件，所以`make`无法生成它的依赖关系和决定它是否要执行。
> 所以我们只能显示地指明这个“目标”才能让其生效。
> 为了避免和其他文件重名的情况，我们可以使用一个特殊的标记“.PHONY”来显示地指明一个目标是“伪目标”，其作用是向`make`说明，不管是否有这个文件，这个目标就是伪目标.
> 
> `.PHONY : clean`
> 
> * 既没有依赖也没有规则的伪目标
> 在Linux源码主目录下的`makefile`中有大量的`FORCE`伪目标，
> 它的定义如下：
<pre><code># vmlinux image - including updated kernel symbols
vmlinux: $(vmlinux-lds) $(vmlinux-init) $(vmlinux-main) vmlinux.o $(kallsyms.o) FORCE

PHONY += FORCE
FORCE:
.PHONY:$(PHONY)
</code></pre>
> 







