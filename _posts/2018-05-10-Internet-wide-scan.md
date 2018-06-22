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

在ip地址扫描策略上，zmap进行了特别的优化，相对于nmap逐个按顺序的扫描ip地址，zmap进行散列式的分组随机扫描。nmap在扫描时需要维护每次扫描时建立的连接状态，以避免重复扫描，而zmap通过一个cyclic multiplicative group机制做到无状态的不重复的扫描。

那么zmap具体是怎么做到如此高效的addressing probes呢？





---

# Zgrab

<br />

## Usage

<br />

`sudo zmap -p 443 --output-field=* -O csv -r 10000 | ztee results.csv | ./zgrab --port 443 --tls --http="/" --output-file=banners.json`











