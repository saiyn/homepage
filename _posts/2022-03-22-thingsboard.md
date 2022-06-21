---
layout: post
title:  "thingsboard 笔记"
date:   2022-03-23 15:15:54
categories: Linux
excerpt: iot thingsboard platform
---

* content
{:toc}


# Run docker with image build from source code

<br />

## Notes when rebuild and deploy a new docker image

<br />

tb-postgres的image内部嵌入了postgres server, 启动逻辑是如果检测到`~/.mytb-data/`目录下如果有`db`目录，那就不会在去创建postgres server，这样导致重新build no cache的image进行
运行后出现无法连接到postgres server的问题。

另外在`~/.mytb-data/`目录下还会生成如下两个隐藏文件，如果不删除`.firstxx`会导致新建的postgres不会创建默认的thingsboard db.

所以，在build出新的iamge后，需要删除`~/.mytb-data/`目录下的db和隐藏文件，不过应该考虑更好的做法。


![tb-db_1](https://raw.githubusercontent.com/saiyn/homepage/gh-pages/images/tb-db_1.png)

<br />


# code flow

<br />

## bulk import csv data

<br />


```
  AbstractBulkImportService (saveKvs()) -> DefaultTelemetrySubscriptionService (saveANdNotify()) -> BaseTimeseriresService (save()) ->  
    -> savePartition()
    -> saveLatest()
    -> save()
    
```   
