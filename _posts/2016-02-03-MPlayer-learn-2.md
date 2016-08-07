---
layout: post
title:  "MPlayer 学习应用笔记(二)"
date:   2016-02-03 15:15:54
categories: MPlayer
excerpt: MPlayer linux
---

* content
{:toc}

记录开发中实际涉及的MPlayer知识

---

## 启动MPlayer的代码执行流程

<pre><code>1)系统初始化
play_next_file:
......
2)打开播放文件构造`stream`以及调用demuxer去识别文件格式:
	2.1)Open & Sync STREAM	
		......
		current_module="open_stream";
        mpctx->stream = open_stream(filename, 0, &mpctx->file_format);
		.......
		if(0 == mpctx->stream->end_pos){
			/*如果播放的是不准确的文件则删除当前文件然后跳到`goto_next_file`*/
			sprintf(tmp_arr, "%s%s", "rm -rf ", filename);
		if(system(tmp_arr) < 0){
			mp_msg(MSGT_CPLAYER,MSGL_INFO,"remove illegal music file err!!!\n");
		}
			goto goto_next_file;
			
		}

	2.2)Open DEMUXERS --- DETECT file type
		......
		current_module="demux_open";
		mpctx->demuxer=demux_open(mpctx->stream,mpctx->file_format,audio_id,video_id,dvdsub_id,filename);
		
		......
		if(!mpctx->demuxer){
		/*如果在demux中未能成功被过滤器识别，则跳到`goto_next_file`*/
		goto goto_next_file;
	}
	......
main:	
3)播放的主体部分
	......
	3.1)  //============================ SETUP AUDIO ===============================
	    ......
		/*为该音频流寻找最佳的codec*/
		reinit_audio_chain();
				->init_best_audio_codec();
					->init_audio();
						->{
						    // ok, it matches all rules, let's find the driver!
		                    for (i = 0; mpcodecs_ad_drivers[i] != NULL; i++){
		                         if (!strcmp(mpcodecs_ad_drivers[i]->info->short_name, sh_audio->codec->drv)){
				                     break;
							 }
							 /*mpcodecs_ad_drivers[]数组里面就是注册的各种codec了*/
							 mpadec = mpcodecs_ad_drivers[i];
		                  
						  }
		
						  

	3.2)  START PLAYING
	/*真正开始输出音频数据*/
	while(!mpctx->eof){
		......
		if(!mpctx->sh_audio && mpctx->d_audio->sh) {
			mpctx->sh_audio = mpctx->d_audio->sh;
			mpctx->sh_audio->ds = mpctx->d_audio;
			reinit_audio_chain();
		}
		//============================ PLAY AUDIO ===============================
		if (!fill_audio_out_buffers()){
				//mp_msg(MSGT_CPLAYER,MSGL_INFO,"\n at eof!!!!!!:\n");
				mp_msg(MSGT_CPLAYER,MSGL_INFO,"ANS_END=at end of !!!!!:\n");
				// at eof, all audio at least written to ao
				if (!mpctx->sh_video)
					mpctx->eof = PT_NEXT_ENTRY;
			}
		
		......
		if(!mpctx->sh_video) {
			// handle audio-only case:
			double a_pos=0;
			// sh_audio can be NULL due to video stream switching
			// TODO: handle this better
			if((!quiet || end_at.type == END_AT_TIME) && mpctx->sh_audio)
				a_pos = playing_audio_pts(mpctx->sh_audio, mpctx->d_audio, mpctx->audio_out);
			if(end_at.type == END_AT_TIME && end_at.pos < a_pos)
				mpctx->eof = PT_NEXT_ENTRY;
			update_subtitles(NULL, a_pos, mpctx->d_sub, 0);
			update_osd_msg();
		}
		
		......
		
		//============================ Handle PAUSE ===============================
		current_module = "pause";
		......
		
		//================= Keyboard events, SEEKing ====================
		current_module = "key_events";
		......
		
	}

goto_next_file: 
4)播放结束，转到下一个文件
	......
	/*清理为了播放上个文件所用到的各种资源*/
	while(mpctx->playtree_iter != NULL) {
		filename = play_tree_iter_get_file(mpctx->playtree_iter,mpctx->eof);
		if(filename == NULL) {
			if(play_tree_iter_step(mpctx->playtree_iter,mpctx->eof,0) != PLAY_TREE_ITER_ENTRY) {
			    play_tree_iter_free(mpctx->playtree_iter);
			    mpctx->playtree_iter = NULL;
			};
		} else
			break;
	}
	......
	if(use_gui || mpctx->playtree_iter != NULL || player_idle_mode){
		if(!mpctx->playtree_iter) filename = NULL;
		mpctx->eof = 0;
		
		goto play_next_file;
	}
</code></pre>	
	
	
	
	
	
	
	
	
	
	
	
