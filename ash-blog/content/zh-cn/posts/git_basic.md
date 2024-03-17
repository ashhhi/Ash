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



