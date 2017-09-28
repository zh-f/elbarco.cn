---
title: Git中使用rebase命令更新本地分支
date: 2017-09-28 14:34:49
tags: [Git, rebase]
---

最近在开发`trove`中，因为误提交，本地项目的`devel`分支已经与上游的的`devel`分支不一致了<!--more-->。为了更好的创建分支，或者后面进行`cherry-pick`准备打包，都需要将本地的分支与上游的分支做一下rebase。

> 注：上游指的是[`eayunstack/trove`](https://github.com/eayunstack/trove)，本地指的是[`2hf/trove`](https://github.com/2hf/trove)

首先，我们要添加`upstream`远程仓库：
```
$ git remote
origin
$ git remote add upstream git@github.com:eayunstack/trove.git
$ git remote -v
origin  git@github.com:2hf/trove.git (fetch)
origin  git@github.com:2hf/trove.git (push)
upstream        git@github.com:eayunstack/trove.git (fetch)
upstream        git@github.com:eayunstack/trove.git (push)

```

然后更新`upstream`：
```
$ git fetch upstream
```

此时远程仓库已经准备就绪了，这时候我们就可以rebase本地的分支了。两种做法：
```
# option one
$ git checkout devel
$ git rebase -i upstream/devel

# option two
$ git checkout devel
$ git reset --hard upstream/devel
```

> 注：如果之前`2hf/devel`分支已经做过push了，为了保持与上游一致，需要`git push -f`。


