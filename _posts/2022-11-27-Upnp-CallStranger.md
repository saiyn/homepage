---
layout: post
title:  "Upnp CallStranger"
date:   2022-11-27 15:15:54
categories: IOT-Security Linux-Network
excerpt: network nmap shodan
---

* content
{:toc}


# 漏洞分析

UPnP协议规范中有一个重要的功能模块，叫做事件(Eventing)。在UPnP服务进行的时间内，只要设备用于UPnP服务的变量值发生变化或者模式发生了改变，就会向239.255.255.250这个地址组进行广播。另外用户也可以事先向UPnP设备发送订阅请求，保证UPnP设备及时地将事件进行单播定向传送给自己。



![upnp_call_0](https://raw.githubusercontent.com/saiyn/homepage/gh-pages/images/upnp_call_0.png)


如上图所示，我们先通过向239.255.255.250组播地址发送M-Search报文，探测当前网络下有哪些开启UPnP服务的设备，然后通过获取1900端口的xml服务描述文件拿到该设备的第一个可以订阅的`eventSubURL`。

然后我们来看一下UPnP DeviceArchitecture 2.0中关于UPnP的NT与CALLBACK订阅模块的格式：

![upnp_call_1](https://raw.githubusercontent.com/saiyn/homepage/gh-pages/images/upnp_call_1.png)


UPnP协议规范中提到， CALLBACK是必填字段，所填信息为发送事件信息的URL，一般情况下由设备供应商提供。另外，`如果其中定义了不止一个URL，设备会按顺序尝试连接，直到有一个连接成功`。

# 漏洞危害

CallStranger漏洞所造成的危害可以分为三个方面：DDos攻击、数据逃逸和端口扫描。其中造成的DDos攻击可以分为两种，SYN洪水攻击和TCP反射发大攻击。

## SYN洪水攻击

如前面协议规范中提到的，若CALLBACK value字段中定义了不止一个URL，则会按顺序尝试TCP连接，直到有一个连接成功。那么攻击者可以在CALLBACK value中精心构造多个URL，是每一个都无法连接成功，这样UPnP设备就会使用多个SUN包依次对每个URL尝试TCP握手。假设攻击者可以操控很多设备，就会导致受害设备遭遇DDoS攻击。

## TCP反射放大攻击

这个放大效果一般与Upnp设备的操作系统和厂商配置有关。

## 数据逃逸

一般情况下，企业内部网络都有不同安全等级的划分，当攻击者渗透到企业内网时，若内网开启了泄漏防护系统，无法将获得的敏感数据传输出去，此时upnp设备会是一个很好的跳板。
在RFC7230的3.1.1节中，并没有对Request Line的长度做任何限制，者使得攻击者可以将数据通过callback的URL值传输出去。


## 端口扫描

还是利用CALLBACK可以定义不止一个URL，并且会按顺序尝试连接这个特性，假设攻击者想扫描IP为192.168.1.13的8080端口是否开启，那么攻击者只需要将某个可以监控的URL放置在其后即可，如果攻击者监控到前面的URL有请求，那么就说明放置在这个URL前面的探测8080端口的URL没有连接成功，反之则可以判断8080端口是开启的。



![upnp_call_2](https://raw.githubusercontent.com/saiyn/homepage/gh-pages/images/upnp_call_2.png)


上图就是构造的一个有效的探测攻击报文。







# 漏洞缓解与修复

在最新的upnp协议规范中，可以看出订阅事件的源ip和目标ip都被限制为内网IP。