---
title: nginx支持流媒体
date: 2021-07-25 11:16:39
categories: 
- 视频
tags:
- 视频
---

## 简介
本章介绍如何搭建一个nginx的流媒体服务器，可以直接通过nginx访问hls的m3u8与rtmp的链接。

## 安装

安装nginx其实没有什么值得说，主要是要下载nginx的nginx-rtmp-module模块,编译时候加下此模块就OK了

下载nginx-rtmp-module：https://github.com/arut/nginx-rtmp-module

```
./configure --prefix=/usr/local/nginx --with-pcre=/home/user/pcre/pcre-8.32 --with-zlib=/home/user/zlib/zlib-1.2.8 --with-openssl=/home/user/openssl/openssl-1.0.1i  --add-module=/home/user/nginx-rtmp-module
```

## 配置

- RTMP
```
rtmp {                #RTMP服务
   server {
       listen 1935;  #//服务端口
       chunk_size 4096;   #//数据传输块的大小
       application vod {
         play /opt/video/vod; #//视频文件存放位置。
       }
   }
}


http {
    include      mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    server {
        listen      80;
        server_name  localhost;
        
        #配置nginx的rtmp一览页面
        location /stat {
                rtmp_stat all;
            rtmp_stat_stylesheet stat.xsl;
        }
        location /stat.xsl {
           root /etc/rtmpServer/nginx-rtmp-module/;
        }

        location / {
            root  html;
            index  index.html index.htm;
        }

        error_page  500 502 503 504  /50x.html;
        location = /50x.html {
            root  html;
        }
    }
}
```


- HLS

该配置是播放hls的配置
```
location /hls {  
    types{  
        application/vnd.apple.mpegurl m3u8;  
        video/mp2t ts;  
    }  
    suffix m3u8;
   #配置一个根路径 
   root /data/baiyun/; 
   add_header Cache-Control no-cache;
   add_header 'Access-Control-Allow-Origin' '*';
   add_header 'Access-Control-Allow-Credentials' 'true';
}
```

该配置是支持远程推流到nginx负责切片成m3u8的配置
```
rtmp {
    server {
        listen 1935;
        chunk_size 4000;
        #HLS
        application hls {
            live on;
            hls on;
            #视频流存放地址
            hls_path /usr/local/nginx/html/hls; 
            hls_fragment 5s;
        }
    }
}
```

## 参考
[视频直播点播nginx-rtmp开发手册中文版](https://blog.csdn.net/weiyuefei/article/details/74001589)