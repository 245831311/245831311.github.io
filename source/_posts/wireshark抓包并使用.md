---
title: wireshark抓包并使用
date: 2021-07-24 11:16:39
categories: 
- 视频
tags:
- 视频
---

## 简介
本篇是记录我在平常抓包过程中如何使用wireshark

## 前提

- wireshark工具


## 详解

### 抓包

使用wireshark，首先就要学会怎么抓包，很简单，在linux下使用以下命令
>tcpdump -i eth0 host [ip] -n -vv -A -w 抓包.pcap


### 使用

将抓到的包用wireshark打开就能见到下面的界面

![wireshark界面](/images/video/抓包界面图.png)
<html>
<center>图一</center>
</html>
           
                
这是一份在对接RTSP视频协议时候抓回来的包，很明显可以看到是基于TCP来传输的。以下记录一下我常用到的主要功能：

1. 面板使用

![wireshark面板](/images/video/抓包上方图.png)
<html>
<center>图二</center>
</html>

一般来说，通过面板就能看出交互的具体内容。如图二可以前三行列表可以看出TCP三次握手的交互流程。参考TCP三次握手流程

![image](/images/video/tcp三次握手图.png)
    
2. 数据详细区

![wireshark下半部](/images/video/抓包下方图.png)
<html>
<center>图三</center>
</html>


整个传输过程的传输信息，具体每一层有什么具体信息可以看下相应的资料。由上往下
1. 物理层
2. 数据链路层
3. 网络层（常用）
4. 传输层（常用）
5. 应用层（常用）


3. 追踪流

展示更具体交互信息

>右键点击面板信息>追踪流>TCP流

![追踪流](/images/video/抓包协议追踪图.png)
<html>
<center>图四</center>
</html>

4. 统计丢包率

一般来说，我们也会去观察究竟数据的丢包率是多小，对于音视频的数据我们一般观察RTP包，特别是以UDP传输的时候。
>电话>>RTP>>RTP流

![RTP丢包率](/images/video/抓包rtp统计包.png)



## 参考

[Wireshark抓包使用指南](https://zhuanlan.zhihu.com/p/82498482)
