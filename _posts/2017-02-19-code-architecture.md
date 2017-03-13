---
layout: post
title:  "软件架构师之路"
date:   2017-02-19 15:15:54
categories: architecture
excerpt: c architecture
---

* content
{:toc}


![image](http://coolshell.cn//wp-content/uploads/2016/10/drawing-recursive-300x204.jpg)

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


## Unix哲学

### 模块性

模块化原则:要编写复杂软件又不至于一败涂地的唯一方法，就是用定义清晰的接口把若干简单模块组合起来，如此一来，多数问题只会出现在局部，并且在进行问题修复时，而不会出现牵动全身的局面。

**封装和最佳模块大小**

模块化代码的首要特质就是封装。封装良好的模块不会过多向外部暴露自身的细节，不会直接调用其他模块的实现码,也不会胡乱共享全局数据。模块之间应该通过一组严密，定义良好的程序接口和数据结构来通信。

一些最有能力的开发者，一开始总是定义接口，然后编写简要注释，对其进行阐述，最后才开始编写代码。

如果将模块大小设为x轴，bug数量或者模块可维护性设为y轴，那么绘制出来的曲线是一个`U`型曲线。Hatton曾经提出过一个模型，程序员可以短期记忆的最大模块大小的微小差别对其他的效率具有倍增效应。

模块小时，几乎所有复杂度都在于接口，想要理解任何一部分代码前必须理解全部代码，因此阅读代码非常困难。Hatton建议模块最佳物理代码行数为400-800。

**紧凑性和正交性**

紧凑性就是一个设计是否能够装进人脑中被轻松记住；正交性是有助于使复杂设计也能紧凑的最重要特性之一.

有科学文章称，人类短期记忆能够容纳的不连续信息数是七，这给了我们一个评测API紧凑性的很好的经验法则:编程者需要记忆的条目数大于七吗？

在通用语言中，c和python是半紧凑性的;perl,java和shell是非紧凑性的；而c++是反紧凑性的！！该语言的设计者已经承认，他根本就不指望有哪个程序员能够完全理解c++。

重构(refactoring)概念跟正交性紧密联系。重构代码就是改变代码的数据和组织，而不改变其外在行为。

对引起设计问题的特殊、意外的情况进行抽象、简化和概况，并尽量从中分离处理。


**胶合层**

一般来说，设计函数和对象的层次结构时可以选择两个方向，一个是自底向上(Bottom-To-Top),从具体到抽象;另一个是自顶向下(Top-To-Bottom),从抽象到具体。两种方向都有各自的优缺点，Unix哲学鼓励程序员
应该尽量双管齐下:一方面以自顶向下的应用逻辑来表达抽象规范，另一方面以函数和库来收集底层的域原语，这样，当高层设计变化时，这样域原语仍然可以重用。
当自顶向下和自底向上发送冲突时，其结果往往是一团糟。顶层的应用逻辑和底层的域原语必须使用`胶合层`逻辑来进行阻抗匹配(impedance match).
薄胶合层原则可以看作是分离原则的升华。策略(应用逻辑)应该与机制(域原语)清晰分离。如果许多代码即不属于策略又不属于机制，就很有可能除了增加系统的整体复杂度之外，没有任何其他用途。

oo语言使抽象变得很容易－－也许是太容易了。OO语言鼓励具有后重的胶合和复杂层次的体系。所有的OO语言都显示出某种使程序员陷入过度分层陷阱的倾向。

Unix风格程序设计所面临的主要挑战就是任何将分离法的优点同代码和设计的薄胶合，浅平透层次结构的优点相结合。





---














