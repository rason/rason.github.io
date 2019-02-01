title: git远程仓库
categories: Git
date: 2015-08-25 08:55:26
tags: git远程仓库
description: git git远程仓库 git命令
---

**为了协作开发，我们需要使用git远程仓库，下面总结一下常用的git远程命令。**

## 查看远程仓库

```
$ git remote -v
origin  git@github.com:rason/rason.github.io.git (fetch)
origin  git@github.com:rason/rason.github.io.git (push)
```

当我们使用`git clone`了一个远程的仓库之后，使用`git remote`命令就可以看到一个origin的远程仓库。当然我们也可以加上参数\-v查看更加详细的信息，以上命令可以看到我的机器中有一个origin的远程仓库。

## 添加远程仓库

```
$ git remote add react https://github.com/facebook/react.git

$ git remote -v
react   https://github.com/facebook/react.git (fetch)
react   https://github.com/facebook/react.git (push)
```

使用`git remote add`命令添加远程仓库，以上命令中我添加了一个最近比较火的前端框架react的远程仓库。以上仅仅是添加了一个远程的仓库，我们可以使用git fetch命令将远程仓库中的信息拉取下来。

<!--more-->

## 从远程仓库中抓取与拉取

```
$ git fetch react
remote: Counting objects: 53419, done.
remote: Total 53419 (delta 0), reused 0 (delta 0), pack-reused 53419
Receiving objects: 100% (53419/53419), 62.27 MiB | 804.00 KiB/s, done.
Resolving deltas: 100% (34093/34093), done.
```

这个命令会访问远程仓库，从中拉取所有你还没有的数据。 执行完成后，你将会拥有那个远程仓库中所有分支的引用，可以随时合并或查看。我们可以使用 `git branch -r`来查看刚拉取的远程分支信息。

```
$ git branch -r
  react/0.10-stable
  react/0.11-stable
  react/0.12-stable
  react/0.13-stable
  react/0.3-stable
  react/0.4-stable
  react/0.5-stable
  react/0.8-stable
  react/0.9-stable
  react/gh-pages
  react/master
```

可以看到我们已经把远程仓库的分支信息拉取下来了，现在我们在本地通过react/master访问到远程仓库react的master分支了。但是现在查看本地的文件夹却看不到拉取的远程分支内容，因为我们还没有检出一个本地分支。

```
$ git checkout -b newBranch react/master
Branch newBranch set up to track remote branch master from react.
Switched to a new branch 'newBranch'
```

使用`git checkout`命令检出一个本地分支，上面命令表示，在react/master的基础上，检出一个新的本地分支，这个从远程分支 checkout 出来的本地分支，称为_跟踪分支(tracking branch)。

> 如果你使用 clone 命令克隆了一个仓库，命令会自动将其添加为远程仓库并默认以 “origin” 为简写。 所以，git fetch origin 会抓取克隆（或上一次抓取）后新推送的所有工作。 必须注意 git fetch 命令会将数据拉取到你的本地仓库 - 它并不会自动合并或修改你当前的工作。 当准备好时你必须手动将其合并入你的工作。

> 如果你有一个分支设置为跟踪一个远程分支，可以使用 git pull 命令来自动的抓取然后合并远程分支到当前分支。 这对你来说可能是一个更简单或更舒服的工作流程；默认情况下，git clone 命令会自动设置本地 master 分支跟踪克隆的远程仓库的 master 分支（或不管是什么名字的默认分支）。 运行 git pull 通常会从最初克隆的服务器上抓取数据并自动尝试合并到当前所在的分支。

## 推送到远程仓库

当我们撸完了一个功能，可以使用以下命令将代码推送到远程仓库。

```
git push [remote-name] [branch-name]
```

当你具有克隆服务器的读写权限，并且之前没有其他小伙伴推送内容到远程仓库，这条命令才能生效。如果有小伙伴比你先提交，你必须先把内容先拉取下来合并之后才能提交。

## 查看远程仓库更多信息

如果想要查看某一个远程仓库的更多信息，可以使用 `git remote show [remote-name]` 命令。

```
 git remote show [remote-name]
```

运行上面的命令，可以得到以下信息

```
$ git remote show react
* remote react
  Fetch URL: https://github.com/facebook/react.git
  Push  URL: https://github.com/facebook/react.git
  HEAD branch: master
  Remote branches:
    0.10-stable tracked
    0.11-stable tracked
    0.12-stable tracked
    0.13-stable tracked
    0.3-stable  tracked
    0.4-stable  tracked
    0.5-stable  tracked
    0.8-stable  tracked
    0.9-stable  tracked
    gh-pages    tracked
    master      tracked
  Local branch configured for 'git pull':
    newBranch merges with remote master
```

这些信息告诉我们正在处于master分支，并且列出了远程分支的URL，远程分支的信息。另外还告诉我们`git pull`命令可以将远程分支master合并到本地分支newBranch。如果你具有服务器的写权限还会显示我们可以使用`git push`命令将哪些分支推送到服务器对应的哪些分支上，哪些远程分支不在你的本地，哪些远程分支已经从服务器上移除了。

## 远程仓库的移除与重命名

将远程仓库react重命名为ra，使用以下命令：

```
$ git remote rename react ra
```

远程仓库的代码搬走了，或者你不再向远程仓库贡献代码了，则可以使用以下命令移除远程仓库

```
$ git remote rm react
```

** 以上为Git-book学习笔记，详情请参考[Git-book](http://git-scm.com/book/zh/v2/Git-%E5%9F%BA%E7%A1%80-%E8%BF%9C%E7%A8%8B%E4%BB%93%E5%BA%93%E7%9A%84%E4%BD%BF%E7%94%A8)。**
