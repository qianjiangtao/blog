---
title: Git常用技巧
date: 2018-08-22 13:50:39
categories:
- Git
tags:
- Git

---

#### 一.本地代码更新冲突



1.本地的所有修改就都被暂时存储起来 ,放入暂存区

```shell
$ git stash
```

2.更新远程仓库最新代码

```shell
$ git pull
```

<!--more-->

3.将本地之前暂存区代码弹出

```shell
$ git stash pop stash@{0}
```

```java
//意思就是系统自动合并修改的内容，但是其中有冲突，需要解决其中的冲突。
//系统提示如下类似的信息：
Auto-merging c/environ.cCONFLICT (content): Merge conflict in c/environ.c
```

4、解决文件中冲突的的部分 



#### 二.如何将本地创建的项目关联到远程仓库并推送到远程



1.在本地项目的文件夹下，Git仓库初始化 

```shell
# 初始化本地git仓库 
$ git init
Initialized empty Git repository in D:/myblog/.git/
```

2.将本地文件索引添加至git库中

```shell
$ git add * 
```

3.提交代码

```shell
$ git commit -m 'myblog init'
```

4.关联远程仓库

```shell
# blog为远程仓库别名 https://github.com/qianjiangtao/blog.git为远程仓库地址
$ git remote add blog https://github.com/qianjiangtao/blog.git
```

5.查看是否配置生效

```shell
$ git remote -v
blog    https://github.com/qianjiangtao/blog.git (fetch)
blog    https://github.com/qianjiangtao/blog.git (push)
```

6.将代码提交到远程对应分支

```shell
# git push <远程主机名> <本地分支名>:<远程分支名>
$ git push blog master:dev
```

```
Counting objects: 211, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (177/177), done.
Writing objects: 100% (211/211), 590.97 KiB | 0 bytes/s, done.
Total 211 (delta 6), reused 0 (delta 0)
remote: Resolving deltas: 100% (6/6), done.
To https://github.com/qianjiangtao/blog.git
 * [new branch]      master -> dev
```

7.此时查看远程仓库，会发现多了一个dev分支 