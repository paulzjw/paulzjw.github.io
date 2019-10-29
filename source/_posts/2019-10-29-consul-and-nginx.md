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