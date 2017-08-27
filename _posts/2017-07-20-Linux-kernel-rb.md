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
它和内核中的radix树、hash表有很大不同，radix树比较合适用于存储sparse arrays，哈希表做不到红黑树那样可以排序以及
存储任意大小的keys.

红黑树和AVL树比较像，但是红黑树有更好的插入和删除性能(最多只要2到3次旋转调节);同时红黑树搜索时间复杂度为O(log n)。红黑树在Linux内核中使用很广，ext3文件系统就使用红黑树记录目录项,VMA(virtual memory ares)的管理，高精度计时器(high-resolution timer)使用红黑树组织定时请求，还有"等级令牌桶"(hierachical token bucket)调度的网络数据包使用红黑树进行组织等等。

红黑树的特性总结如下:

* 一个节点要么是红色的，要么是黑色的。
* 根节点是黑色的。
* 所有leaves(NULL)都是黑色的。
* 如果一个节点是红色的，那么它的子节点必须是黑色的，也就是说沿着从根节点出发的任何路径上都不会出现连续的红色节点。
* 从一个节点到一个leave(NULL)指针的每个路径上必须包含相同数目的黑色节点。

![rb_tree](http://omp8s6jms.bkt.clouddn.com/image/git/rb_tree.png)




---
## 结构体定义

Linux中的红黑树实现在"lib/rbtree.c"文件中，查看它的结构体定义就可以发现，它的使用和内核中的链表(list)有些类似，都是将结构体嵌入到自定义的数据结构中。

<pre><code>/* include/linux/rbtree.h*/

struct rb_node{
	unsigned long  __rb_parent_color;
	struct rb_node *rb_right;
	struct rb_node *rb_left;
}__attribute__((aligned(sizeof(long))));

</code></pre>

这个结构体非常的讲究，首先它是`sizeof(long)`字节对齐的，long在32位系统下是4字节,64位的是8字节；由于结构体中的
首个字段地址就是整个结构体的地址，所以意外着`__rb_parent_color`至少是4字节对齐的，那么分配给`__rb_parent_color`字段
地址值的最后两个bits始终是0，可以用来存储额外的信息。事实也真是这样，`__rb_parent_color`保存了2个信息:

* 父节点的地址
* 本节点的颜色

另外，从`__rb_parent_color`变量名前有意的加上`__`2个下划线可以看出，不建议直接操作这个变量，而是应该借助辅助宏或者
函数。下面就来看一下相关的重要的宏定义。

<pre><code>/* include/linux/rbtree.h */

#define  rb_parent(r) ((struct rb_node*)((r)->__rb_parent_color & ~3))

/* 'empty' nodes are nodes that are known not be inserted in an rbtree */
#define RB_EMPTY_NODE(node)	\
	((node)->__rb_parent_color == (unsigned long)(node))

#define RB_CLEAR_NODE(node)	\
	((node)->__rb_parent_color = (unsigned long)(node))

/* 其他一些辅助宏　*/

#define RB_RED   0
#define RB_BLACK 1

#define RB_ROOT (struct rb_root){NULL,}

#define rb_entry(ptr, type, member) contaier_of(ptr,type, member)

#define RB_EMPTY_ROOT(root) (READ_ONCE((root)->rb_node) == NULL)

/* 设置parent的相关内联函数　*/

static inline void rb_set_parent(struct rb_node *rb, struct rb_node *p)
{
	rb->__rb_parent_color |= (unsigned long)p; /* 不影响rb节点的颜色属性　*/
}

static inline void rb_set_parent_color(struct rb_node *rb, struct rb_node *p, int color)
{
	rb->__rb_parent_color = (unsigned long)p | color; /* 指定rb节点的颜色属性　*/
}

/* 初始化新节点的内联函数　*/

static inline void rb_link_node(struct rb_node *node, struct rb_node *parent, struct rb_node **rb_link)
{
	/**
     * 设置其双亲节点的首地址(根节点的双亲节点为NULL),且颜色属性为黑色
	 */
	node->__rb_parent_color = (unsigned long)parent;

	node->rb_left = node->rb_right = NULL;
	/**
	 * 指向新节点
	 */
	*rb_link = node;

}

</code></pre>

---
## 具体实现

为了实现性能的优化，linux中的红黑树的实现比其他传统的树实现少了一层抽象间接层。所以我们看到，linux中的红黑树没有
使用指向数据的rb_node指针进行数据的组织和管理，而是将rb_node数据结构作为实例直接嵌入到数据的结构体中。由于基于这样的
数据结构设计，我们需要自己实现我们的查找、插入、删除等功能函数。

下面通过一个简单的实例来研究linux中红黑树的实现:

<pre><code>struct mytype{
		struct rb_node node;
		char *keystring;
	};

</code></pre>

### 插入

红黑树的插入主要分为:查找插入点、插入然后进行再平衡两个主要步骤。

<pre><code>
int my_insert(struct rb_root *root, struct mytype *data)
{
	struct rb_node **new = &(root->rb_node), *parent = NULL;

	/* Figure out where to put new node*/
	while(*new){
		struct mytype *this = contianer_of(*new, struct mytype, node);
		int result = strcmp(data->keystring, this->keystring);

		parent = *new;

		if(result < 0)
			new = &((*new)->rb_left);
		else if(result > 0)
			new = &((*new)->rb_right)
		else
			return FALSE;
	
	}

	/* Add new node and rebalance tree. */
	rb_link_node(&data->node, parent, new);
	rb_insert_color(&data->node, root);

	return TRUE;
}
</code></pre>

从上面的代码我们可以看出，我们只需要操作我们自己定义的数据部分，而核心的红黑树数据的操作交给linux内核实现的接口就可以了。这样我们通过调用几个核心的api就可以实现使用红黑树的算法来组织我们的数据了。上面代码中核心api就是`rb_insert_color()`函数，下面进行具体分析。


先看一下这个核心插入api函数的实现的总体框架

<pre><code>static __always_inline void
__rb_insert(struct rb_node *node, struct rb_root *root,
		void (*augment_rotate)(struct rb_node *old, struct rb_node *new))
{
	struct rb_node *parent = rb_red_parent(node), *gparent, *tmp;

	while(1){
		/**
		 * Loop invarint: node is red
		 *
		 * If there is a black parent, we are done.
		 * Otherwise, take some corrective action as we don't
		 * Want a red root or two consecutive red nodes.
		 */

		if(!parent){
			rb_set_parent_color(node, NULL, RB_BLACK);
			break;
		}else if(rb_is_black(parent))
			break;
		
		gparent = rb_red_parent(parent);

		tmp = gparent->rb_right;

		if(parent != tmp){ /* parent == gparent->rb_left */
			if(tmp && rb_is_red(tmp)){
				/**
				 * case 1 - color flips
				 */

				...

				continue;
			}	
			

			tmp = parent->rb_right;
			if(node == tmp){
				/**
				 * case 2 - left rotate at parent
				 */
 				
				...
			}
	
			/**
   			 * case 3 - right rotate at gparent
			 */		

				...

				break;
		}else{
			
			...
		}
		
	}

}

</code></pre>

分析上面的代码框架可见，rb_insert_color函数使用ｇｔwhile循环不断判断插入新节点的双亲是否存在，双亲的颜色是不是红色。
如果双亲不存在，说明当前红黑树为空，调用rb_set_parent_color函数设置双亲节点的为黑色后直接退出即可。

如果双亲节点存在，但是双亲颜色是黑色的，那么也不需要进行再平衡。所以，如果双亲节点存在且是红色，那么就需要一直进行下面的再平衡处理。上面的第一层判断依据是双亲节点是祖父节点的左子树还是右子树，根据二叉树的平衡性可知，第一层判断下的两部分处理应该是一样的，所以下面只分析其中一种情况，即双亲节点是祖父节点的左子树。此时，又分成3个case去处理，下面是3个处理的算法示意图:

* a)存在叔父节点，且叔父节点为红色时，调整双亲、叔父、祖父节点的颜色，然后继续while循环判断。

