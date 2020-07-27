---
title: "微服务下的网关Kong"
date: 2018-03-30T23:18:36+08:00
lastmod: 2019-03-30T23:18:36+08:00
draft: false
author: "WithLin"
tags: ["网关", "Kong", "中间件"]
categories: ["Kong", "网关"]
toc: true
comment: true
autoCollapseToc: false
---

Kong是Mashape开源的高性能高可用API网关和API服务管理层。它基于OpenResty，进行API管理，并提供了插件实现API的AOP。Kong在Mashape 管理了超过15,000 个API，为200,000开发者提供了每月数十亿的请求支持。本文将从架构、API管理、插件三个层面介绍Kong。
## 架构
按照康威定律，我们系统架构会拆的很散，系统由一堆服务组成，如下图所示：
![image.png](https://upload-images.jianshu.io/upload_images/5590759-30d176805df8c4b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
库存服务、优惠券服务、价格服务时之前都会做一些特殊处理，如限流、黑白名单，日志、请求统计。而这些处理几乎是所有服务都需要的，这不就是我们常说的AOP嘛，当我们服务多起来的时候，应该将这些通用处理集中到一个地方进行管理，如下图所示：
![image.png](https://upload-images.jianshu.io/upload_images/5590759-0c17ff06ea8814b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
和下图有点相似：
![image.png](https://upload-images.jianshu.io/upload_images/5590759-09a5bc4ce5b7b70c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 1.为什么要用Kong作为NetCore下的API网关？
   1.开源，云原生(Cloud-Native)，ServiceMesh,快速，弹性，RESTful还有分布式微服务的抽象层
   2.基于NGINX构建的网关，拥有更高的性能,并且在2015开源
   3.活跃的社区，在github上有111个Contributors,修复bug迅速，基本每3个月一个版本
   4.支持插件化，目前支持的插件有32个，包含授权，安全，限流，Serverless，分析和监控，转换，日志。
   5.支持企业版本和社区版本

## 架构预览
#### 基于OpenResty(Nginx & Lua Scripting)
![image.png](https://upload-images.jianshu.io/upload_images/5590759-617015431e901e14.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图很清晰的看见Kong的架构图，以Nginx作为基础, OpenResty构建RESTful，支持集群和数据库存储数据，插件化，还有支持用RESTful来管理端。

#### 集群架构预览
![image.png](https://upload-images.jianshu.io/upload_images/5590759-26f72ed9c45459eb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这里讲下Kong的集群原理吧，Kong在0.11.0版本之前用的是serf来做集群的，那么为什么不用serf做集群呢？开发者给出的理由如下:
        1.依赖serf，serf并不属于Nginx/OpenResty
        2.这种依赖相互间通信来同步的机制对于deployment和容器化都有些不便
        3.在运行的Kong节点触发serf需要一些阻塞的I/O
0.11.0版本的实现思路是以数据库为中心，增加一个cluster events的表，任何Kong node都可以向数据库发送变更消息，其他节点轮训数据库改动，然后更新缓存内容，如果有节点重启连上数据库节点就可以工作了。

#### Kong的安装
 Kong的安装方式支持很多主流的平台，目前不支持Windows，支持的安装方式如下：
![image.png](https://upload-images.jianshu.io/upload_images/5590759-1bd0377da7ff3681.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

   Kong的安装，为了方便我这里就使用docker安装了
  1.创建专属kong的网络（docker的最佳实践）--link 过时了啊
   ```
   docker network create kong-net
   ```
  2.选择你使用的数据库，默认使用的是PostgreSQL
     如果你使用的是Cassandra数据库：
     提示下:Cassandra >=3.0
   ```
     docker run -d --name kong-database \
              --network=kong-net \
              -p 9042:9042 \
              cassandra:3
   ```
   如果你使用的是PostgreSQL
   ```
    docker run -d --name kong-database \
              --network=kong-net \
              -p 5432:5432 \
              -e "POSTGRES_USER=kong" \
              -e "POSTGRES_DB=kong" \
              postgres:9.6
   ```
  3.数据库迁移，初始化库表结构：
```
 docker run --rm \
    --network=kong-net \
    -e "KONG_DATABASE=postgres" \
    -e "KONG_PG_HOST=kong-database" \
    -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
    kong:latest kong migrations up
```
  4.启动kong
```
docker run -d --name kong \
    --network=kong-net \
    -e "KONG_DATABASE=postgres" \
    -e "KONG_PG_HOST=kong-database" \
    -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
    -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
    -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
    -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
    -p 8000:8000 \
    -p 8443:8443 \
    -p 8001:8001 \
    -p 8444:8444 \
    kong:latest
```
5.看网关有没有启动
在本机 curl -i http://localhost:8001/，或者用浏览器访问8001端口。如果出来一大堆json，表示成功。

## 以AspNetCore为例子访问
```
mkdir AspNetCore

cd AspNetCore

dotnet new webapi

dotnet run

```
我们以netcore做的api为例子访问localhost:5000/api/values，前面网关搭建起来了，并且支持RESTful，现在有开源的dashboard，我们就用KongDashboard来演示，如何构造搭建和访问。
``` shell
# 全局安装kong-dashboard
npm install -g kong-dashboard

# 启动 kong-dashboard
kong-dashboard start --kong-url http://localhost:8001

# 启动kong-dashboard，并且自定义端口
kong-dashboard start \
  --kong-url http://kong:8001 \
  --port [port]

# 启动kong-dashboard并且启动基础认证
kong-dashboard start \
  --kong-url http://kong:8001 \
  --basic-auth user1=password1 user2=password2

# 看kong-dashboard 启动参数
kong-dashboard start --help
```
启动成功后用浏览器打开localhost:8080如下图所示：
![image.png](https://upload-images.jianshu.io/upload_images/5590759-4b276d5a35034a73.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那么我们增加一个NetCoreAPI，在DashBoard，如图所示：
![image.png](https://upload-images.jianshu.io/upload_images/5590759-ada7586a2db761df.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

因为是GET请求，那我我们用浏览器访问，浏览器 -> 网关 -> NetCore程序。
打开浏览器直接访问http://localhost:8000/api/values,返回["value1","value2"]则代表正常。
如下图所示：
![image.png](https://upload-images.jianshu.io/upload_images/5590759-6974f01f992eaeee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


最后，AspNetCore微服务下的网关-Kong系列，后面会继续更新，会讲解到Kong的插件的使用，插件的开发，使用的一些坑，网关性能分析和日志可视化，源码解析等。