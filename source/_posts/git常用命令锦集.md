---
title: Git 常用命令锦集
date: 2017-6-25 20:17:16
tags: git
---
记录一些工作中常用的命令

### 查看当前工作目录状态 & 查看修改了哪些内容

``` bash
git status
git diff
```
###  丢弃当前目录所有的修改

``` bash
git checkout .
```

### 添加当前目录 & 提交

``` bash
git add .
git commit -m "这是新提交的文件"
```

### 提交代码到远程新的分支 & 删除远程分支

``` bash
git push -u origin hotfix/20180503
git push origin -d hotfix/20180301
```
### 从当前分支创建新的分支并

``` bash
git checkout -b hotfix/20180503
```

### 暂存修改的文件 & 释放暂存修改的文件
本地修改了文件，但是需要切换分支或者拉取最新代码时
``` bash
git stash 
git stash pop
git pull 
```
### 丢弃上一次的提交 & 回到某次提交

``` bash
git reset HEAD~1
git reset 83f1901
```

###  查看分支提交记录

``` bash
git log
```
###  查看某次提交的内容

``` bash
git log
git show 83f1901
```

###  查看本地所有的分支 & 切换某个分支

``` bash
git branch -a
git checkout hotfix/20180301
```
###  删除本地多余的分支 & 将本地的分支与远程保持一致

``` bash
git branch -d hotfix/20180301
git fetch -p
```

###  在当前分支合并某个特征分支

``` bash
git merge my-page
```
###  强制提交

``` bash
git push -f origin master
```
###  查看当前分支下的tag & 切换到tag

``` bash
git tag
git checkout v20190121

```
###  创建tag

``` bash
git tag -a V1.2 -m 'release 1.2'
```