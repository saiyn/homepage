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

## steps

* `sudo mvn  install -DskipTests -Ddockerfile.skip=false` to build docker image
* `sudo docker tag thingsboard/tb-postgres saiyn/thingsboard` to update the image which will be replaced in tht running container
* `sudo docker-compose stop` to stop container
* `sudo cat ~/mytb-data/db/postmaster.pid` to check the process of postgres int the container
* wait for a moment to let the stop done, and then `sudo docker run -it -v ~/.mytb-data:/data --rm saiyn/thingsboard upgrade-tb.sh`

if everythings goes fine, then we should see logs like below:

![tb-db_0](https://raw.githubusercontent.com/saiyn/homepage/gh-pages/images/tb-db_0.png)

* `sudo ls ~/.mytb-data/db` to check postmaster.pid exist or not, it should be gone otherwise we will get troblue later
* `sudo docker-compose rm mytb` to remove the old container
* `sudo docker-compose up -d ` to bring everything up to work and all process done


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


# backup 

## script rule node

### parse Gps

```
var new_meta = JSON.stringify(metadata, function(key, value){
    switch (key) {
        case 'ts':
            return new Date().getTime(); 
        
        default:
            return value;
    }
});

var longit = msg.gps.longit;
var latitu = msg.gps.latitu;


var du = Math.floor((Math.floor(longit) / 100));
var fen = parseInt((Math.floor(longit) / 100 + "").split(".")[1]);
var miao = parseInt((longit + "").split(".")[1]) * 60 / 1000000;


var longit_new = parseFloat((du + fen / 60 + miao / 60 / 60).toFixed(6));

var new_msg = {
    longit : longit_new,
    latitu : latitu,
    
};

```









