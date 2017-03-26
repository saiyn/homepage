---
layout: post
title:  "技术杂记"
date:   2017-02-19 15:15:54
categories: other
excerpt: unix linux note
---

* content
{:toc}


![image](http://coolshell.cn//wp-content/uploads/2012/05/Banni%C3%A8re-Unix-linux.jpg)

记下阅读一些技术书籍时的知识，或者是书中某个点引发的思想灵感。



---

## 多进程还是多线程

从入职接触到我司项目代码开始，一直有个疑问，对于这么复杂庞大的音视频项目软件，是否需要使用多进程，还是说使用多线程是最佳解决方案。

最近看的《The arr of unix prigramming》书中非常明确肯定的支出，应该使用多进程而不是多线程，多线程只有在万不得已时才使用。但是纵观目前音视频领域的各个开源软件，情况好像有所不同。

VLC is heavily multi-thread.

The single-thread approach would have introduced too much complexity because decoder preemptiblity and scheduling would then be a mastermind(for instance decoders and outputs have to be separated, otherwise it cannot be warrantied that a frame will be played at the exact presentation time).

Multi-process approach wasn't chosen either, because multi-process decoders usually imply more overhead(problems of shared memory) and communication between processes is harder.

在我看来，如果使用多进程的话，首先得为多进程之间的通信制定良好的通信协议。而且，音视频领域的软件需要交互大量的音视频数据，使用进程间的通信进行大量数据传输可能带来非常大的开销。而多线程天生的优势就是隐式的自动的提供无任何消耗的共享内存通信机制。




## Form 《The art of unix programming》

MacOs 在历史上发生过３次重大设计变革，其中第三次是把自己和来自Unix的架构进行融合形成了现在的MacOs X。

MacOs主要设计理念就是:界面方针(the Mac Interface Guidelines)，也就是说Mac系统是一个`客户端操作系统`,而与之对应的unix类操作系统是
`服务器操作系统`。

为什么Linux系统上面的GUI一直无法达到Mac的水平？因为将服务器操作系统特性，如多用户优先权和完全多任务处理，改装到客户端操作系统上非常困难，
很可能打破对旧版本客户端的兼容性，而且通常做出的系统即脆弱又令人不满意。

但是现在非常火爆的Andrid系统使用了另外的方向，就是将GUI应用于服务器操作系统。这样会出现的问题大部分可通过灵活处理和投入更廉价硬件资源得到解决。

从上面也可以看出为什么现在的苹果手机在硬件配置远低于安卓手机的情况下还能够表现得比安卓手机流畅很多。


---

