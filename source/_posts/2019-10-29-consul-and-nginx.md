---
title: consul-and-nginx
date: 2019-10-29 19:25:29
tags:
   - consul
   - nginx
---

# 使用consul实现nginx动态负载均衡

## consul简介

consul 是一个 service mesh解决方案，主要提供服务发现、健康检查、KV存储、数据中心等功能。

## 安装consul

consul 使用go语言实现，只需要到官网[https://www.consul.io/downloads.html](https://www.consul.io/downloads.html)下载对应平台的二进制文件即可。 linux和windows平台均可以放在环境变量已有路径或者另外配置环境变量。

## 运行 agent

consul 完成安装后，需要运行agent 。 agent可以分为 server 和 client 模式，每个数据中心至少必须有一台 server ，建议一个集群里有3到5个 server ，部署单一的 server，在出现失败时会不可避免的造成数据丢失。

