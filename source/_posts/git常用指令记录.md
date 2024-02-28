---
title: git指令用法
date: 2017-02-23 11:16:39
categories: 
- git总结
tags:
- 笔记
---

### 常用git命令的使用
```github
git init 初始化当前文件夹
#分支
git branch -r 查看已有的分支
git checkout -b 分支 创建并切换分支
git checkout 分支 切换分支
git branch 显示当前分支
git branch -d 分支 删除分支
#提交
git add . 添加操作，将将所有修改过的文件添加到版本库的暂缓区内
git commit -m "文字说明" 提交操作，将暂缓区内的代码提交到本地仓库中
git push origin 分支 push到远程仓库中
#版本回退
git reset HEAD^ 回退到上一个版本
git reset --hard commit_id 回退到指定的commit_id的版本中
#添加远程仓库
git remote add origin git@github.com:245831311(github账号名)/245831311.github.io.git(github仓库名)  添加远程仓库
git remote -v 查看origin远程主机名对应远程仓库
git remote rm origin 删除远程仓库
#查看配置
git config -l 查看
#查看git状态
git status
#拉取代码
git fetch 将远程分支上的代码拉取下来并不合并
git rebase
git merge 合并代码
git pull 拉取并与当前分支合并代码
```

### 重点记录
#### 1.git push <远程主机名> <本地分支名>  <远程分支名>
__1.1 git push origin__
 省略了本地分支与远程分支，当前分支与上一次的远程分支存在对应的关系，则会通过这个关系推送出去
__1.2 git push origin master__
 省略了远程分支名，则本地分支会推送到上一次对应存在的远程分支关系(一般同名).不存在则创建新的远程分支
__1.3 git push origin :refs/for/master__
 省略了本地分支名，则相当与删除远程分支
__1.4 git push -u origin master__
-u可以将本地分支与远程分支建立一个upstream的关系，若使用，则本地分支与远程分支建立了对应关系





