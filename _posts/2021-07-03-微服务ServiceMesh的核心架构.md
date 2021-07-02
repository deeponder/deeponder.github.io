---
layout:     post
title:      微服务ServiceMesh的核心架构
subtitle:   
date:       2021-07-03
author:     jabin
header-img: 
catalog: true
tags:
    - 计算机
    - 架构
    - 微服务
    - ServiceMesh
---

# 一、整体架构
![图片描述](/img/sm-roadmap.png)
![图片描述](/img/sm-basic.png)
# 二、协议
## 2.1 协议格式
### 2.1.1 CS
1. ws:  [头长度][包头][json格式包体]
2. http
   服务处理来自外部系统的CS包时，还需把请求信息(参数、cookie等)转换为pb，存进包体

### 2.1.2 SS
格式： [包头长度][pb序列化后的包头][序列化后的包体]
1. 包头： servicename, funcname, 包体序列化方式, 路由规则，xxxx
2. 包体:  pb定义

## 2.2 打解包函数
1. 包头直接pb序列化，反序列化
2. 包体json/pb

# 三、边车代理--go
## 3.1 集群节点注册、发现
1. 启动时，从etcd， 拉取所有服务节点的ip端口，并建立连接，加入服务节点的连接池
2. 监听etcd的服务节点，若有节点上线/下线， 建立/断开连接，加入连接池/删除连接池中下线的服务节点
3. 负责同pod/本机节点的注册、反注册

## 3.2 流量转发--可做流量拦截，调用链买点上报
1. 监听两个端口， 分别处理来自本机/跨机的连接、流量
2. 解包，拿到目标servicename+funcname， 以及路由策略（轮询/一致性hash/广播）
3. 通过连接池，找到对应节点的连接，write

# 四、业务接入层--go
## 4.1 协议支持
1. http
2. websocket

## 4.2 http/ws路由uri的服务信息发现
1. 与边车代理的集群节点注册、发现逻辑类似
2. 这里主要获取uri对应的servicename、funcname等路由信息

## 4.3 登录鉴权
1. qq、wx、其它

## 4.4 构造rpc请求包
http/ws请求数据，转成约定的集群内部rpc的pb格式

