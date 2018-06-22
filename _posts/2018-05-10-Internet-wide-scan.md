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

### zmap的addressing probes

<br />

在ip地址扫描策略上，zmap进行了特别的优化，相对于nmap逐个按顺序的扫描ip地址，zmap进行散列式的分组随机扫描。nmap在扫描时需要维护每次扫描时建立的连接状态，以避免重复扫描，而zmap通过重载每个发送包中没有使用到的字段来区分接收到的包，实现无状态的不重复的扫描。下面是具体的实现代码：

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


![zmap_3](http://omp8s6jms.bkt.clouddn.com/image/git/zmap_3.png)

<br />

具体到icmp echo报文，格式如下:

![zmap_4](http://omp8s6jms.bkt.clouddn.com/image/git/zmap_4.png)

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

---

# Zgrab

<br />

## Usage

<br />

`sudo zmap -p 443 --output-field=* -O csv -r 10000 | ztee results.csv | ./zgrab --port 443 --tls --http="/" --output-file=banners.json`











