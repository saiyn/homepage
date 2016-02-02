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

显然得进入`parse_playlist_file`函数继续追踪。
<pre><code>play_tree_t*
parse_playlist_file(char* file) {
  stream_t *stream;
  play_tree_t* ret;
  int f=DEMUXER_TYPE_PLAYLIST;
  stream = open_stream(file,0,&f);

  if(!stream) {
    return NULL;
  }
  ret = parse_playtree(stream,1);
  free_stream(stream);

  play_tree_add_bpf(ret, file);

  return ret;
}
</code></pre>
上面的`open_stream`主要工作是继续调用`open_stream_full()`。
<pre><code>stream_t* open_stream_full(const char* filename,int mode, char** options, int* file_format) {
  int i,j,l,r;
  const stream_info_t* sinfo;
  stream_t* s;
  char *redirected_url = NULL;

  for(i = 0 ; auto_open_streams[i] ; i++) {
    sinfo = auto_open_streams[i];
    if(!sinfo->protocols) {
      continue;
    }
    for(j = 0 ; sinfo->protocols[j] ; j++) {
      l = strlen(sinfo->protocols[j]);
      // l == 0 => Don't do protocol matching (ie network and filenames)
      if((l == 0 && !strstr(filename, "://")) {
	*file_format = DEMUXER_TYPE_UNKNOWN;
	s = open_stream_plugin(sinfo,filename,mode,options,file_format,&r,
				&redirected_url);
	if(s) return s;
	}
	break;
      }
    }
  }

  return NULL;
}
</code></pre>
上面的代码还是比较复杂的，我已经去除了其他的LOG输出和不会进入的程序分支，排除一下干扰，简化一下代码。
BY THE WAY,看到这里我感觉MPlayer的代码写的很一般。
`auto_open_streams[]`是定义在Stream.c中的静态全局数组，其实里面就是注册一些MPlayer所支持的流格式的“处理器”。
查看`auto_open_streams[]`中的所有注册的处理器，我们发现只有`stream_info_file`是用来处理我们传入的`playlist`的。
<pre><code>struct stream;
typedef struct stream_info_st {
  const char *info;
  const char *name;
  const char *author;
  const char *comment;
  /// mode isn't used atm (ie always READ) but it shouldn't be ignored
  /// opts is at least in it's defaults settings and may have been
  /// altered by url parsing if enabled and the options string parsing.
  int (*open)(struct stream* st, int mode, void* opts, int* file_format);
  const char* protocols[MAX_STREAM_PROTOCOLS];
  const void* opts;
  int opts_url; /* If this is 1 we will parse the url as an option string
		 * too. Otherwise options are only parsed from the
		 * options string given to open_stream_plugin */
} stream_info_t;

const stream_info_t stream_info_file = {
  "File",
  "file",
  "Albeu",
  "based on the code from ??? (probably Arpi)",
  open_f,
  { "file", "", NULL },
  &stream_opts,
  1 // Urls are an option string
};
</code></pre>
`stream_info_file`的protocols[]里面是{"file", "", NULL}，所以符合if((l == 0 && !strstr(filename, "://"))代码处的判断，所以程序
会继续调用`open_stream_plugin()`。














