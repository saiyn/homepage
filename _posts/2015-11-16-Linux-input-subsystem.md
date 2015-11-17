---
layout: post
title:  "Linux input subsystem 浅析"
date:   2015-11-16 11:48:54
categories: Linux
excerpt: Linux input driver
---

* content
{:toc}

最近一个项目中需要实现几个按键功能，按键驱动如果用misc设备驱动架构还是比较简单的，但是如果将按键功能加入内核`input`子系统，事情就变得复杂了。本文就是借此机会去浅析一下`input`子系统的实现与使用，为以后更深入的开发做点笔记。


---

##站在高处总体把握一下这个逼格较高的子系统
> 分析一个架构首先肯定是要分析一下这个架构中用到的主要数据结构，但是如果一开始就展开所有的结构体，那一下就晕了。
> 主角登场：
>
> * input.c 我们把他看成市委书记，简称INPUT书记，他的职责就是统筹负责这个`input subsystem`系统。这个书记手里有一个大箱子叫`input_table[]`。
>
> * evdev.c 我们把他看成“事件处理部门”简称EV部门， 他下属于上面的INPUT书记。但是一开始是没有部长的，因为这个部门的规定就是，只有在有事件处理时，才招聘一个部长。
> 这个部门的部长叫`struct evdev`，简称为evdev. 但是一开始，这个部门并没有这个部长，
> 想要成为部长，本事得有，简历得漂亮，简历的模板是`struct input_handler`， 一看并知道这个是市里面定的模板。候选的部长写好自己的简历，我们不妨来看一看：
<pre><code>static struct input_handler evdev_handler = {
	.event		= evdev_event,
	.connnect 	= evdev_connect,
	.disconnect = evdev_disconnect,
	.fops		= &evdev_fops,
	.minor		= EVDEV_MINOR_BASE,
	.name		= "evdev",
	.id_table	= evdev_ids,
};
</code></pre>
> 好了， 从简历中我们看出，这个部长候选人的简历名字是`evdev_handler`。
>
> * 这时可以调用`input_register_handler(&evdev_handler);`去向市里面申请一个部长。
> 这时我们来好好看看市里面是怎样“审核、注册、任用”一个部长的。其实就是分析一下函数`int input_register_handler(struct input_handler *handler)`的实现。
<pre><code>if (handler->fops != NULL) {
		if (input_table[handler->minor >> 5]) {
			retval = -EBUSY;
			goto out;
		}
		input_table[handler->minor >> 5] = handler;
	}
</code></pre>
> 从什么一段代码可见，书记都看看这个候选部长是否具有基本的“VFS 系统操作技能”， 也就是是是否有`->fops`，如果没有直接让他滚蛋，如果有，好，书记就把把候选部长的简历`evdev_handler`放到我们上面提到的大箱子`input_table[]`中。
>
> * 从`handler->minor >> 5`这个处理细节来看，我们的书记可不是随便放他的简历的，其中意义我们后面再细细道来。
<pre><code>list_add_tail(&handler->node, &input_handler_list);
</code></pre>
> 从上面这行代码看，书记手里还有一个列表叫`input_handler_list`， 用来记住这个候选人，以备后用。
<pre><code>list_for_each_entry(dev, &input_dev_list, node)
		input_attach_handler(dev, handler);
</code></pre>
> 终于看到了书记是怎样筛选候选人了，书记在`input_dev_list`里面查看有没有“事件”是我们这个候选部长可以胜任的，如果有赶紧就让这个候选人去上任。具体看下面代码：
<pre><code>
static int input_attach_handler(struct input_dev *dev, struct input_handler *handler)
{
	const struct input_device_id *id;
	int error;

	if (handler->blacklist && input_match_device(handler->blacklist, dev))
		return -ENODEV;

	id = input_match_device(handler->id_table, dev);
	if (!id)
		return -ENODEV;

	error = handler->connect(handler, dev, id);
	if (error && error != -ENODEV)
		printk(KERN_ERR
			"input: failed to attach handler %s to device %s, "
			"error: %d\n",
			handler->name, kobject_name(&dev->dev.kobj), error);

	return error;
}
</code></pre>
> 上面有书记筛选的具体方法，我们暂时不去细看，以后再分析，我们重点先关注一下其中的`error = handler->connect(handler, dev, id);`，我们在上面提到过这位候选部长的简历，
> 简历中就有一项`.connect = evdev_connect`，这个`connect`就是部长应用的组建自己部门的能力。只有在他通过这个能力组建好自己的部门，他才能算是真正的部长了。
>
> * 好，现在我们重点看看这位部长是如何组建自己的部门的：
<pre><code>
static int evdev_connect(struct input_handler *handler, struct input_dev *dev,
			 const struct input_device_id *id)
{
	struct evdev *evdev;
	int minor;
	int error;

	for (minor = 0; minor < EVDEV_MINORS; minor++)
		if (!evdev_table[minor])
			break;

	if (minor == EVDEV_MINORS) {
		printk(KERN_ERR "evdev: no more free evdev devices\n");
		return -ENFILE;
	}

	evdev = kzalloc(sizeof(struct evdev), GFP_KERNEL);
	if (!evdev)
		return -ENOMEM;

	INIT_LIST_HEAD(&evdev->client_list);
	spin_lock_init(&evdev->client_lock);
	mutex_init(&evdev->mutex);
	init_waitqueue_head(&evdev->wait);

	dev_set_name(&evdev->dev, "event%d", minor);
	evdev->exist = 1;
	evdev->minor = minor;

	evdev->handle.dev = input_get_device(dev);
	evdev->handle.name = dev_name(&evdev->dev);
	evdev->handle.handler = handler;
	evdev->handle.private = evdev;

	evdev->dev.devt = MKDEV(INPUT_MAJOR, EVDEV_MINOR_BASE + minor);
	evdev->dev.class = &input_class;
	evdev->dev.parent = &dev->dev;
	evdev->dev.release = evdev_free;
	device_initialize(&evdev->dev);

	error = input_register_handle(&evdev->handle);
	if (error)
		goto err_free_evdev;

	error = evdev_install_chrdev(evdev);
	if (error)
		goto err_unregister_handle;

	error = device_add(&evdev->dev);
	if (error)
		goto err_cleanup_evdev;

	return 0;

 err_cleanup_evdev:
	evdev_cleanup(evdev);
 err_unregister_handle:
	input_unregister_handle(&evdev->handle);
 err_free_evdev:
	put_device(&evdev->dev);
	return error;
}
</code></pre>
> 至此，我们的部长evdev就诞生了。这里面的细节非常的多，现在我们一一道来。
> 
> * 像市里面一样，EV部门也有一个大箱子叫`evdev_table[]`， 为什么要这个箱子呢？我们之前说过这个部门的部长是根据具体情况
> 临时招聘的，所以这个部门在有些时候会出现不止一位部长。这时候就出现一个问题，如果外面的百姓像找这个部门的一位部长办事，
> 他得指明是哪位部长，所以每个部长会把自己的信息扔进这个`evdev_table[]`箱子。当然是要按位置摆放好，记录这个位置的就是`minor`.
<pre><code>
for (minor = 0; minor < EVDEV_MINORS; minor++)
		if (!evdev_table[minor])
			break;
