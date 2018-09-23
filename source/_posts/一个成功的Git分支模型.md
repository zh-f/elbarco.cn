---
title: 一个成功的Git分支模型
date: 2016-03-08 10:29:22
tags: Git
---

## 前言

从大公司跳到了小团队，版本控制软件从Git换到了SVN，然而前段时间，头儿让我研究下如何搭建私有Git服务器，如何“优雅”地使用Git。

针对如何搭建私有Git服务器，我选用的是[GitLab](https://about.gitlab.com/)，有一键安装包，也有很多Step by Step的教程，可自行Google。本文就上面提出的后两个问题，参考文章[《A successful Git branching model》](http://nvie.com/posts/a-successful-git-branching-model/)讲述如何合理的使用Git branch进行开发和版本管理。

<!--more-->

先来张图：
![分支模型全貌](http://bop-to.top/git-model%402x.png)


## 详细展开

### 主要分支

在这个模型中，中央仓库持有两个生命周期无限长的主要分支：

* `master`
* `develop`

![主要分支](http://bop-to.top/main-branches%402x.png)

我们认为，`origin/master`这个主要分支上，源码的`HEAD`永远保持生产就绪的状态。`origin/develop`这个主要分支的源码`HEAD`则永远代表了最新提交的开发变更，所以也被称为是“集成分支”。该分支可以用于每晚的自动化构建所使用。

当`develop`分支的代码能够到达一个稳定点，并且已经准备好进行版本发布，所以的变更应当合并到`master`上，并且用版本号标注。具体操作后详细谈到。

因此，每当变更最终合并到`master`分支，这就是一个新的生产版本。对待这个分支，要极其严格，所以理论上来讲，可以使用一个Git hook脚本来进行自动化构建，每当有新内容提交到`master`，脚本自动将软件发布到成产环境。


### 支持性分支

在这个模型中，有各类支持性分支来协助团队成员的并行开发，方便跟踪功能特性，准备生产版本和快速修复生产问题。与主要分支不同的是，这三个支持性分支是有有限生命周期的，最终会被移除。

这里使用的三类分支分别是：

* 功能特性分支（`Feature branches`）
* 发布用分支（`Release branches`）
* 补丁分支（`Hotfix branches`）

这三类分支目的明确，所以对于这些分支的源分支和合并的目标分支具有十分严格的规则。当然，这三类分支也仅仅是分支而已，并没有特别的地方。

#### 功能特性分支

> 
分支来源：`develop`
合并目标：`develop`
命名惯例：除`master`、`develop`、`release-*`或者`hotfix-*`之外的任何名字均可

![](http://bop-to.top/fb%402x.png)

功能特性分支（或者有时被称作专题分支）被用于开发接下来或者将来版本的新功能、新特性。当开始开发一项功能时，目标发布用分支并未明确，但只要功能在开发中，这个分支就存在，最终会合并回`develop`（意味着即将发布的版本中一定会包含该功能）或者被废弃（这当然是一种令人十分失望的情况）。

功能特性分支仅存在与开发的代码仓库，并不在`origin`。

*创建一个功能特性分支*

当着手开发新功能时，先在开发分支上检出新分支：

```bash
$ git checkout -b myfeature develop
Switched to a new branch "myfeature"
```

*将完成的功能合并到开发分支上*

完成的功能特性被合并到`develop`分支上，表示该功能要添加到即将发布的版本中：

```bash
$ git checkout develop
Switched to branch 'develop'
$ git merge --no-ff myfeature
Updating ea1b82a..05e9557
(Summary of changes)
$ git branch -d myfeature
Deleted branch myfeature (was 05e9557).
$ git push origin develop
```
`--no-ff`表示合并总是创建新的提交对象，这样可以避免在合并分支时丢失历史信息，对比图如下：

![](http://bop-to.top/merge-without-ff%402x.png)

显而易见，这就是证据啊，证据！:joy:

#### 发布用分支

> 
分支来源：`develop`
合并目标：`develop`和`master`
命名惯例：`release-*`

发布用分支用于支持生产环境新版本，如修改少数的缺陷，准备版本发布的元数据（如版本号，构建日期等）。做完这些操作之后，`develop`分支便可以为了下个大版本接受这些新功能了。

将发布用分支从`develop`分支上检出的关键时刻是在开发几乎完全可以反映新功能理想状态的时候。此时，至少下个版本要发布的功能所在的功能分支要合并到`develop`上，而功能发布在将来的版本中则可以暂时不合并，等待下一次发布用分支的检出。

在发布用分支拉出时，就需要给其分配一个版本号。而此后的`develop`分支上的变更都将反映这个版本。

*创建一个发布用分支*

发布用分支在`develop`分支上检出。举例来讲，目前我们的生产环境版本是1.1.5，马上就要发布一个大版本。`develop`分支已经准备就绪，我们决定将下一个版本的版本号为1.2.所以我们拉出一个发布用分支，命名需要反映新的版本号：

```bash
$ git checkout -b release-1.2 develop
Switched to a new branch "release-1.2"
$ ./bump-version.sh 1.2
Files modified successfully, version bumped to 1.2.
$ git commit -a -m "Bumped version number to 1.2"
[release-1.2 74d9424] Bumped version number to 1.2
1 files changed, 1 insertions(+), 1 deletions(-)

```

创建完新分支之后，变更版本号（这里的[`bump-version.sh`](https://gist.github.com/pete-otaqui/4188238)脚本用于修改文件版本号，当然，针对不同的场景，也可手动变更版本号）。

该分支会存在一段时间，这段时间内，该分支允许修改缺陷（而不是在`develop`上面）。在该分支上禁止添加新特性。最终，该分支必须合并到`develop`、

*完成一个发布用分支*

当发布用分支已经准备就绪可以发布一个现实的版本，我们仍然有很多工作要做。首先，将发布用分支合并到`master`（切记，所以提交到`master`内容一定是一新版本）。接着，提交到`master`上的变更必须添加标记（如使用版本号等进行标记），用于将来参考。最后，在这个发布用分支上进行的更改需要合并回`develop`，以保证将来的版本包含缺陷的修复。

在Git中的前两步：

```bash
$ git checkout master
Switched to branch 'master'
$ git merge --no-ff release-1.2
Merge made by recursive.
(Summary of changes)
$ git tag -a 1.2
```
此时，版本已发布，并且已标记。

>Tips: 你可以使用`-s`或者`-u <key>`来加密标记。

为了保留发布用分支的变更，需要合并回`develop`分支：

```bash
$ git checkout develop
Switched to branch 'develop'
$ git merge --no-ff release-1.2
Merge made by recursive.
(Summary of changes)
```
这一步可能也会产生冲突，所以，解决冲突并且提交。

此时，我们可以移除该发布用分支：

```bash
$ git branch -d release-1.2
Deleted branch release-1.2 (was ff452fe)

```

#### 补丁分支

> 
分支来源：`master`
合并目标：`develop`和`master`
命名惯例：`hotfix-*`

![Hoxfix branches](http://bop-to.top/hotfix-branches%402x.png)

这类分支与发布用分支很类似，不过补丁分支的产生是为了快速响应生产环境中的紧急问题。当线上遭遇紧急缺陷需要立刻解决，则需要在对应标记的`master`分支上拉出一个补丁分支。

在某一位或者几位开发者修复线上问题的同时，`develop`分支可以继续进行。

*创建一个补丁分支*

补丁分支在`master`上拉出，举例来说，1.2版本是目前的线上版本，由于一个严重bug造成宕机的情况出现，但是目前`develop`分支上的变更还不够稳定，此时，我们可以使用补丁分支，先来解决紧急问题：

```bash
$ git checkout -b hotfix-1.2.1 master
Switched to a new branch "hotfix-1.2.1"
$ ./bump-version.sh 1.2.1
Files modified successfully, version bumped to 1.2.1.
$ git commit -a -m "Bumped version number to 1.2.1"
[hotfix-1.2.1 41e61bb] Bumped version number to 1.2.1
1 files changed, 1 insertions(+), 1 deletions(-)
```

不要忘记增加版本号。

然后，修复bug并提交变更。

```bash
$ git commit -m "Fixed severe production problem"
[hotfix-1.2.1 abbe5d6] Fixed severe production problem
5 files changed, 32 insertions(+), 17 deletions(-)
```

*结束使用一个补丁分支*

修复bug之后，补丁分支必须合并到`master`，同时，也需要合并到`develop`，确保在下一个版本中包含bug的修复。此时的操作与发布用分支完全一致。

首先，更新`master`并且标注版本：

```bash
$ git checkout master
Switched to branch 'master'
$ git merge --no-ff hotfix-1.2.1
Merge made by recursive.
(Summary of changes)
$ git tag -a 1.2.1
```

接着，合并到`develop`：

```bash
$ git checkout develop
Switched to branch 'develop'
$ git merge --no-ff hotfix-1.2.1
Merge made by recursive.
(Summary of changes)
```

**这个规则存在一个例外情况：如果发布用分支当前存在，则需要将补丁分支合并到发布用分支，而不是`develop`**，因为该发布用分支最终会合并到`develop`（如果`develop`分支立刻需要这个bug得到修复，而等不到发布用分支结束，则你需要小心谨慎的将修正合并到未准备就绪的`develop`分支上）。

最后，移除这个临时分支：

```bash
$ git branch -d hotfix-1.2.1
Deleted branch hotfix-1.2.1 (was abbe5d6).
```

## 结语

这个模型看起来并没有什么特别令人震惊的地方，但是却十分合理。在StackOverflow上问题[What does Bump Version stand for?](http://stackoverflow.com/questions/4181185/what-does-bump-version-stand-for)中，有解答者提到了该文，并十分推荐。


>原文作者Twitter：[@nvie](http://twitter.com/nvie) 
















