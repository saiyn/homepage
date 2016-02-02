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

命令			|描述
conf=文件		|读取另外的input.conf.如果没有给出路径名，将假设是`~/.mplayer`.
ar-delay		|在开始自动重复一个键之前等待多少毫米(0表示禁用)。
ar-rate			|当自动重复时，每秒重复多少次。
keylist			|列出所有可以被绑定的键。
cmdlist			|列出所有可以被绑定的命令。
js-dev			|指定可用的游戏操纵杆设备。
file			|从指定文件读取命令，用于命令管道。

上面我们就是通过`file=/temp/my_fifo`指定`my_fifo`这个命令管道来接受我们应用程序发送过来的命令。

###`-playlist`选项

由于我们是想让MPlayer播放我们所建`playlist.txt`中的音乐文件，所以我们通过`-playlist`传入`/usr/sbin/playlist.txt`参数，让MPlayer读取这个文件生成`playtree`。
下面我们来追踪一下MPlayer是如何一步步生成`playtree`的。
<pre><code>mpctx->playtree = m_config_parse_mp_command_line(mconfig, argc, argv);
</code></pre>
在`MPlayer.c`文件中我们可以找到MPlayer的main函数，上面这行代码就是解析`-playlist`选项生成`playtree`的入口。
<pre><code>tmp = is_entry_option(opt,(i+1<argc) ? argv[i + 1] : NULL,&entry);
</code></pre>
进入`m_config_parse_mp_command_line`，我们会发现代码应该会进入上面这句继续执行，里面出现的“if(strcasecmp(opt,"playlist") == 0)”这句话验证了我们的推断。
<pre><code>static int is_entry_option(char *opt, char *param, play_tree_t** ret) {
  play_tree_t* entry = NULL;

  *ret = NULL;

  if(strcasecmp(opt,"playlist") == 0) { // We handle playlist here
    if(!param)
      return M_OPT_MISSING_PARAM;

    entry = parse_playlist_file(param);
    if(!entry)
      return -1;
    else {
       *ret=entry;
       return 1;
    }
  }
    return 0;
}
</code></pre>


















