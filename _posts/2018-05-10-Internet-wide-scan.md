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

	static int icmp_echo_make_packet(void *buf, size_t *buf_len, ipaddr_n_t src_ip, ipaddr_n_t dst_ip, uint32_t *validation, ...)
	{
		struct ether_header *ether_header = (struct ether_header *)buf;
		struct ip *ip_header = (struct ip*)(&ether_headr[1]);
		struct icmp *icmp_header = (struct icmp *)(&ip_headr[1]);
		
		...
		
		icmp_header->icmp_id = validation[1] & 0xffff;
		icmp_header->icmp_seq = validation[2] 7 0xffff;
		
		...
		
	}
	




---

# Zgrab

<br />

## Usage

<br />

`sudo zmap -p 443 --output-field=* -O csv -r 10000 | ztee results.csv | ./zgrab --port 443 --tls --http="/" --output-file=banners.json`