![g_r_c](http://omp8s6jms.bkt.clouddn.com/image/git/g_r_c.png)

<pre><code>if(tmp && rb_is_red(tmp)){
	rb_set_parent_color(tmp, gparent, RB_BLACK);/* 设置叔父节点为黑色　*/
	rb_set_parent_color(parent,gparent,RB_BLACK);/* 设置父节点为黑色　*/
	node = gparent;
	parent = rb_parent(node);
	rb_set_parent_color(node,parent,RB_RED); /* 设置祖父节点为红色　*/
	continue;
}
</code></pre>

* b)当node节点是其双亲节点右子树的根时，则在parent节点处左旋，然后执行第c步。

![n_l_x](http://omp8s6jms.bkt.clouddn.com/image/git/n_l_x.png)

<pre><code>tmp = parent ->rb_right;
if(node == tmp){
	tmp = node->rb_left;
	WRITE_ONCE(parent->rb_right,tmp);
	WRITE_ONCE(node->rb_left,parent);
	if(tmp)
		rb_set_parent_color(tmp,parent,RB_BLACK);

	rb_set_parent_color(parent, node, RB_RED);
	augment_rotate(parent,node);
	parent = = node;
	tmp = node->rb_right;

}

</code></pre>

* c)当node是其双亲节点左子树的根时,在祖父节点处右旋即可。

