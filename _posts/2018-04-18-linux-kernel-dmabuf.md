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


# 驱动实例

<br />

## 需求背景

<br />

考虑这样一种场景，摄像头采集的视频数据需要送到GPU中进行编码、显示。负责数据采集和编码的模块是Linux下不同的驱动设备，将采集设备中的数据送到编码设备中
需要一种方法。最简单的方法可能就是进行一次内存拷贝，但是我们这里需要寻求一种免拷贝的通用方法。

实际的硬件环境是，采集设备是一个pciv驱动，编码设备就是i915驱动。现在就是要编写一个驱动程序，让i915驱动可以直接访问pciv中管理的视频数据内存。

<br />

## 驱动设计



---

<br />

# DMA_BUF

<br />

## 概述

<br />

**引入dma-buf机制的原因**

<br />

* 之前内核中缺少一个可以让不同设备、子系统之间进行内存共享的统一机制。

* 混乱的共享方法：

   * V4L2(video for Linux)使用`USERPTR`的机制来处理访问来自其他设备内存的问题，这个机制需要借助于以后空间的mmap方法。
   
   * 类似的，wayland和x11各自定义了客户端和主进程之间的内存共享机制，而且都没有实现不同设备间内存共享的机制。
   
   * 内核中各种soc厂商的驱动、各种框架和子系统都各自实现各自的内存共享机制。
  
* 之前共享方式存在问题：

   * 使用用户层的mmap机制实现内存共享方式太过简单粗暴，难以移植。
  
   * 没有统一的内存共享的API接口。

<br />

**dma_buf是一种怎样的存在**

<br />

dma_buf是内核中一个独立的子系统，提供了一个让不同设备、子系统之间进行共享缓存的统一框架，这里说的缓存通常是指通过DMA方式访问的和硬件交互的内存。
比如，来自摄像头采集的通过pciv驱动传输的内存、gpu内部管理的内存等等。

其实一开始，dma_buf机制在内核中的主要运用场景是支持GPU驱动中的`prime`机制，但是作为内核中的通用模块，它的适用范围很广。

dma_buf子系统包含三个主要组成:

1. dma-buf对象，它代表的后端是一个sg_table,它暴露给应用层的接口是一个文件描述符，通过传递描述符达到了交互访问dma-buf对象，进而最终达成了
共享访问sg_table的目的。

2. fence对象, which provides a mechanism to signal when one device as finished access.

3. reservation对象, 它负责管理缓存的分享和互斥访问。.

<br />

## dma-buf实现

<br />

**整体构架**

<br />

DMA_BUF框架下主要有两个角色对象，一个是`exporter`，相当于是buffer的生产者，相对应的是`importer`或者是`user`,即buffer的消费使用者。

假设驱动A想使用由驱动B产生的内存，那么我们称B为exporter,A为importer.

The exporter

   * 实现struct dma_buf_ops中的buffer管理回调函数。
   
   * 允许其他使用者通过dma_buf的sharing APIS来共享buffer。
   
   * 通过struct dma_buf结构体管理buffer的分配、包装等细节工作。
   
   * 决策buffer的实际后端内存的来源。
   
   * 管理好scatterlist的迁移工作。
      
The buffer-usr

   * 是共享buffer的使用者之一。
   
   * 无需关心所用buffer是哪里以及如何产生的。
   
   * 通过struct dma_buf_attachment结构体访问用于构建buffer的scatterlist,并且提供将buffer映射到自己地址空间的机制。

<br />

**数据结构**

<br />

	struct dma_buf{
		size_t size;
		struct file *file; /* file pointer used for sharing buffers across,and for refcounting */
		struct list_head attachments; /* list of dma_buf_attachment that denotes all devices attached */
		const struct dma_buf_ops *ops;
		struct mutex lock;
		unsigned vmapping_counter;
		void *vmap_ptr;
		const char *exp_name; /* name of the exporter; useful for debugging */
		struct module *owner;
		struct list_head list_node; /* node for dma_buf accounting and debugging */
		void *priv; /* exporter specific private data for this buffer object */
		
		struct reservation_object *resv; /* reservation object linked to this dma-buf */
		
		wait_queue_head_t poll;
		struct dma_buf_poll_cb_t{
			struct fence_cb cb;
			wait_queue_head_t *poll;
			unsigned long active;
		}cb_excl, cb_shared;
	};

dma_buf对象中最重要的成员变量是ops方法集，dma_buf本身是一个通用的框架，正是依靠这里的ops回调函数集来实现dma_buf对象的`重载功能`，所谓重载就是
说dma_buf框架可以用于不同的运用场景。所以ops定义的回调函数是我们编写dma_buf框架下exporter驱动的主要实现代码。

ops中定义的回调函数都对应着dma_buf模块外部头文件dma_buf.h中的API，比如其他驱动调用dma_buf.h中的dma_buf_attach()API时，实际最终调用的就是
我们实现的ops中的`int (*attach)(struct dma_buf *, struct device *, struct dma_buf_attachment *)`方法；调用dma_buf_map_attachment() API
实际就是调用ops中的`struct sg_table * (*map_dma_buf)(struct dma_buf_attachment *, enmu dma_data_direction)`方法。

dma_buf对象中更加重要的一个成员变量是file,我们知道`一切皆文件`是unix的核心思想。dma_buf子系统之所以可以使不同的驱动设备可以共享访问内存，就
是借助于文件系统是全局的这个特征。和dma_buf对象中file成员变量对应的API接口有，dma_buf_export()、dma_buf_fd()。

	/**
	 * dma_buf_export - Create a new dma_buf, and associates an anon file with this buffer,
	 * so it can be exported.
	 */
	struct dma_buf *dma_buf_export(const struct dma_buf_export_info *exp_info)
	{
		struct dma_buf *dma_buf;
		struct file *file;
		
		...
		
		dmabuf = kzalloc(alloc_size, GFP_KERNEL);
		if(!dmabuf){
			module_put(exp_info->owner);
			return ERR_PTR(-ENOMEM);
		}
		
		...
		
		dmabuf->ops = exp_info->ops; //[0]
		
		...
		
		file = anon_inode_getfile("dmabuf", &dma_buf_fops, dmabuf, exp_info->flags); //[1]
		file->f_mode |= FMODE_LSEEK;
		dmabuf->file = file;
		
		...
		
		return dmabuf;
	}


上面[0]出传入的ops方法集就是上面提到的我们编写驱动应该实现的回调函数集，dmabuf通过anon_inode_getfile()函数挂载到了file对象上的
priv指针上，而dma_buf_fops回调函数集挂载在file对象上的ops上，最后dma_buf_fops函数集中的回调函数实现都会通过file->priv拿到dma_buf对象，
然后直接调用dma_buf中的ops方法。这样的函数重载实现是file作为驱动程序接口功能实现的`常规操作`.




---

<br />







[dma_buf文档](https://elinux.org/images/a/a8/DMA_Buffer_Sharing-_An_Introduction.pdf)





