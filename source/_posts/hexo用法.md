---
title: hexo的搭建与使用
date: 2017-02-22 12:00:00
categories: 
- hexo总结
tags:
- 笔记
---

### Hexo 是什么
    hexo是基于node.js的一个简洁的博客框架。hexo是一款使用markdown渲染，快速生成静态页面的框架。

### 为什么使用hexo作为博客
* 不走数据库，可以通过md作为存储文件
* 高效快速，生成和渲染静态页面快
* 部署发布简单，操作简便

### hexo部署
一. 安装node.js（网上很多，不详细解释）
安装完毕后可以输入node -v查看是否安装成功.

二. 安装hexo
```html
#由于npm的源比较慢，建议先换成淘宝的镜像
npm config set registry "https://registry.npm.taobao.org"

#安装hexo客户端
npm install hexo-cli g

#安装服务端插件
npm install hexo-server --save

#安装hexo发布插件
npm install hexo-deployer-git --save
```
### hexo相关语法
```hexo
1.hexo init 文件夹 初始化文件夹为hexo目录形式
2.hexo server(hexo s) 开启服务端,默认4000
3.hexo s -p 5000 开启服务端并设置端口号
4.hexo generate(hexo g) 生成静态页面
5.hexo clean 清空静态页面
6.hexo deploy 发布
7.hexo g -d 生成并发布
8.hexo g -s 生成并开启服务
```

