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

## A Guide To Developing MPlayer Codes -- by Mike Melanson, who has developed a number of open source decoders for the MPlayer.

> Points:
> 1.If the encoded data is stored in a media file format that MPlayer doesn't understand, then you will either need to 
> somehow convert the format to a media file format that the program does understand, or write your own MPlayer file `demuxer`
> that can handle the data.
> 2.vv













