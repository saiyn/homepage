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

## Usage

<br />

do upnp protocol scan - `sudo zmap -p 1900 --probe-module="upnp" -O json -r 10000`



we need more specific information, so try to figure out what's else zmap can offer us - `zmap --probe-module="upnp -O json --list-output-fields"


use -f options to add what we are interested - `sudo zmap -p 1900 --probe-module="upnp" -O json -r 10000 -f "saddr, classifiction, location"

`

