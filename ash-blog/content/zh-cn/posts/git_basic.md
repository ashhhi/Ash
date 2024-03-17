---
title: "Git笔记"
date: 2024-03-17
lastmod: 2024-03-17
draft: false
tags: ["Git"]
categories: ["Note"]
authors:
  - "Ash"
---

# Git 笔记

## 清除git缓存，使修改后的gitignore生效
```shell
git rm -r --cached .
git add .
git commit -m 'update .gitignore'
```

## 查看commit记录
```shell
git log 查看所有
git show 查看最新的commit  
git show commitId 查看指定commit hashID的所有修改 
git show commitId fileName 查看某次commit中具体某个文件的修改
```



