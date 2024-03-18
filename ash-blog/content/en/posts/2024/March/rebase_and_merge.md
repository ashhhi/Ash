---
title: "git rebase 和 git merge的区别"
date: 2024-03-18
lastmod: 2024-03-18
draft: false
tags: ["Git"]
categories: ["Note"]
authors:
  - "Ash"
---

# git rebase 和 git merge的区别

merge 和 rebase 操作均用于合并分支，但在具体操作上有不同，同时，rebase会给你一套清晰的代码历史，而merge则是乱七八糟的代码历史. 下面我们分别说说 merge 和 rebase 的具体操作

## rebase

假设你有两个分支，一个master主分支，一个从master分离出来的feature分支，如下所示

```shell
       <- E <- F   feature 分支
     /
A <- B <- C <- D   master 分支
```

当使用 merge 的时候，比如在feature分支上使用 "git rebase master", git 会先找到两个分支的最近公共祖先，在这里也就是B，然后对比feature分支相比于B的历次commit，也就是E和F，提取相应的修改并存为临时文件，再将header指向master分支的最后一次提交，也就是D，以此作为新的基底将之前存起来的的临时文件按顺序应用

```shell
A <- B <- C <- D <- E‘ <- F’  feature 分支
```

也可以理解成 rebase 是直接将以B的作为基底切换到了以D作为基底，并提交了两个内容相同的commit（E和F），但实际上E和F已经不再是以前的E和F，变成了E‘和F’，他们是在master分支commit C和D之后再进行的提交，因此实际上 rebase 操作是丢弃了一些原有的commit（E和F），而新提交一些内容一样但实际上不同的commit（E‘和F’）

使用 rebase 可以使得代码历史简介有线性，例如这次操作的历史变为：

```shell
# 最上面是最新的
* 74199ce (HEAD -> master, feature) commit F
* e7c7111 commit E
* d9623b0 commit D
* 73deeed commit C
* c50221f commit B
* ef13725 commit A
```

## merge

以同样的例子举例，使用merge操作生成的代码历史显得比较复杂
- 切换到 feature 分支： git checkout feature。
- 合并 master 分支的更新： git merge master。

``` shell
*   875906b Merge branch 'master' into feature
|\  
| | 5b05585 commit F
| | f5b0fc0 commit E
* * d017dff commit D
* * 9df916f commit C
|/  
* cb932a6 commit B
* sd8fd7f commit A

```
也就是

```shell
       <- E <- F   
     /           \
A <- B <-  C  <-  D 
```

可以看到 merge 会形成一个新的节点，之前的两个分支提交都分开显示，而 rebase 不会生成新的节点，是将两个分支融合成一个线性操作.
