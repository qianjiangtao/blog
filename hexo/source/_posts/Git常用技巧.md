---
title: Git常用技巧
date: 2018-08-22 13:50:39
categories:
-Git
tags:
-Git

---

####一.本地代码更新冲突



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



####二.如何将本地创建的项目关联到远程仓库并推送到远程

