---
title: frp穿透
date: 2021-07-25 11:16:39
categories: 
- 视频
tags:
- 视频
---

# frp穿透

主要介绍如何将内网摄像头的rtsp通过frp穿透工具映射到互联网上。

准备：

摄像头ip：172.16.10.2

内网服务器：10.144.21.90（与摄像头是互通）

互联网服务器：47.106.89.193

frp工具包：frp_0.35.1_linux_386.tar.gz


## 操作步骤

1. 两台分别安装frp穿透工具
```
wget https://github.com/fatedier/frp/releases/download/v0.35.1/frp_0.35.1_linux_386.tar.gz
```

2. 配置服务端

互联网服务器47.106.89.193作为服务端,配置frps.ini

```
[common]
bind_port = 7000    #绑定端口

[rtsp]
listen_port = 10000 #监听端口
```

开启服务端监听命令
```
nohup ./frps -c ./frps.ini &
```

3. 配置客户端

内网服务器10.144.21.90作为客户端,配置frpc.ini

```
[common]
server_addr = 47.106.89.193 #服务端ip
server_port = 7000  #服务端绑定的端口

#[ssh]
#type = tcp
#local_ip = 127.0.0.1
#local_port = 22
#remote_port = 6000

[rtsp]
type = tcp
local_ip = 172.16.10.2  #摄像头Ip
local_port = 554    #摄像头端口
remote_port = 10000 #服务端监听得端口
```

开启服务端监听命令
```
nohup ./frpc -c ./frpc.ini &
```

4. 结果

rtsp://admin:a12345678@47.106.89.193:10000/h264/ch33/main/av_stream


## 参考
[frp映射摄像头地址及端口](https://www.cnblogs.com/liusingbon/p/12623774.html)

[frp 内网穿透工具](https://www.oschina.net/p/frp?hmsr=aladdin1e1)