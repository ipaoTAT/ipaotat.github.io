---
layout: post
title: Git查看/删除远程分支
comments: true
category: 工具
tags: [Git]
---

[toc]
##查看远程分支
加上-a参数可以查看远程分支，远程分支会用红色表示出来（如果你开了颜色支持的话）：

```python
$ git branch -a
  master
* v1.8
  remotes/origin/master
  remotes/origin/v1.0
  remotes/origin/v1.1
  remotes/origin/v1.3
  remotes/origin/v1.5
  remotes/origin/v1.6
  remotes/origin/v1.7
```

##删除远程分支

git 1.7以上可以直接删除分支：

```python
$ git push origin --delete <branchName>
```

之前版本则向remote分支push一个空本地分支

```python
$ git push <remote-hostname> <local-branch>:<remote-branch>
$ git push origin  :<branchName>
```
<!-- more -->
##删除不存在对应远程分支的本地分支

假设这样一种情况：

我创建了本地分支b1并pull到远程分支 origin/b1；
其他人在本地使用fetch或pull创建了本地的b1分支；
我删除了 origin/b1 远程分支；
其他人再次执行fetch或者pull并不会删除这个他们本地的 b1 分支，运行 git branch -a 也不能看出这个branch被删除了，如何处理？
使用下面的代码查看b1的状态：

```python
$ git remote show origin
* remote origin
  Fetch URL: git@github.com:xxx/xxx.git
  Push  URL: git@github.com:xxx/xxx.git
  HEAD branch: master
  Remote branches:
    master                 tracked
    refs/remotes/origin/b1 stale (use 'git remote prune' to remove)
  Local branch configured for 'git pull':
    master merges with remote master
  Local ref configured for 'git push':
    master pushes to master (up to date)
```

这时候能够看到b1是stale的，使用 git remote prune origin 可以将其从本地版本库中去除。

更简单的方法是使用这个命令，它在fetch之后删除掉没有与远程分支对应的本地分支：

```python
$ git fetch -p
```
