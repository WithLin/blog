---
title: "GO下GC的优化"
date: 2019-04-08T15:35:56+08:00
lastmod: 2019-04-08T15:35:56+08:00
draft: true
author: "WithLin"
tags: [ "go","GC"]
categories: ["go"]
toc: true
comment: true
autoCollapseToc: false
---

> 最近在看gopherChina2018的视频看到了[基于Go构建滴滴核心业务平台的实践 石松然](https://www.youtube.com/watch?v=w-mm9Oeny5o&list=PLx_Mc4dJcQbl3VCLLQFo-FrF_U4UN1s7b)的在高并发下遇到GC问题，小对象过多引起服务吞吐量的问题。

### 现象

* 随着流量增大，请求超时增多
* 耗时毛刺严重，99分位耗时较长
* 总体内存变化不大


### 排查

* go tool pprof --alloc_objects
* go tool pprof --inuse objects       

##### 某函数生成20%的对象，约800w对象持续被引用


* go tool pprof bin/dupsdc
##### GC扫描函数占用大量CPU，rutime.scanobject，runtime.mallocgc, runtime.greyobject


### 原因

* go的GC 三色算法 启动多了 多个gorutine 并发标记，导致去遍历了那800w个对象，然后CPU资源被利用的比较多。
* 对象数量过多，导致GC三色算法耗费较多CPU

### 优化

原始数据结构    | 优化数据结构     | 优化点
--------|---------|---------
map[string]SampleStruc     | map[[32]byte]SampleStruct      | Key使用值类型避免对map遍历
map[int]*SampleStruct   | map[int]SampleStruct      |   Value使用值类型避免对map遍历
sampleSlice []float64   | sampleSlice [32]float64      |  利用值类型代替对象类型
