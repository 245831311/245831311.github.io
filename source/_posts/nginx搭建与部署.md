---
title: nginx的搭建
date: 2017-12-00 12:00:00
categories: 
- nginx总结
tags:
- 总结
---

### nginx是什么
nginx是一个开源且高性能、可靠的http中间件、代理服务

### 为什么使用nginx（详细在下一章总结）
- io多路复用(select,epoll等模型实现)
- cpu亲和性
- 轻量级

### nginx的使用场景 （详细在另外一章总结）
1. 代理服务
    1. 正向代理
    2. 反向代理
2. 缓存服务
3. https服务
4. 静态web服务
    1. 跨域
    2. 防盗链
    3. 客户端缓存
5. 动静分离
6. nginx+lua 
    1. 流量监控
    2. waf
    3. 灰度发布(建议使用openresty)
7. rewrite规则
    1. 后台维护页面
    2. seo搜索优化
    3. 安全（伪静态）
    4. url访问跳转（新旧域名替换）

### nginx下linux的部署(fedora27)
    简单说明下安装nginx的所需要的依赖，并不做详细讲解
1. 下载prec,zlib,openssl
2. yum install -y gcc g++ (c编译工具)
3. yum install -y pcre pcre-devel(重写rewrite,正则表达式)
4. yum install -y zlib zlib-devel(gzip压缩)
5. yum install -y openssl openssl-devel
6. 下载nginx（可到官网上下载.tar文件）
7. 解压 tar -zxvf nginx-xxxx.tar.gz
8. 进入解压目录，并执行./configure --prefix=/usr/local/nginx  
9. make && make install

### nginx的目录配置说明
├── auto            自动检测系统环境以及编译相关的脚本<br>
│   ├── cc          关于编译器相关的编译选项的检测脚本<br>
│   ├── lib         nginx编译所需要的一些库的检测脚本<br>
│   ├── os          与平台相关的一些系统参数与系统调用相关的检测<br>
│   └── types       与数据类型相关的一些辅助脚本<br>
├── conf            存放默认配置文件，在make install后，会拷贝到安装目录中去<br>
├── contrib         存放一些实用工具，如geo配置生成工具（geo2nginx.pl）<br>
├── html            存放默认的网页文件，在make install后，会拷贝到安装目录中去<br>
├── man             nginx的man手册<br>
└── src             存放nginx的源代码<br>
    ├── core        nginx的核心源代码，包括常用数据结构的定义，以及nginx初始化运行的核心代码如main函数<br>
    ├── event       对系统事件处理机制的封装，以及定时器的实现相关代码<br>
    │   └── modules 不同事件处理方式的模块化，如select、poll、epoll、kqueue等<br>
    ├── http        nginx作为http服务器相关的代码<br>
    │   └── modules 包含http的各种功能模块<br>
    ├── mail        nginx作为邮件代理服务器相关的代码<br>
    ├── misc        一些辅助代码，测试c++头的兼容性，以及对google_perftools的支持<br>
    └── os          主要是对各种不同体系统结构所提供的系统函数的封装，对外提供统一的系统调用接口<br>



### nginx相关命令
```nginx
./nginx 启动
./nginx -s stop 关闭
./nginx -s reload 重启
./nginx -tc nginx 配置文件   检测nginx配置文件语法
```

### **<font color="red"> 注意 </font>**
- 在重新编译nginx时候，先将原有的nginx二进制文件备份,再执行./configure && make && make install 以免被新的nginx覆盖
- 若直接使用yum最新版本的openssl,则会在编译某些旧版本nginx的时候出现不兼容的情况。原因是由于openssl新版本改动得挺大导致。解决方案
    - 升级nginx版本
    - 降级openssl的版本(额外章会记录如何降级，这里遇到了一个坑) 

### 参考文献
1. [nginx从入门到精通](http://tengine.taobao.org/book/)
2. [nginx中文官方文档](http://www.nginx.cn/doc/)
