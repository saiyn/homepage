---
layout: post
title:  "ARM开发板 NFS 网络设置"
date:   2015-10-20 11:48:54
categories: Linux
excerpt: 虚拟机:Vmware，系统：Ubuntu14.04。
---

* content
{:toc}

本文主要记录如何搭建一个嵌入式Linux开发环境，当然开发环境的搭建有好几种方法，本文只介绍基于NFS和Samba服务的一种环境搭建方式。

---

##软、硬件环境
主机：WIN XP
虚拟机：VmWare Workstation  系统：Ubuntu14.04。
开发板：MINI2440

---

##VmWare的网络配置
> 搭建开发板的NFS网络环境，最主要是搞清楚VmWare的网络如何配置。
> 点击VMwware的菜单栏“虚拟机(M)”->设置->网络适配器，
> 我们可以发现VWware主要支持3中网络模式：桥接(Bridge)模式、NAT模式(网络地址转换模式)以及仅主机模式(Host-only).
> 为了让我们的开发板、主机和虚拟机3者能够互相通信，我们得让他们处于同一个局域网即可。因此，我们应该选择桥接(Bridge)模式。


##Ubuntu14.04的网络配置
> 这里主要介绍2个常用配置方法：通过配置文件配置、通过命令行配置。
> * 主要文件：`/etc/network/interfaces`, 这里是IP、网关、掩码等的一些配置; `/etc/resolv.conf`这个保存DNS有关信息。
> * 主要命令：`sudo /etc/init.d/networking restart`在14.04版本中已经不起作用了，会出现如下问题：
<pre><code># tail -f /var/log/upstart/networking.log
Stopping or restarting the networking job is not supported.
Use ifdown & ifup to reconfigure desired interface.
</code></pre>
> 从以上日志内容可以看出，传统的service 重启和停止网络已经不再支持了，需要通过ifdown 和 ifup 来实现相应操作。
> * 重启指定网卡
<pre><code># ifdown eth0 && ifup eth0
</code></pre>
> * 重启除lo网卡的所有网卡
<pre><code># ifdown --exclude=lo -a && sudo if up --exclude=lo -a
</code></pre>

##NFS挂载
<pre><code># mount -t nfs -o nolock 192.168.1.245:/home/saiyn/nfs /home/nfs
</code></pre>





