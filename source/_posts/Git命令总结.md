---
title: Git命令总结
author: linWang
categories:
date: 2022-01-06 21:29:04
tags: git命令
top: true
---
>   本文总结了平时生活中经常用到的Git命令，详细的描述了命令的用法，后续还会继续补充，方便以后查看。

<!--more-->

## Git的配置命令

### 初始化仓库

```shell
# 1.进入某个目录
cd d:/gitDemo
# 2. 初始化(执行之后，在当前目录下会有一个.git文件夹)
git init
```

### 配置Git的邮箱和用户名

```shell
# 设置git的全局邮箱地址
git config --global user.email "903862530@qq.com"
# 设置git的全局用户名
git config --global user.name "linwang-zf"

# 查看全局配置的用户名和邮箱地址
git config --global user.email
git config --global user.name
```

## Git常用命令

### add 和 commit

```shell
# 1.将工作区的修改提交到暂存区
git add .
# 2.将暂存区的内容提交到版本库
git commit -m "note"

# 如果修改的文件已经被Git管理，则上述命令可以简化为
git commit -am "note"
```

### status

```shell
# 该命令会比较 <工作区和暂存区的内容> 和 <暂存区和版本库的内容>
# 比较结果用颜色区分：
# 	（1）工作区和暂存区的内容不一致时，此时文件的颜色为红色
# 	（2）暂存区和版本库的内容不一致时，此时文件的颜色为绿色
git status
```

### cherry-pick

`git cherry-pick`可以将**某次提交或者某几个提交(其他分支)**检入当前分支，详细的操作如下：

```shell
# 将某次提交的commitHash值检入当前分支
git cherry-pick <commitHash>
```

举例来说，代码仓库目前有两个分支master和feature，提交历史如下图

![cherry-pick演示](image-20220110230544989.png)

现在将feature分支的提交f应用到master分支上，命令如下：

```shell
# 1.切换到master分支
git checkout master
# 2.Cherry Pick操作
git cherry-pick f
```

除了上述的单次提交之外，cherry-pick还支持**检入多次提交**以及**一段连续提交**到当前分支，命令如下：

```shell
# 将A和B两个提交检入当前分支
git cherry-pick <HashA> <HashB>

# 将A到B的所有提交检入当前分支(提交A必须先于提交B，且不会包含A提交)
git cherry-pick A..B (不包含A提交,相当于(A,B])
git cherry-pick A^..B (包含A提交,相当于[A,B])
```

当检入的过程中发生了冲突，此时cherry-pick将停下来，让用户决定如何操作：

```shell
# 发生冲突后，取消操作，并回退到操作之前的样子
git cherry-pick --abort
# 发生冲突后，取消操作，但是不会回退到操作之前的样子
git cherry-pick --quit

# 解决完冲突后,并继续执行(最常用的方式)
git add .
git cherry-pick --continue
```

`cherry-pick`命令还支持配置参数，命令如下(命令中的 | 表示两者都可以)

```shell
# 在提交的时候，打开外部编辑器，可以编辑提交信息，如果发生冲突，在解决冲突之后仍然会打开
git cherry-pick -e | --edit <commitHash>

# 只将commitHash提交更新到工作区和暂存区，不提交到版本库
git cherry-pick -n | --no-commit <commitHash> 

# 提交的信息后面增加一行 “(cherry picked from commit ...)”，方便以后查到这个提交是如何产生的
git cherry-pick -x <commitHash>

# 在提交信息的末尾追加一行操作者的签名，方便以后查到这个提交是如何产生的。
git cherry-pick -s | --signoff <commitHash>

# 当commitHash提交是一个merge提交时，cherry-pick将默认失败，此时可以采用-m告诉git应该采用哪个分支的变动
# 1表示接受变动的分支(merge时所处的分支)，2表示变动来源的分支(merge时指定合并的分支)
git cherry-pick -m | --mainline 1 <commitHash> 
```

