---
title: kettle-8.2安装手册
date: 2017-08-21 12:00:00
categories: 
- 大数据
tags:
- 大数据
---

# 1 简介
本编文章主要介绍如何安装kettle 8.2

# 2 准备
  1) 安装jdk1.8(已在上一章说明)
  2) 下载kettle 8.2安装包：*[https://sourceforge.net/projects/pentaho/files/latest/download?aliId=137249511](https://sourceforge.net/projects/pentaho/files/latest/download?aliId=137249511)*
# 3 安装kettle 8.2
  1) 解压下载的安装包
  2) 进入目录pdi-ce-8.2.0.0-342 **->** data-integration **->** ,打开spoon.bat
![spoon-bat](/images/bdata/kettl-pc启动器.png)
  3) 出现以上页面，代表安装成功
![success-page](/images/bdata/kettle-pc界面.png)

# 4 如何运行kettle脚本程序
  1) 下载kettle脚本程序.zip,并重命名成后缀为zip压缩包：*[http://192.168.2.45:1174/AAYQAf__AAAAAQAAAAAAAAAAyUKMcgAORL6xOFABvf-aoA](http://192.168.2.45:1174/AAYQAf__AAAAAQAAAAAAAAAAyUKMcgAORL6xOFABvf-aoA)*
  2) 解压压缩包，获取rest2file.ktr脚本程序
  3) 点击打开按钮并选择存放脚本的指定位置
![open](/images/bdata/kettle打开界面.png)
  4) 执行脚本，并看当前脚本返回结果
![perform](/images/bdata/kettle执行按钮.png)