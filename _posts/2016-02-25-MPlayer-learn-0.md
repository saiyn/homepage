---
layout: post
title:  "MPlayer 学习应用笔记(0)"
date:   2016-1-26 15:15:54
categories: MPlayer
excerpt: MPlayer linux
---

* content
{:toc}

记录研究MPlayer源码过程中获得的零散知识点

---

## FourCC

FourCC全称Four-Character Codes, 即四字节代码，其实就是：
<pre><code>typedef unsigned in FourCC
</code></pre>
音视频播放软件通过查询与FourCC代码相关联的音视频解码器来播放特定的音视频流。
比如`DIV3`=`DivX Low-Motion`, `DIV4`=`DivX Fast-Motion`
<pre><code>#define MAKE_FOURCC(a,b,c,d) \
(((uint32)d) | ((uint32)c << 8) | ((uint32)b << 16) | ((uint32)a << 24))
...
switch(val){
	case MAKE_FOURCC('f', 'm', 't', ''):
		...
		break;
		
	case MAKE_FOURCC('Y', '4', '4', '2')
		...
		break;
		
	...
}</code></pre>
通过上面程序可以看出来，一般使用宏生成FOURCC.

## A Guide To Developing MPlayer Codes 

-- by Mike Melanson, who has developed a number of open source decoders for the MPlayer.

> Points:
>
> 1. If the encoded data is stored in a media file format that MPlayer doesn't understand, then you will either need to 
> somehow convert the format to a media file format that the program does understand, or write your own MPlayer file `demuxer`
> that can handle the data.
> 2. vv

> Step:
>
> 1. Modify the local copy of codecs.conf.
> 2. Follow the ad_sample.c or vd_*.c to create a new source file which contains the main decoding function that MPlayer will call to decode data.
> 3. Modify the Makefile so that it will compile your new source file.
> 4. Add your codec to the array in ad.c or vd.c.


## 播放器一般原理

<pre><code>

Source filter -> 
	Demux filter ->
        -->Video Decoder filter  
				-> Color Space Converter filter
					->Video Render filter
					
		-->Audio Decoder filter
			->Audio Render filter
					
</code></pre>

> Demux filter: 
> 1. 解复用过滤器的作用是识别文件类型，媒体类型，分离出各种媒体原始数据，
> 打上时钟信息后送给下级decoder filter.
> 2. 为了识别出不同的文件类型和媒体类型，常规的做法是读取一部分数据，然后遍历解复用过滤器支持的文件格式和媒体格式，
> 做匹配来确定是哪种文件类型，哪种媒体类型。