`cherry-pick`还支持将远程仓库的某次提交检入当前分支，具体命令如下：

```shell
# 增加远程仓库的地址
git remote add origin git://gitAddress
# 将远程仓库抓取到本地
git fetch origin
# 查找将要检入的commitHash
git log origin/master
# 将上述获取的commitHash检入当前分支
git cherry-pick <commitHash>
```

### stash

某一天你在dev分支上开发新需求时，突然被要求改一个线上的紧急Bug，此时你新的功能还未开发完成，不想进行提交。这个时候你就可以采用stash命令，将工作区和暂存区的内容放入栈中，然后切换到其他分支修复bug，当修复完成后，再从栈顶中取出刚刚的内容继续后续的开发。详细的命令如下：

```shell
# 保存当前工作区和暂存区的修改内容,括号中的命令可以增加注释信息
git stash (git stash save "message")

# 显示栈中保存信息的列表
git stash list
```

当需要从栈中取出之前保存的信息时，可以采用下面的命令：

```shell
# 从栈顶取出一次改动内容，并从栈中删除(恢复最新的进度到工作区)
git stash pop

# 恢复最新的进度到工作区和暂存区(尝试将原来暂存区的改动还恢复到暂存区)
git stash pop --index

# 将指定的进度恢复到工作区
git stash pop stash@{1}
```

下面简要分析加--index和不加的区别。

![git statsh之前的文件状态](image-20220111223151327.png)

从上图中可以看出，testStash文件此时处于暂存区的改动（即已经进行了add操作，还未进行commit操作），这时采用`git stash`将修改内容保存到栈中，当我们用`git stash pop`命令取出内容时，效果如下：

![git stash pop操作结果](image-20220111223513807.png)

从上图中可以看到，如果不加--index，Git会将所有的改动全部放入工作区(即还未add的状态)，如果加了--index则和之前的状态保持一致。

如果恢复保存进度的时候不想要删除栈中的进度，则可以使用下面的命令：

```shell
# apply命令和pop命令的用法完全一致，区别仅在于pop会删除进度，而apply不会
git stash apply 
```

**删除栈中进度命令**

```shell
# 删除stash_id对应的进度，如果没有指定，则删除最新的进度
git stash drop [stash_id]

# 清除栈中所有存储的进度
git stash clear
```

### rebase

#### rebase常用命令



#### rebase和merge的区别

当我们在自己的分支上开发完成后，通常需要合并其他分支，我们一般会使用merge命令来合并，但是其实也可以采用rebase命令。

假设git当前的Commit提交记录如下所示：

![git原始commit图](image-20220113231142561.png)

此时分别在master分支和mywork分支上进行两次提交，提交之后状态如下图：

![提交两次之后的commit图](image-20220113231336735.png)

此时在master上合并mywork分支，如果使用merge命令，则合并之后的图如下所示：

![merge合并示意图](image-20220113231555007.png)

如果采用rebase合并，则合并之后如下所示：

![rebase合并示意图](image-20220113231647697.png)

从上述分析可以知道，rebase在合并的时候采用的方法时变基，这样的好处是让提交更整齐，但是可能会带来很多冲突；merge合并则直接将两个分支进行合并为一个分支，简单粗暴。

## Git远程仓库命令

### 删除远程仓库

```shell
# 将空分支推送到远程dev分支，即删除远程dev分支
1. git push origin :dev

# 删除远程的dev分支
2. git push --delete origin/dev
```



### 筛选远程仓库地址

```shell
# 查看远程仓库的地址,只显示一行，并筛选中第二列元素(仓库地址)
git remote -v|head -1|awk '{print $2}'
```
### 关联远程分支

在关联远程分支的同时，在本地创建一个同名分支
```shell
# 关联远程dev分支，并在本地创建一个同名dev分支
git checkout --track origin/dev
```

### 同步远程分支

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

