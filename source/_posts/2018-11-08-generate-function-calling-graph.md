---
title: generate function calling graph
date: 2018-11-08 23:13:17
tags: 
  - graphviz
  - doxygen
---

# 生成函数调用图

目的:记录ubnutu下doxygen和graphviz生成函数调用关系图

## 安装graphviz
命令行输入`sudo apt-get install graphviz`

## 安装doxygen
进入doxygen官网[下载页面](http://www.stack.nl/~dimitri/doxygen/download.html)
按照提示git clone源码
```shell
git clone https://github.com/doxygen/doxygen.git
cd doxygen
mkdir build
cd build
cmake -G "Unix Makefiles" ..
make
```
在cmake这一步出了问题,报错 `Could NOT find FLEX (missing: FLEX_EXECUTABLE)`
网上搜索一番,需要安装flex和bison
命令行输入`sudo apt-get install flex bison`解决

- 据悉,doxygen安装可以使用命令行`sudo apt-get install doxygen`一键完成

至此需要的工具已经全部安装完成

## 生成函数调用图
1.下载代码
因为想学习redis源码,本次以redis的源码为例,首先搜索个版本最低的redis,路径为https://code.google.com/archive/p/redis/downloads?page=7 我们选择[redis-beta-1.tar.gz](https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/redis/redis-beta-1.tar.gz)作为入门代码,下载完成后解压

2.生成doxygen文件Doxyfile
`doxygen -g`生成带有注释的配置文件,`doxygen -s -g`生成不带注释的文件,生成文件文件名Doxyfile

3.修改Doxyfile
略

4.生成
命令行输入`doxygen Doxyfile`

5.查看
本人相对来说喜欢html格式,因为可以本地查看,或者搭建httpserver作为静态资源当doc,应该也可以部署到github pages上.展示其中的main函数调用,效果图如下:![](https://s1.ax1x.com/2018/11/09/iH5yWj.png)
