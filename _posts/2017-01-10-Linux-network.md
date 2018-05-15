---
layout: post
title:  "Linux 网络笔记"
date:   2017-01-10 15:15:54
categories: Linux-Application
excerpt: linux
---

* content
{:toc}

记录开发中实际涉及的Linux网络知识

---

## ARP:地址解析协议

<br />

### 基础知识

<br />

地址解析协议(ARP)用于实现IP地址到网络接口硬件地址的映射，以太网(ether)是这里所说的网络中的一种。以太网48位的硬件地址就是我们常说的MAC地址。

在以太网中，当某个主机要向以太网中的另一台主机发送IP数据时，它首先根据目的主机的IP地址在ARP缓存(*一个IP地址到相应以太网地址的映射表*)中查询相应的以太网地址(MAC)。如果查询到匹配的结点，则相应的以太网地址被写入以太网帧首部，数据报被加入到输出队列等待发送；如果查询失败，ARP会先保留待发送的IP数据报，然后广播一个询问目的主机硬件地址的ARP报文，等收到回答后再将IP数据报发送出去。


<br />

### 运用

<br />

**获取Gateway的MAC地址**

<br />

执行`arp -a`获取当前的路由缓存信息，因为其中必定包含有Gateway的信息，所以可以通过这个命令获取Gateway的MAC地址。






---

## netstat

<br />

今天在运行ffserver时出现`Address already in use`的错误提示，使用netstat工具去查看是哪个进程占用了端口号是一个不错的选择。现在记录一下netstat的一些用法。

-a	|列出当前所有域的连接，包含TCP/IP域和UNIX域
-tu	|只显示TCP/IP的连接信息
-l	|只列出正在监听(LISTEN)状态的套接字
-p	|查看连接所对应的进程名、进程号
-e	|查看连接所对应的用户名
-s	|列出所有网络包的统计情况
-r	|显示内核路由信息
-ie	|输出结果和ifconfig一致
-g	|输出IPV4/V6的多播组信息
-n	|禁用反向域名解析，加快查询速度

下面是使用带`n`和不带的对比:

