title: git分支管理模型
categories: Git
date: 2016-12-22 15:16:02
tags: git分支管理
description: git分支管理
---

## 前言

git之所以强大就是因为它对分支的管理非常简便，鼓励我们开发任何功能点都应该先建立一个分支，甚至是修改一个极其微小的Bug，因为建立分支的成本实在是太低了。今天我们来学习一下如何利用git分支来进行开发工作，前人已经总结出了一个相对成熟的分支管理模型，如下图：

![git-branching-mode](http://nvie.com/img/git-model@2x.png)

<!-- more -->

## 主分支

![主分支](http://nvie.com/img/main-branches@2x.png)

- master和develop是主分支，这两个分支存在于整个项目的开发周期，除非项目挂了或公司倒闭了，一般不会删除这两个分支。

- 这两个分支一般来说会控制权限，只有管理员才能进行merge和push操作，普通开发者只能往其它分支(比如特性分支)提交代码，然后发起`merge request`请求将其它分支的代码合并到develop分支中。

- master分支是一个稳定、无Bug的分支，线上系统都应该是根据master分支进行发布的，在下一个发布代码提交之前，master分支的代码应该是跟线上代码一样才对，只有在准备发布生产环境时才将develop(测试通过后)的代码merge到master中。

- develop是开发分支，在进行特性开发之前，都是以develop分支为基础检出一个特性分支来进行开发。多个特性开发同时进行则以develop分支为基础检出多个特性分支。也就是说，develop分支是开发的基础分支，特性功能开发完之后提交merge request请求将特性功能合并到develop分支中进行测试环境测试。

## 辅助分支

- Feature branches（特性分支）
- Release branches（发布分支）
- Hotfix branches（紧急修复分支）

以上三个分支称为辅助分支，利用完之后一般会删除掉。

### Feature branches

在开发一个新功能特性的时候，我们就会以develop分支为基础检出一个特性分支，开发完之后将代码合并回develop分支，没问题的话这个特性分支就可以删除了。

![Feature branches](http://nvie.com/img/fb@2x.png)

一般工作流程：

- 检出一个特性分支：`git checkout -b myfeature develop`
- 进行开发
- 将代码推送到远程仓库对应的特性分支：`git push origin myfeature`
- 在gitlab中提交merge request请求合并到develop分支中（此处以gitlab为例）

### Release branches

在准备发布生产环境时，管理员会已develop分支为基础检出一个发布分支，注意这个操作一般是由管理员来进行的。发布分支可以用来添加一些版本信息或者进行一些小修改。如果准备发布时开发人员发现还有一点小Bug，就可以已发布分支为基础检出一个新的分支修复问题，修复完之后提交merge request请求合并回release分支，然后管理员准备发布工作，将release分支的代码分别合并到develop分支和master分支，然后将master分支的代码发布生产。

一般工作流程：

- 检出一个发布分支：` git checkout -b release-1.2 develop`
- 修改版本信息
- 合并回master分支，打tag
- 合并回develop分支
- 删除release分支

### Hotfix branches

当生产发现问题时，我们就可以已master分支为基础检出一个Hotfix分支，因为master分支的代码跟线上代码是一致的，develop分支的代码可能已经合并了一些正在进行的特性功能代码在测试。然后开发人员在Hotfix分支进行Bug修复，修复完之后由管理员将Hotfix分支的代码合并回master分支和develop分支，这个过程跟release分支合并类似。

一般工作流程：

- 检出一个Hotfix分支：`git checkout -b hotfix-1.2.1 master`
- 修改bug
- 修改版本信息
- 合并回master分支，打tag
- 合并回develop分支
- 删除Hotfix分支

## git-flow

看完上面分支管理模型，用惯SVN的人一定会觉得异常麻烦，修复一个小Bug也需要进行一系列操作：检出分支，提交代码，合并分支，删除分支等。`git-flow`就是将这些复杂的操作合并起来的命令集，关于`git-flow`的学习可以参考[git-flow备忘清单](http://danielkummer.github.io/git-flow-cheatsheet/index.zh_CN.html)和它的[源码](https://github.com/nvie/gitflow)，此处就不展开学习了。

## 参考资料

- [A successful Git branching model](http://nvie.com/posts/a-successful-git-branching-model)
- [基于git-flow的团队协作](https://zhangmengpl.gitbooks.io/gitlab-guide/content/whatisgitflow.html)
- [git-flow源码](https://github.com/nvie/gitflow)
- [git-flow工作流程](https://www.git-tower.com/learn/git/ebook/cn/command-line/advanced-topics/git-flow#start)
- [git-flow备忘清单](http://danielkummer.github.io/git-flow-cheatsheet/index.zh_CN.html)
