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

##	tcpdump

tcpdump根据使用者的定义对网络上的数据包进行截获分析，它可以将网络中传送的数据包头完整截获下来提供分析。它支持针对网络层，协议，主机，网络或者端口的过滤。

### Options

It's important to note that `tcpdump` only takes the first 96 bytes of data from a packet by default.


-n	|names are not resolved,resulting in the IPs themselves always being displayed
-nn	|don't resolved hostnames or port names
-X	|displays both hex and ascii content within the packet
-S	|changes the display of sequence numbers to absolute rather than relative
-s	|change the number of bytes you want to capture,and 0 means gets everything
-i eth0	|listen on the eth0 interface
-D	|show the list of available interfaces
-v,-vv,-vvv	|increase the amount of packet informaton you get back
-c	|only get x number of packets and then stop
icmp	|only get ICMP packets


### Basic Usage




### Examples

### Writing to a File

### Getting Creative

### Advanced 





![tcp-header](https://danielmiessler.com/images/tcp_header.png)



---


## wireshark

### 设置数据抓取选项

点击常用按钮中的设置按钮，就会弹出如下图所示的选项对话框:

![catch-set](http://www.bo56.com/wp-content/uploads/2014/11/wireshark_catch_set.png)

这里面主要有两个设置，一个是设置混杂模式(promiscuous mode)，使能该模式就可以捕获局域网内所有的数据包，否则你只能捕获发往你主机或者从你主机出去的数据包。

另一个设置项是捕获过滤条件(capture filter),注意它不同于后面将要提到的显示过滤条件。如果我们只想捕获http相关的数据包，我们就可以设置捕获过滤条件为"port 80"

### 使用显示过滤器

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










---

