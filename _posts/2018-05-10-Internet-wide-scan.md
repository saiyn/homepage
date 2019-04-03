---
layout: post
title:  "Internet-wide scan"
date:   2018-05-10 15:15:54
categories: network
excerpt: Internet network zmap Internet-wide scan
---

* content
{:toc}


---

# zmap

<br />

## Usage

<br />

Do upnp protocol scan - `sudo zmap -p 1900 --probe-module="upnp" -O json -r 10000`

![zmap_0](http://omp8s6jms.bkt.clouddn.com/image/git/zmap_0.png)

<br />

We need more specific information, so try to figure out what's else zmap can offer us - `zmap --probe-module="upnp -O json --list-output-fields" `

![zmap_1](http://omp8s6jms.bkt.clouddn.com/image/git/zmap_1.png)

<br /> /

use -f options to add what we are interested - `sudo zmap -p 1900 --probe-module="upnp" -O json -r 10000 -f "saddr, classifiction, location" `

![zmap_2](http://omp8s6jms.bkt.clouddn.com/image/git/zmap_2.png)

<br />

## zmap实现原理

<br />

### ip地址扫描

<br />

在ip地址扫描策略上，zmap进行了特别的优化，主要在三个方面:

1. 相对于nmap逐个按顺序的扫描ip地址，zmap进行散列式的分组随机扫描，以保障目标地址网段不会出现饱和拥塞。

2. nmap在扫描时需要维护每次扫描时建立的连接状态，以避免重复扫描，而zmap通过使用`cyclie multiplictive group`的算法，使用三个整形变量实现随机无重复的覆盖扫描。

3. 探测优化，使用raw socket编程，重载每个发送包中没有使用到的字段来区分接收到的包，实现无状态的不重复的扫描，并且每个目标地址都只发送固定数量的探测包（默认是1个）。

<br />

**ip散列算法**

zmap中ip散列分布扫描的算法基于[Primitive root modulo n](https://en.wikipedia.org/wiki/Primitive_root_modulo_n)数学理论,关于这个理论本身的定义
和证明我们暂不关心，我们关心其特性的运用价值。

IPv4的地址空间范围是0~2^32，为了实例简单易懂，我们假设我们需要扫描的范围是(0,7)，那么n就是7,7的primitive root有3和5，下面的表说明为什么3是7的
本源根:


![zmap_5](https://raw.githubusercontent.com/saiyn/homepage/gh-pages/images/zmap_5.png)

<br />

和3类似，5也可以通过上面计算验证为是7的本源根，下面通过一张示意图来说明如何本源根的特性:

![zmap_6](https://raw.githubusercontent.com/saiyn/homepage/gh-pages/images/zmap_6.png)


<br />

结合上图，假设我们第一个扫描的ip地址整数值是3，那么通过`3 * 5 mod 7 = 1`我们得到下一个我们应该扫描的ip地址是1,然后再通过`1 * 5 mod 7 = 5`得到下一个ip地址是5,因此类推，我们就随机的扫描了所有的ip地址。

zmap中n值是(2^32 + 15 = 4,294,967,311)，即比2^32大的最小质数，这个质数的本源根恰好也是3。这样zmap通过三个整形变量(本源根、第一个IP地址、当前的IP地址)就可以维持ip地址扫描的状态。但扫描到的下一个IP地址和第一个IP地址相等时，说明我们已经扫描完整个ipv4地址了。



<br />

**无状态跟踪实现代码**

我们知道，正常的TCP通信都需要三次握手，在此期间需要维护记录与对方交互的状态，这种状态记录开销是很大的，所以zmap不进行三次握手，只发送一个SYN，然后
等待对方回复SYN-ACK,之后就发送RST取消连接。虽然nmap中也支持这种扫描，但关键性的问题是对回复的SYN-ACK进行seq number的校验。正常的做法是需要记录状态，而zmap是将目标ip地址进行hash，将其保存到探测包中未使用的字段中，这样当SYN-ACK回来时，就可以根据其携带的hash信息来进行校验，免去了状态存储。
具体代码实现如下:


	//send.c
	
	/* one sender thread */
	int send_run(sock_t st, shard_t *s)
	{
		...
		
		/* actually send a packet */
		for(int i = 0; i < zconf.packet_streams; i++)
		{
			size_t length = zconf.probe_module->packet_lenght;
			uint32_t validation[VALIDATE_BYTES / sizeof(uint32_t)];
			...
			
			validate_gen(src_ip, current_ip, (uint8_t *)validation); 
			
			zconf.probe_module->make_packet(buf, &length, src_ip, current_ip, validation, i, probe_data);
			
			...
		
		}
		
	}

上面代码是在实际发送一个packet时的组包处理，validate_gen()函数将源ip地址src_ip和目的ip地址current_ip进行哈希，哈希得到的结果存入validation变量
传入probe module中实际组包的回调函数，make_packet回调函数会加哈希结果写到可以重载的字段。不同的probe module发送的包处于不同的协议层，因此重载的字段不一样，下面以icmp echo这个probe module为例：

	//

	static int icmp_echo_make_packet(void *buf, size_t *buf_len, ipaddr_n_t src_ip, ipaddr_n_t dst_ip, uint32_t *validation, ...)
	{
		struct ether_header *ether_header = (struct ether_header *)buf;
		struct ip *ip_header = (struct ip*)(&ether_headr[1]);
		struct icmp *icmp_header = (struct icmp *)(&ip_headr[1]);
		
		...
		
		icmp_header->icmp_id = validation[1] & 0xffff;
		icmp_header->icmp_seq = validation[2] & 0xffff;
		
		...
		
	}
	
icmp报文的格式如下:



![zmap_3](https://raw.githubusercontent.com/saiyn/homepage/gh-pages/images/zmap_3.png)

<br />

具体到icmp echo报文，格式如下:


![zmap_4](https://raw.githubusercontent.com/saiyn/homepage/gh-pages/images/zmap_4.png)

<br />

通过报文格式和前面的ping和traceroute程序的实现，可见，icmp_id和icmp_seq字段是可以重载而不会影响通信的。


下面再来看看tcp syncscan的probe module的make packet函数实现:
	//module_tcp_synscan.c
	
	static int synscan_make_packet(*buf, size_t *buf_len, ...)
	{
		struct ether_header *ether_header = (struct ether_header *)buf;
		struct ip *ip_header = (struct ip*)(&ether_headr[1]);
		struct tcphdr *tcp_header = (struct tcphdr *)(ip_header[1]);
		
		...
		
		tcp_header->th_sport = htons(get_src_port(num_ports, probe_num, validation));
		tcp_header->th_seq = validation[0];
		
		...
	
	}

	//packet.h

	static inline uint16_t get_src_port(int num_ports, int probe_num, uint32_t *validation)
	{
		return zconf.source_port_first + ((validation[1] + probe_num) % num_ports);
	}
	
因为在zmap 的tcp synscan机制中，tcp协议字段中的源端口sport和seq是没有作用的，所以将哈希值重载到了th_sport字段。

在接收报文时通过解析对应的被重载过的字段，来判断接收到的包是不是对于我们scan的回应，这样就实现了不需要再scan时通过保存状态信息来识别
对应的response。



<br />


### Packet Transmission and Receipt

<br />




---

# Zgrab

<br />

## Usage

<br />

`sudo zmap -p 443 --output-field=* -O csv -r 10000 | ztee results.csv | ./zgrab --port 443 --tls --http="/" --output-file=banners.json`