![netstat_n](http://omp8s6jms.bkt.clouddn.com/netstat1.png)

![netstat](http://omp8s6jms.bkt.clouddn.com/netstat2.png)

---

## 网络IPC:套接字

SOCK_SEQPACKET 套接字类型和SOCK_RAW套接字类型很类似，但是该套接字得到的是基于报文的服务而不是字节流服务;SCTP(stream control transmission protocal)－－流控制传输协议提供的就是因特网域上的顺序数据包服务。

SOCK_RAW套接字提供一个数据报接口用于直接访问下面的网络层(在因特网域中就是IP)。
使用这个接口时，应用程序负责构造自己的协议首部，这是因为传输协议(TCP、UDP)被绕开了。当创建一个原始套接字时需要有超级用户特权，用以防止恶意程序绕过安全机制。


---

##	tcpdump

tcpdump根据使用者的定义对网络上的数据包进行截获分析，它可以将网络中传送的数据包头完整截获下来提供分析。它支持针对网络层，协议，主机，网络或者端口的过滤。

> ### Options

It's important to note that `tcpdump` only takes the first 96 bytes of data from a packet by default.


-n	|names are not resolved,resulting in the IPs themselves always being displayed
-nn	|don't resolved hostnames or port names
-X	|displays both hex and ascii content within the packet
-S	|changes the display of sequence numbers to absolute rather than relative
-s	|change the number of bytes you want to capture,and 0 means gets everything
-i eth0	|listen on the eth0 interface
-D	|show the list of available interfaces
-v,-vv,-vvv	|increase the amount of packet informaton you get back
-tttt	|give maximally human-readable timestamp output
-c	|only get x number of packets and then stop
icmp	|only get ICMP packets

有三个主要的types of expression: type,dir,and proto.

> * Type options 主要包含:host,net,and port.
> * Direction lets you do src,dst,and combinations thereof.
> * Protocol 主要让你限定数据包所属的协议类型，必然tcp,udp,icmp,ah等等.


> ### Basic Usage

如果想得到和wireshark软件中默认的抓包显示效果，可以执行:

`#tcpdump -nnvS -s 0`

如果比较关心数据包的时间值，可以添加`-tttt`,即执行:

`#tcpdump -ttttnnvS -s 0`


上面的`-s 0`表示不对抓取的数据量做限制，运行实际效果如下图。

![nnvs](http://omp8s6jms.bkt.clouddn.com/image/git/verbase.PNG)

如果想查看所抓包里面的具体二进制内容，可以添加`-X`选项，同时为了控制所抓包的数量，添加`-c`选项。下面就是从送往ip:172.8.1.98的数据包中抓取2个数据包进行内容查看。

![X_dst](http://omp8s6jms.bkt.clouddn.com/image/git/tcp_c2.PNG)


> ### Writing to a File

其实通常的情况是，我们在设备上通过tcpdump工具抓包保存到文件中，然后通过图形化的软件wireshark来分析数据。tcpdump保存的文件格式类型是PCAP.执行命令中添加`-w`选项即可。例如:
<pre><code>#tcpdump port 80 -w cap_file
</code></pre>


> ### Advanced 

组合起各种选项设置可以实现非常强大的功能。实现组合的三种方式为:

1. AND 	-- and or &&

2. OR  	-- or  or ｜｜

3. EXCEPT -- not or !


指定源ip和目的端口

`#tcpdump -nnvvS src 172.8.1.98 and dst port 37777`

查看所有192.168.x.x网段内到10.x网段或者到172.16.x.x网段的数据包

`#tcpdump -nvX src net 192.168.0.0/16 and dst net 10.0.0.0/8 or 172.16.0.0/16`

过滤掉发往某个ip的icmp数据包

`#tcpdump dst 192.168.0.2 and src net and not icmp`

当我们使用更加复杂的过滤选项时，有时我们需要使用单引号来规避特殊符号的干扰

例如`#tcpdump src 10.0.2.4 and (dst port 3389 or 22)`这样的写法是错误的，应该是`#tcpdump 'src 10.0.2.4 and (dst port 3389 or 22)'`


![tcp-header](https://danielmiessler.com/images/tcp_header.png)

> 下面借助上图介绍如何通过TCP FLAGS来进行过滤。

过滤出所有URG packets

`#tcpdump 'tcp[13]&32 !=0'`

过滤出所有ACK packets

`#tcpdump 'tcp[13]&16 != 0'`

过滤出所有RST packets

`#tcpdump 'tcp[13]&8 != 0'`

过滤出所有SYNACK(SYNCHRONIZE/ACKNOWLEDGE) packets

`#tcpdump 'tcp[13]=18'`

> 下面介绍几个需要死记但是非常实用的过滤条件

packets with both the RST and SYN flags set

`#tcpdump 'tcp[13]=6'`

find cleartext http GET requests

`#tcpdump 'tcp[32:4]=0x47455420'`

find SSH connections on any port(via banner text)

`#tcpdump 'tcp[(tcp[12]>>2):4] = 0x5353482d'`

find packets with a TTL less than 10(usually indicates a problem or use of TRACEROUTE)

`#tcpdump 'ip[8]<10'`



---


## wireshark

### 设置数据抓取选项

点击常用按钮中的设置按钮，就会弹出如下图所示的选项对话框:

![catch-set](http://www.bo56.com/wp-content/uploads/2014/11/wireshark_catch_set.png)


这里面主要有两个设置，一个是设置混杂模式(promiscuous mode)，使能该模式就可以捕获局域网内所有的数据包，否则你只能捕获发往你主机或者从你主机出去的数据包。

另一个设置项是捕获过滤条件(capture filter),注意它不同于后面将要提到的显示过滤条件。如果我们只想捕获http相关的数据包，我们就可以设置捕获过滤条件为"port 80"

### 使用显示过滤器

在Filter对话框里面可以直接填写过滤条件，常见的如指定只想查看某个ip过来的所有数据包，如下图所示:

![filter](http://omp8s6jms.bkt.clouddn.com/image/git/filter_ip.PNG)


我们同时限定了源ip和目的ip。另外，当我们想将我们过滤好的数据包生成新的抓包文件时，如下图所示，点击`Export special pakcts as...`，进行保存即可。

![save_spe](http://omp8s6jms.bkt.clouddn.com/image/git/save_s.PNG)




点击显示表达式按钮(Expression),对应对话框如下:

![expression](http://www.bo56.com/wp-content/uploads/2014/11/wireshark_expression.jpg)


对话框中分为协议域(field name)，关系(relation)，条件值(value),预定义值(predefined value)。
如上图，我们创建的表达式的作用是，只显示http协议包中包含关键字"bo56.com"的所有数据包。

**Field name说明**

这个列表展示了所有支持的协议。点击前面的三角形标志后，可以列出本协议的可过滤字段。
直接找需要的协议可能比较麻烦，可以输入协议中开头字符进行自动模糊查询自动跳转。

**Relation说明**

is present	|如果选择的协议域存在，则显示相关数据包
contains	|判断一个协议，字段或者分片包含一个值
matchs	|判断一个协议或者字符串匹配一个给定的perl表达式


**Predefined values说明**

有些协议域包含了预定义的值，有点类似c语言中的枚举类型，如果你选择的协议域包含这样的值，你可以在这个列表中选择。

**Function函数说明**

过滤器的语言还有下面几个函数:

upper(string-field) - 把字符串转换为大写
lower(string-field) - 把字符串转换为小写

例如:

upper(ncp.nds_stream_name) contains "BO56.COM"
lower(mount.dump.hostnmae) == "BO56.COM"

### 使用着色规则

点击"view"菜单，然后选择"coloring rules"选项就会弹出设置颜色规则设置对话框，如下图:

![color-rules](http://www.bo56.com/wp-content/uploads/2014/11/wireshark_color_rules.jpg)


对话框已经包含了很多已经存在的规则，如果要新建则点击"New",弹出对话框如下:


![color-set](http://www.bo56.com/wp-content/uploads/2014/11/wireshark_color_set.png)


name字段中填写规则名称，方便记忆。

string字段填写过滤规则，这里的语法和显示规则表达式一致。

注意:wireshark在应用规则的时候，是自上而下的顺序去应用规则，因此刚添加的规则会优先应用。如果想调整顺序，可以选中规则，然后点击右边的"UP"和"Down"按钮。


### 跟踪tcp流

这个功能是将接收到的数据排好顺序使之容易查看，而不需要一小块一小块地查看。这在查看http，ftp等纯文本应用层协议时非常有用的。

使用方法就是对所获取数据包感兴趣的条目中执行右击，然后选择跟踪跟踪流(Follow TCP Stream)。具体可以参照以下两张图:


![tcp-follow](http://www.bo56.com/wp-content/uploads/2014/11/wireshark_tcp_follow.png)



![tcp-follow-dialog](http://www.bo56.com/wp-content/uploads/2014/11/wireshark_tcp_follow_dialog.png)



上图中底部有个下拉框，默认是如图的`Entire conversation`，即上面的stream content是tcp两端对话的往返流，有时我们只需要分析单向的流数据，如下图，这时可以点击下拉框，选择我们想指定的一个数据流向。

![stream](http://omp8s6jms.bkt.clouddn.com/image/git/direction.PNG)




---