</code></pre>
> 上面这段代码很简单，就是部长在大箱子里面找空的位置，可能在他之前还有其他部长过来就职过。
<pre><code>
evdev->minor = minor;
</code></pre>
> 部长自己也记下了自己的位置号，我们可以把他看成这位部长在这个部门的工作编号。
<pre><code>
error = evdev_install_chrdev(evdev);

static int evdev_install_chrdev(struct evdev *evdev)
{
	/*
	 * No need to do any locking here as calls to connect and
	 * disconnect are serialized by the input core
	 */
	evdev_table[evdev->minor] = evdev;
	return 0;
}
</code></pre>
> 上面部长将自己信息放到箱子里，至此，这个箱子中的`minor`位置就是这位部长的了。
> 下面我们将看到，来访者是怎样通过这个`minor`编号找到部长的，其实也就是用户程序通过`open`系统调用是如何访问到用户想要的资源信息。
> 
> * 好了，让我们来看看重点，新官上任，这位部长有权自己招聘一个自己的小秘，这个小秘叫`handle`, 请记好她的名字，因为后面很多事情都需要她出面解决。
<pre><code>
evdev->handle.handler = handler;
evdev->handle.private = evdev;
</code></pre>
> 上面就是小秘需要保管着部长的简历`handler`, 同时记着部长的一切信息。
<pre><code>
error = input_register_handle(&evdev->handle);
</code></pre>
> 上面这句话的作用是部长得在他个人简历里面写明他招聘来的小秘。让我们回顾一下当时部长写的简历，也就是下面这段代码：
<pre><code>static struct input_handler evdev_handler = {
	.event		= evdev_event,
	.connnect 	= evdev_connect,
	.disconnect = evdev_disconnect,
	.fops		= &evdev_fops,
	.minor		= EVDEV_MINOR_BASE,
	.name		= "evdev",
	.id_table	= evdev_ids,
};

int input_register_handle(struct input_handle *handle)
{
	struct input_handler *handler = handle->handler;
	struct input_dev *dev = handle->dev;
	int error;

	/*
	 * We take dev->mutex here to prevent race with
	 * input_release_device().
	 */
	error = mutex_lock_interruptible(&dev->mutex);
	if (error)
		return error;
	list_add_tail_rcu(&handle->d_node, &dev->h_list);
	mutex_unlock(&dev->mutex);

	/*
	 * Since we are supposed to be called from ->connect()
	 * which is mutually exclusive with ->disconnect()
	 * we can't be racing with input_unregister_handle()
	 * and so separate lock is not needed here.
	 */
	list_add_tail(&handle->h_node, &handler->h_list);

	if (handler->start)
		handler->start(handle);

	return 0;
}

</code></pre>
> 填简历时部长的`h_list`还是空着的，因为那是他还没有被任用，自然没有权利招聘小秘。
> `list_add_tail_rcu(&handle->d_node, &dev->h_list);`这句话说明，小秘不仅要记录在部长简历里，同时要备案到市里面去。
> 
> * 好了，借助于`evdev.c`这个“EV部门”实例，我们现在差不多对整个`input`子系统已经有了一个比较形象的认识，之所以要形象化的剖析不是闲着没事做，而是为了避免其他技术博客分析的那样，上来就是几个复杂的结构体，再加上一通代码，初学者一看就晕了。
> 需要注意的是，`evdev.c`本身不是`input`子系统的一部分，他只是一个使用`input`子系统的实例，下面会介绍与比较其他实例来更好的理解这个系统。
> 下面就要结合应用程序深入剖析这个子系统。

---

##长刀直入子系统
> * 让我们借助于`../drivers/input/keyboard/gpio_keys.c`这个按键驱动实例来深入分析。
>  
>
> * 这个部门还有一个职员叫`evdev_clint`，他的职责是外勤，就是要与“Linux VFS”对接业务。这个职员也只是在这个部门有任务时才招聘的。
> 


---

##扩展，由`input`子系统反观`platform`驱动架构


---





