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
`$(if <condition>, <then-part>)`
> 或者是：
`$(if <condition>, <then-part>, <else-part>)`
> 如果\<condition\>为真则返回\<then-part\>否则返回\<else-part\>





