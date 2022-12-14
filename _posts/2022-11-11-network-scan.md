---
layout: post
title:  "Network Scan"
date:   2022-11-11 15:15:54
categories: IOT-Security Linux-Network
excerpt: iot nmap linux network scan
---

* content
{:toc}


# Technical Detail

<br />

## TCP端口探测

tcp端口扫描的方法有多种，nmap支持SYN/Connect()/ACK/Window/Maimon这几种，每一种方法都存在非常tricky的技术细节。
<br />

### connect

通过调用系统调用connect()和目标机器的tcp端口进行连接是准确性很高的方法，但是显然易见的就是其耗时会比较长，不过通过设置socket的某些option可以提高性能。

* SO_LINGER

```
    struct linger {
        int l_onoff; //0=off, nonzero=on(开关)
        int l_linger; //linger time(延迟时间)
    }
```

l_onoff |l_linger   |close行为  |发送队列   |底层行为
--- |---    |---    |---    |---
0   |=  |立即返回   |保持直到发送完成   |系统接管套接字并保证将数据发送到peer
非0  |0 |立即返回   |立即放弃   |直接发送RST包，不用等待2MSL，立即复位
非0 |非0    |阻塞直到l_linger时间超时或者数据发送完成   |在超时时间内保持尝试发送   |超时后行为同第二种情况


根据TCP的四次挥手状态转换我们知道，调用close()系统调用发起断连的一方会进入TIME_WAIT状态，需要等待2MSL时间才能复位进行下一次连接。这样在进行TCP connect扫描端口时会出现大量TIME_WAIT并且浪费时间在完整的4次挥手。

通过设置`l_onoff=1;l_linger=0`使得调用close后直接发送RST实现快速复位，进行下一个扫描。


* SO_ERROR

在调用完connect后，通过`getsockopt(sd, SOL_SOCKET, SO_ERROR, (char *) &optval,&optlen)`获取连接的状态，根据返回状态码去判断host端口的状态。





### SYN

要实现SYN扫描，必须得有创建raw socket的权限，正常情况下SYN的发送都是系统API的内部行为，比如connect()系统调用会触发三次握手中的SYN包。

SYN扫描逻辑是，主动发送SYN报文，通过观察peer回复ACK的情况来判断端口是否开启。

* 如果收到了对方的SYN+ACK，说明对方端口肯定是开启的。
* 如果收到了对方的RST，说明对方端口是关闭的。
* 如果没有收到任何回应或者是收到了ICMP错误报文，则无法判断准确的状态。

SYN扫描的最大好处就是速度很快，首先扫描只需要发送一个SYN包，而且因为我们是通过raw socket创建发送的SYN包，没有经过TCP/IP协议栈，所以内核完全是不知道连接状态的，这样的好处就是，当peer回复ACK时，内核层会直接发送RST进行回应，自动清理了这次扫描尝试的连接状态。也是因为这一点决定了我们在处理peer的rsp报文时必须得使用pcap技术，比如libpcap等等。

**SYN cookies**

SYN扫描只是nmap端口探测的一种可选项，但是却是zmap和masscan(都号称几分钟扫描完整个公网)的主要或者说是唯一方式。为什么zmap和masscan也是使用SYN扫描，速度却是nmap的很多倍呢，因为他们都使用了(SYN cookies)[http://cr.yp.to/syncookies.html]技术实现了无状态扫描，不同与nmap需要维护每一个SYN probe的状态以此来判断端口是否开启。

SYN cookies技术就是借助与tcp报文头部的seq字段是可以自定义的这一点，通过将关键信息编码进seq字段，然后在收到SYN+ACK报文中的acknowledgment number字段减去1后进行解码，得到关键数据，从而判断是哪个peer发送的ack。

![network_scan_0.png](https://raw.githubusercontent.com/saiyn/homepage/gh-pages/images/network_scan_0.png)

上图是zmap编码后设置tcp header中seq的代码。


```
    uint32_t src_ip = get_src_ip(current_ip, i);
    uint32_t validation[VALIDATE_BYTES /
                sizeof(uint32_t)];
    validate_gen(src_ip, current_ip,
                (uint8_t *)validation);
    uint8_t ttl = zconf.probe_ttl;
    size_t length = 0;
    zconf.probe_module->make_packet(
        buf, &length, src_ip, current_ip, ttl,
        validation, i, probe_data);
```                    

从调用make_packet()接口的代码段可以看到，zmap是将当前源ip地址(src_ip)和目标ip地址(current_ip)编码进了header。



除了运用与网络扫描，SYN cookies也可以一定程度上抵挡SYN FLOOD类型的DDoS攻击。比如通过`echo 1 > /proc/sys/net/ipv4/tcp_syncookies`可以使能linux内核中的syn cookies功能，这样服务器在接受到SYN报文时，如果SYN报文的接受数量超过了backlog设定值，那么对于后续的SYN会启动cookies机制，就是发送出去的syn+ack是加入编码数据的，同时不会再为半连接状态分配资源，直到收到对应的ack报文才会分配资源加入到accept队列中。

<br />

### SYN SCAN VS connect SCAN

type    |权限   |发送探测   |接受回应   |实现难易   |准确性 |扫描性能
--- |---    |---    |---    |---    |---    |---
SYN |root   |SOCK_RAW,IPPROTO_RAW，sethdrinclude发送自建的ip层到tcp层的数据包   |通过pcap在数据链路层拿数据，虽然也可以SOCK_RAW,IPPROTO_TCP监听所有tcp返回包    |困难   |一般   |很快
connect |user   |SOCK_STREAM,IPPROTO_TCP, connect() |select(), getsockopt(, SOL_SOCKET, SO_ERROR,)  |简单   |较高   |较慢

<br />


### ACK SCAN

ACK scan主要用来探测目标端口是否处于fireware屏蔽状态，因为之前所描述的SYN和connect扫描，如果遇到目标端口处于fireware保护状态的话，是无法进行探测的。

根据RFC 793规范，如果一个host收到的第一个package是ACK，那么无论该TCP端口是否开启，Host都会发送RST回应。
而如果该host的端口处于fireware屏蔽状态，那么host就会不回应或者只是发送一个ICMP error进行回应。

ACK SCAN的判断逻辑就是基于上面2点。

相比于前面的SYN和connect扫描，ACK SCAN是比较难以被监控到和block的，如果要检测到ACK SCAN行为，防火墙必须记录每个连接的状态， 比如通过iptables设置如下规则:

```
    iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```

设置上面规则后，系统只接受已有连接上的数据包或者相关联的数据包，而对于ACK SCAN中发送的`Unsolicited ACK`则会直接丢弃掉不做任何回应。

<br />


### FIN,NULL,Xmas SCAN

根据RFC 793第65页的内容，对于处于close状态的端口，如果接收到了任何数据包中不含有SYN、RST、ACK中所有的flag，那么host应该回应RST，如果端口是open的，则不进行任何回应。

基于上面的规范，可以构造设置剩余FIN、PSH、URG 3个flag的任意组合的探测包。

