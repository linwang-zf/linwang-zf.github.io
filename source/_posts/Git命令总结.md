---
title: Git命令总结
author: linWang
categories: blog
date: 2022-01-06 21:29:04
tags:
top: true
---
## Git远程仓库命令
### 筛选远程仓库地址
```shell
# 查看远程仓库的地址,只显示一行，并筛选中第二列元素(仓库地址)
git remote -v|head -1|awk '{print $2}'
```
<!--more-->
### 关联远程分支
在关联远程分支的同时，在本地创建一个同名分支
```shell
# 关联远程dev分支，并在本地创建一个同名dev分支
git checkout --track origin/dev
```
## 同步远程分支
同步远程分支，并且可以删除远程不存在的本地分支
```shell
# 同步远程分支
1. git remote prune origin
```
## 分支命令
```shell
# 将某个提交(其他分支)检入当前分支
git cherry-pick hashCommit

# 将某个分支连续的提交检入当前分支(包含开始的提交的分支e1223b03)
git cherry-pick e1223b03^..47eee952
# 发生冲突之后，解决冲突
git add .
git cherry-pick --continue

# 查看分支名，并匹配以*开头的分支(当前分支)，并打印最后一个字符串的值(以空格分割)
git branch|egrep '^\*'|awk '{print $NF}'
```
## checkout相关命令
```shell
# 将dev分支的某个文件检入master分支
1. git checkout master
2. git checkout --patch dev filePath
```