![z_l_s](http://omp8s6jms.bkt.clouddn.com/image/git/z_l_s.png)

<pre><code>WRITE_ONCE(gparent->rb_left,tmp); /* == parent->rb_right */
WRITE_ONCE(parent->rb_right,gparent);
if(tmp)
	rb_set_parent_color(tmp,gparent,RB_BLACK);

__rb_rotate_set_parents(gparent,parent,root,RB_RED);
break;

</code></pre>

---
### 删除

对于删除一个已经存在的节点，首先我们自己实现通过键值寻找到相应的节点，然后调用`rb_erase()`进行实际的删除操作。事例代码如下:

	struct mytype *data = mysearch(&mytree, "saiyn");

	if(data){
		rb_erase(&data->node, &mytree);
		myfree(data);
	}


下面来具体分析`rb_erase()`函数的实现:

	void rb_erase(struct rb_node *node, struct rb_root *root)
	{
		struct rb_node *rebalance;

		rebalance = __rb_erase_augmented(node, root, &dummy_callbacks);
		if(rebalance)
			____rb_erase_color(rebalance, root, dummy_ratate);
	}


上面函数中，通过`__rb_erase_augmented()`函数先进行节点的删除，有些节点删除后需要进行平衡调整的则继续调用`____rb_erase_color()`函数。__rb_erase_augmented()的函数骨架如下:

	struct rb_node *__rb_erase_augmented(struct rb_node *node, struct rb_root *root, const struct rb_augment_callbacks *augment)
	{
		struct rb_node *child = node->rb_right;
		struct rb_node *tmp = node->rb_left;

		if(!tmp || !child){
			/**
			 * case 1: node to erase has no more then 1 child.
			 *
			 * Note that if there is one child it must be red duo to 5)
			 * and node must be black duo to 4). We adjust colors locally
			 * so as to bypass __rb_erase_color() later on.
			 */
							
			...


		}else{
			
			struct rb_node *successor = child, *child2;

			tmp = child->rb_left;

			if(!tmp){
				/**
				 * case 2: node's successor is its right child
				 *
				 */
			
				...

			}else{

				/**
				 * case 3: node's successor is leftmost under
				 * node's right child subtree
				 */

				...
			}
	
			...

		}
		
		augment->propagate(tmp,NULL);
		return rebalance;
	}


从上面骨架代码可见，删除节点时，主要分成３种情况，下面结合示意图分析具体实现细节:

* 1) 待删除的节点只有一个或者没有子树,这种情况最简单。

![no_l_r](http://omp8s6jms.bkt.clouddn.com/image/git/no_l_o_r.png)

	if(!tmp){ /* 无左子树 */
		pc = node->__rb_parent_color;
		parent = __rb_parent(pc);
		__rb_change_child(node, child, parent, root); /* 将node替换成child */
		if(child){
			/**
			 * 如果右子树存在，则更新一下其属性即可。
			 */
			child->__rb_parent_color = pc;
			rebalance = NULL;
		}else{
			/**
			 * 如果右子树也不存在，那么如果被删除节点是黑色的那么就需要进行再平衡处理。
			 */	
			rebalance = __rb_is_black(pc) ? parent : NULL;
		}
		tmp = parent;
	}else if(!child){ /* 无右子树 */
		tmp->__rb_parent_color = pc = node->__rb_parent_color;
		parent = __rb_parent(pc);
		__rb_change_child(node, tmp, parent, root);
		rebalance = NULL;
		tmp = parent;
	}


* 2) 待删除节点的successor 是他的右子树。这里需要解释一下binary tree中的`in-order predecessor`和`in-order successor`概念。

![pre_suc](http://omp8s6jms.bkt.clouddn.com/image/git/pre_succ.png)

如上图所示，节点4就是节点3的`in-order successor`，意思据说4节点是3节点左子树中排序上紧跟其后的节点。节点8的`in-order predecessor`是节点7。概念清楚后，这里的case2就很好理解了。

![case2](http://omp8s6jms.bkt.clouddn.com/image/git/case2.png)

	struct rb_node *successor = child, *child2;

	tmp = child->rb_left;
	if(!tmp){
		parent = successor;
		child2 = sucessor->rb_right;
	}


* 3) 待删除节点的successor是其右子树中的leftmost,这种情况最为复杂。

![case3](http://omp8s6jms.bkt.clouddn.com/image/git/case3.png)

	do{
		parent = successor;
		successor = tmp;
		tmp = tmp->rb_left;
	}while(tmp);

	child2 = successor->rb_right;
	WRITE_ONCE(parent->rb_left, child2);
	WRITE_ONCE(successor->rb_right, child);
	rb_set_parent(child, successor);


经过上面的分类处理之后，有些情况需要调用`____rb_erase_color()`函数继续进行再平衡处理，该函数实现较为复杂，这里不再展开。

### 遍历


