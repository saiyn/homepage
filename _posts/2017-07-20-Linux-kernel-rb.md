---
layout: post
title:  "Linux内核笔记之红黑树"
date:   2017-07-20 15:15:54
categories: Linux-Kernel
excerpt: linux kernel rb-tree 红黑树
---

* content
{:toc}

最近一两年对于Linux内核部分的工作涉及主要是ALSA框架下的Intel HDA驱动和DRM/KMS框架下的Intel i915驱动，本系列记录研究这些内核代码时遇到的内核中的非常重要和常见的数据结构。

---

## 概念

红黑树(rb tree)是一种自平衡的binary search tree,主要用于存储或者说索引可排序的键值对数据。
它和内核中的radix树、hash表有很大不同，radix树比较合适用于存储sparse arrays，哈希表不同做到红黑树那样可以排序以及
存储任意大小的keys.


