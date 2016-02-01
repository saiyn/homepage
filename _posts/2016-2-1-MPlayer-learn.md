---
layout: post
title:  "MPlayer 学习应用笔记"
date:   2016-2-1 15:15:54
categories: MPlayer
excerpt: MPlayer linux
---

* content
{:toc}

记录开发中实际涉及的MPlayer知识

---

##启动MPlayer的代码执行流程
项目中我从TF卡中读取所有被支持的音频文件，创建成一个简单的`playlist.txt`。应用程序中收到播放音乐命令时通过以下代码启动MPlayer。
<pre><code>ret = execlp("mplayer", "mplayer",
			 "-slave", "-input",
			 "file=/tmp/my_fifo",
			 "-playlist",
			 "/usr/sbin/playlist.txt",
			 "-loop", "0",
			 NULL
			);
</code></pre>

`-slave`选项是将MPlayer设置为slave模式，这样就可以通过向MPlayer进程发送命令来控制其运行状态。通过`-input`选项来指定如何传输命令。

###`-input`选项详细说明

命令	|描述
----|---
conf=<文件>		|读取另外的input.conf.如果没有给出路径名，将假设是`~/.mplayer`.
ar-delay		|在开始自动重复一个键之前等待多少毫米(0表示禁用)。
ar-rate			|当自动重复时，每秒重复多少次。
keylist			|列出所有可以被绑定的键。
cmdlist			|列出所有可以被绑定的命令。
js-dev			|指定可用的游戏操纵杆设备。
file			|从指定文件读取命令，用于命令管道。

上面我们就是通过`file=/temp/my_fifo`指定`my_fifo`这个命令管道来接受我们应用程序发送过来的命令。




















