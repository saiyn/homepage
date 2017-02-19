---
layout: post
title:  "软件架构师之路"
date:   2017-02-19 15:15:54
categories: architecture
excerpt: c architecture
---

* content
{:toc}

会写代码的很多，能写的好的不多，可以设计出很好架构的更是凤毛麟角。一直对代码的质量有洁癖，渴望写出架构清晰，易扩展，易维护的高质量代码。

面向对象的高级语言那么多，有人说C语言早就过时了，说这些话的人透露了他们的无知与肤浅。C语言是一粒一粒细小的沙子，那些面向对象的高层语言就像是一块一块成型的砖头。确实使用砖头砌墙的速度要比使用沙子筑墙的速度要快很多，方便很多。但是要记住有些砖头就是首先由沙子造出的。也不用以为只有使用面向对象的语言才能面向对象的编程。面向对象编程是一种编程思想，与使用哪种语言没有必然联系。看过太多的同事使用面向对象的语言写着完全面向过程的代码。而那些使用c语言写出高质量的令人拍案叫绝的面向对象的代码牛人真心难道。




---


## 基本功

### 如何组织好C的头文件

其实说来惭愧，工作快４年了还经常在实际写代码时碰到不知道如何组织好头文件的情况。能够组织好或者更准确地说能够懂得如何设计头文件，如何规划不同的数据结构到准确的头文件中，是一件非常考验代码基本功的东西。也是能设计出合理软件架构的前提。

A well organized C program has a good choice of modules, and properly constructed header files make it easy to understand and access the functionality in a module.

所以说，头谈架构是没有用的，首先得打好基础。
 
下面总结一下some rules how to set up your header and source files for 尽可能做到表达清晰:

**1.Each module with it's .h and .c file should 对应一个明确的功能.**

Conceptually, a module is a group of declarations and functions can be developed and maintained separately from other modules. Don't force together into a module things that will be used or maintained separately, and don't separate things that will always be used and maintained together.


**2.All of the declarations needed to use a module must appear in its header file,不要在其他模块中进行extern这样的hard-coded declaration**

If module A needs usr module X's functionality, it should always `#include "X.h"`, and never contain hard-coded declarations for structure or functions that appear in module X. Why? If module X is changed, but you forget to change the `hard-coded declaratons` in module A, module A could easily fail with subtle run-time errors that won't be detected by either the compiler or linker. This is a violation of the "One Definition Rule" which C compilers and linkers can't detect.


**3.If an incomplete declaration of a structure type X will do,use it instead of #incluing its header X.h.这个是最重要的一点技巧**

If a struct type appears only as a pointer type in a structure declaration or its function, and the code in the header does not attempt to access any member variables of X, the you should not #include X.h, but instead make `an incomplete declaration of X before the first use of X. Here is the example:

<pre><code>struct X; /*incomplete declaration*/

struct Thing{
	int i;
	struct X* x_ptr;
}
</code></pre> 

The compiler will be happy to accept code containing pointers to an incompletely known structure type, basically because pointers always have the same size and 特征,无论它们指向什么。`This ia a powerful technique for 封装一个模块以及解除和其他模块间的耦合度`

**4.The A.c file should first #include A.h file ,and then any others.**

Always #include A.h file to avoid hiding anything it is missing that gets included by other .h file.Then, if A's implementation code uses X, explicitly #include X.h in A.c, so A.c is not dependent on X.h accidentally being #included somewhere else.





---

