---
layout: post
title:  "Linux内核笔记之DMA_BUF"
date:   2018-04-18 15:15:54
categories: Linux-Kernel
excerpt: linux kernel dma-buf gpu drm prime
---

* content
{:toc}


内存管理始终是底层软件的核心部分，尤其是对于音视频的解码显示功能。本文将通过编写一个实例驱动程序，同内核中的i915显卡驱动进行内存方面的交互来剖析
Linux内核中的通用子系统DMA_BUF。

---


# 驱动实例程序

<br />

## 需求背景

<br />

考虑这样一种场景，摄像头采集的视频数据需要送到GPU中进行编码、显示。负责数据采集和编码的模块是Linux下不同的驱动设备，将采集设备中的数据送到编码设备中
需要一种方法。最简单的方法可能就是进行一次内存拷贝，但是我们这里需要寻求一种免拷贝的通用方法。

实际的硬件环境是，

<br />

## 
