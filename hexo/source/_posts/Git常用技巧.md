---
title: Git常用技巧
date: 2018-08-22 13:50:39
categories:
- Git
tags:
- Git

---

![](https://gitee.com/qianjiangtao/my-image/raw/master/blog/2018-11-1-13-57.jpg)

<!--more-->

#### 一.本地代码更新冲突



1.本地的所有修改就都被暂时存储起来 ,放入暂存区

```shell
$ git stash
```

2.更新远程仓库最新代码

```shell
$ git pull
```



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

2.关联远程仓库

```shell
# origin本地仓库名 https://github.com/qianjiangtao/spring-cloud-summary.git为远程仓库地址
$ git remote add origin https://github.com/qianjiangtao/spring-cloud-summary.git
```

3.将本地文件索引添加至git库中

```shell
$ git add * 
```

4.提交代码

```shell
$ git commit -m 'myblog init'
```

5.查看是否配置生效

```shell
$ git remote -v
origin    https://github.com/qianjiangtao/spring-cloud-summary.git (fetch)
origin    https://github.com/qianjiangtao/spring-cloud-summary.git (push)
```

6.将代码提交到远程对应分支

```shell
# git push -u <本地分支名>:<远程分支名>
$ git push -u origin master
```

7.此时会报错，是因为远程有`read.md`文件，需要合并冲突

```shell
 ! [rejected]        master -> master (non-fast-forward)
error: failed to push some refs to 'https://github.com/qianjiangtao/spring-cloud-summary.git'
hint: Updates were rejected because the tip of your current branch is behind
hint: its remote counterpart. Integrate the remote changes (e.g.
hint: 'git pull ...') before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.

```

8.需要先将远程仓库中的内容拉下来,再执行merge命令 

```shell
$ git fetch origin master 
#Git 2.9及以上的版本中，merge和pull的命令将不允许两个不相关历史的分支进行合并，除非加上–allow-
#unrelated-histories，否则会报fatal: refusing to merge unrelated histories
$ git merge origin/master --allow-unrelated-histories
```

9.成功推送到远程

```
Counting objects: 6, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (6/6), done.
Writing objects: 100% (6/6), 1.20 KiB | 0 bytes/s, done.
Total 6 (delta 0), reused 0 (delta 0)
To https://github.com/qianjiangtao/spring-cloud-summary.git
   6f298ed..536bd89  master -> master
Branch master set up to track remote branch master from origin.
```

