---
layout: post
title:  "Makefile 笔记"
date:   2015-11-17 15:15:54
categories: Linux-Application
excerpt: linux makefile
---

* content
{:toc}

记录开发中实际涉及的Makefile知识

---

# if 函数
---

之前一直混淆`if`函数和条件语句`ifeq`,if函数的语法是：

$(if <condition>, <then-part>)
	
或者是：

$(if <condition>, <then-part>, <else-part>)

如果\<condition\>为真则返回\<then-part\>否则返回\<else-part\>.

---

# 伪目标
---

**没有依赖的伪目标**

我们经常会写出一下的makefile内容：

	clean:
		rm *.o temp
		
`clean`就是一个伪目标。伪目标并不是一个文件，只是一个标签。
由于“伪目标”不是文件，所以make无法生成它的依赖关系和决定它是否要执行。
所以我们只能显示地指明这个“目标”才能让其生效。
为了避免和其他文件重名的情况，我们可以使用一个特殊的标记“.PHONY”来显示地指明一个目标是“伪目标”，其作用是向make说明，不管是否有这个文件，这个目标就是伪目标.
 
	.PHONY : clean
 
 **既没有依赖也没有规则的伪目标**
 
在Linux源码主目录下的makefile中有大量的`FORCE`伪目标，它的定义如下：

	# vmlinux image - including updated kernel symbols
	vmlinux: $(vmlinux-lds) $(vmlinux-init) $(vmlinux-main) vmlinux.o $(kallsyms.o) FORCE

	PHONY += FORCE
	FORCE:
	.PHONY:$(PHONY)

---

# 变量
---

### 变量的定义与引用

变量在声明时需要给予初值，赋值的方法有好几种，根据不同需要要使用不同的方法。
最简单的和其他编程语言一样是使用“=”号，"="左侧是变量，右侧是变量的值，而且特点就是变量的定义可以在文件任何一处，
也就是说，右侧的变量可以使用后面定义的值，如：
<pre><code>foo = $(A)
A = $(B)
B = Saiyn
all:
	echo $(foo)
</code></pre>
 我们执行`make all`将会打印出$(foo)的值是“Saiyn”
 但是这种定义变量的方法有递归定义无限循环的问题，如：
<pre><code>A = $(B)
B = $(A)
</code></pre>
 为了避免上面的问题，就引出了另外一种方法：“:=”操作符。
 这种方法就是只能使用前面已经定义好的变量，例如：
<pre><code>y := $(x) bar
x := foo
</code></pre>
 那么，y的值是“bar”, 而不是“foo bar”。
 还有一种非常类似于`C`语言中的`_weak`属性的操作符“?=”。意思就是如果变量的值没有定义过，那就使用这个值。
 但是如果被定义过，那么这句话就直接被忽略。
 我们可以使用操作符"+="来给变量追加值,例如：
<pre><code>obj = main.o foo.o
obj += another.o 
</code></pre>
 于是， 我们的$(obj)值变成："main.o foo.o another.o"
 值得注意的是，如果变量没有定义过，"+="会自动变成"="，如果前面有变量定义，那么"+=“
 会继承于前面操作的赋值符。例如：
<pre><code>obj := main.o
obj += another.o 
</code></pre>
等价于
<pre><code>obj := main.o
obj := $(obj) main.o
</code></pre>



---












