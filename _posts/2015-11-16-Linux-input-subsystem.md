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
> 这时可以调用`input_register_handler(&evdev_handler);`去向市里面申请一个部长。
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
> 从`handler->minor >> 5`这个处理细节来看，我们的书记可不是随便放他的简历的，其中意义我们后面再细细道来。
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
> 好，现在我们重点看看这位部长是如何组建自己的部门的：
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

> 这个部门有一个大箱子叫`evdev_table[]`，这个大箱子放的是什么，干什么用的，东西怎么放进去的，如果拿出里面的东西进行使用，这个我们先不管，以后用到时再说。
> 这个部门还有一个职员叫`evdev_clint`，他的职责是外勤，
> 



---





