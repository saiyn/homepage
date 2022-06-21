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

## Notes when rebuild and deploy a new docker image





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
