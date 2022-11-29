---
layout: post
title:  "Network Scan"
date:   2022-11-11 15:15:54
categories: IOT Security
excerpt: iot nmap linux network scan
---

* content
{:toc}


# Technical Detail

## TCP端口探测

tcp端口扫描的方法有多种，nmap支持SYN/Connect()/ACK/Window/Maimon这几种，每一种方法都存在非常tricky的技术细节。

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


