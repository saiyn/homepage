---
layout: post
title:  "Makefile 笔记"
date:   2015-11-17 15:15:54
categories: Linux-Application
excerpt: linux makefile
---

* content
{:toc}

记录开发中实际涉及的Makefile知识.
参考链接：

* [GNU MAKK](https://www.gnu.org/software/make/manual/make.html#Prerequisite-Types)

---

# CMAKE

<br />

[参考文档](https://internalpointers.com/post/modern-cmake-beginner-introduction)


<br />

## 项目实例

<br />

在部署jenkins的ci时，需要将jenkins的任务序号编入程序的版本号，这时需要向cmake传递参数到代码中的宏。
实现方法是使用target_compile_definitions()方法。CMakeLists.txt中的用法如下:

	set(SDK_TARGET_LIB_NAME "DRScanner")
	add_library(${SDK_TARGET_LIB_NAME} SHARED ${SOURCE_FILES_SDK})
	...
	
	target_compile_definitions(${SDK_TARGET_LIB_NAME} PUBLIC PROGVER=${PROGVER})
	
	...
	
target_compile_definitions()的具体用法可以参照[这里](https://cmake.org/cmake/help/latest/command/target_compile_definitions.html)

执行`cmake -D PROGVER=\"1.6.131\" ..` 传入参数。

这时会在`CMakefiles/xxx/flags.make`中生成`CXX_DEFINES = -DPROGVER=\"1.6.131\"`。



<br />

# 基础知识

掌握makefile中的基本语法和常用函数是构建make编译环境最基本的要求，但是对于构建复杂一点的工程，仅掌握这些也是不够的。

<br />

## Secondary Expansion

<br />

makefile中可以使用`.SECONDEXPANSION:`和`$$()`的搭配可以运用make的两种工作阶段的特性实现一些特别的需求。

所谓的make的两种工作阶段和secondary expansion的具体细节参加[make doc 3.9](https://www.gnu.org/software/make/manual/html_node/Secondary-Expansion.html)



<br />

## if 函数

<br />


之前一直混淆`if`函数和条件语句`ifeq`,if函数的语法是：

	$(if <condition>, <then-part>)
	
或者是：

	$(if <condition>, <then-part>, <else-part>)

如果\<condition\>为真则返回\<then-part\>否则返回\<else-part\>.

---

## 伪目标

<br />

**没有依赖的伪目标**

我们经常会写出一下的makefile内容：

	clean:
		rm *.o temp
		
`clean`就是一个伪目标。伪目标并不是一个文件，只是一个标签。
由于“伪目标”不是文件，所以make无法生成它的依赖关系和决定它是否要执行。
所以我们只能显示地指明这个“目标”才能让其生效。
为了避免和其他文件重名的情况，我们可以使用一个特殊的标记“.PHONY”来显示地指明一个目标是“伪目标”，其作用是向make说明，不管是否有这个文件，这个目标就是伪目标.
 
	.PHONY : clean
	
<br />	
	
 
 **既没有依赖也没有规则的伪目标**
 
在Linux源码主目录下的makefile中有大量的`FORCE`伪目标，它的定义如下：

	# vmlinux image - including updated kernel symbols
	vmlinux: $(vmlinux-lds) $(vmlinux-init) $(vmlinux-main) vmlinux.o $(kallsyms.o) FORCE

	PHONY += FORCE
	FORCE:
	.PHONY:$(PHONY)

---

<br />

## 变量

<br />



### 变量的定义与引用

<br />

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

<br />

### 自动化变量

<br />

所谓自动化变量，就是这种变量会把**模式中**所定义的一系列的文件自动地挨个取出，直至所有的符合模式的文件都取完。这种自动化变量只应出现在规则的命令中。下面逐个介绍常用的一些自动化变量：

**$<**

依赖目标中的第一个目标名字。特别要记住的是，如果依赖目标是以模式(即“%”)定义的，比如常见的`%.o:%.c`，那么"$<"将是符合模式的一系列的文件集，而是**一个一个**取出来的。

<br />

**$^**

所有的依赖目标的集合。以空格分隔。如果在依赖目标中有多个重复的，那么这个变量会去除重复的依赖目标，只保留一份。


**$(@F)**

目标文件路径名的部分。比如`$@`的是`dir/foo.o`，那么`$(@F)`就等于foo.o。




其他更详细的自动化变量的定义参加[GNU_doc](https://www.gnu.org/software/make/manual/html_node/Automatic-Variables.html)


---

<br />

## patsubst函数

<br />

patsubst是模式字符串替换函数，它的语法是:

	$(patsubst <pattern>, <replacement> , <text>)
	

功能是查找`<text>`中的单词是否符合模式`<pattern>`,如果匹配的话，则以`<replacement>`替换。这里，`<pattern>`中可以包括通配符"%",表示任意长度的
字符串。如果`<replacement>`中也包含"%",那么，`<replacement>`中的这个"%"将是`<pattern>`中的那个"%"所代表的字符串。

<br />

**实际应用**

一个常见的实际需求是，给目标文件或者依赖文件添加其所在目录信息，因为稍微复杂一点的工程，各种文件会位于不同的子目录下面。

	IDIR =../include
	CC=gcc
	CFLAGS=-I$(IDIR)

	ODIR=obj
	LDIR =../lib

	LIBS=-lm

	_DEPS = hellomake.h
	DEPS = $(patsubst %,$(IDIR)/%,$(_DEPS))

	_OBJ = hellomake.o hellofunc.o 
	OBJ = $(patsubst %,$(ODIR)/%,$(_OBJ))


	$(ODIR)/%.o: %.c $(DEPS)
		$(CC) -c -o $@ $< $(CFLAGS)

	hellomake: $(OBJ)
		$(CC) -o $@ $^ $(CFLAGS) $(LIBS)

	.PHONY: clean

	clean:
		rm -f $(ODIR)/*.o *~ core $(INCDIR)/*~ 




## foreach 函数

<br />

foreach函数是用来做循环处理的，就想c语言中的for一样。它的语法是：

	$(foreach <var>, <list>, <text>)

这个函数的语法就是，把参数list中的单词逐一取出放到参数var所指定的变量中，然后再执行text所包含的表达式。每次text会
返回一个字符串，循环过程中，所返回的字符串都以空格分隔，最后当整个循环结束时，text所返回的每个字符串所组成的整个字符串
将会死foreach函数的返回值。

<br />

**实际应用**

一个经常遇见的场景是，我们需要链接多个目录下多个.a库生成可执行文件，对于一个目录下的所有.a文件我们可以这样做:

	LIBS := $(wildcard /lib/path/*.a)
	
对于多个目录，我们自然而然的想法是，使用循环自动对每个目录进行上面的操作，于是我们可以这样做:

	LIBS := $(foreach dir, $(LIB_PATH), $(wildcard $(dir)/*.a))
	
这样LIB_PATH指定的多个目录下的所有.a文件都被包含了。

<br />

## findstring函数

<br />

**实际应用**

* `ifneq ($(findstring $(ARCH), diamond), $(ARCH))`

如上，如果`$(ARCH)`中包含diamond字符串，那么`$(findstring($ARCH), diamond)`返回`$(ARCH)`，否则返回空。
这样就实现了字符串的模糊匹配功能。


---

<br />

# 实战经验

<br />

## order-only prerequisite的应用

makefile中的依赖有两种，一种是如下我们常见的`normatl-prerequisite`, 另外一种就是跟在`|`符号后面的`order-only prerequisite`。

	target: normal-prerequisite | order-only prerequisite
		recipe
		
我们知道，对于normal-prerequisite, 只要target所依赖的时间戳比自己新，或者依赖不存在，make都会执行recipe, 但是有些情况下，我们只希望依赖比target提前存在即可，而不需要每次因为时间戳而执行recipe。这时就可以使用order-only prerequisite。

<br />

* 应用1 - 将目标文件放置到特定目录

<br />

	OBJ_DIR = ./build/obj
	OBJS := $(addprefix $(OBJ_DIR)/, foo.o bar.o)
	
	$(OBJ_DIR)/%.o: %.c | $(OBJ_DIR) [1]
		$(CC) -c $< -o $@ $(CFLAGS)		

	all: $(OBJS)


	$(OBJ_DIR):
		mkdir -p $(OBJ_DIR)



如上，如果[1]处的依赖不是order-only prerequisite，那么因为每编译生成一个.o文件，OBJ_DIR目录的时间戳都会更新一次，导致`mkdir -p $(OBJ_DIR)`这个recipe被多次执行



<br />













