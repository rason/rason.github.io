title: 【Django现学现卖2】将项目托管到github远程仓库上
categories: Django
date: 2016-03-16 16:49:25
tags: Django
description: 项目托管到github
---

## 前言

在[Django现学现卖1](http://rason.me/2016/03/03/Deploy-django-on-aliyun/)中，为了简单起见，我直接在阿里云服务器上搭建了一个Helloworld应用。但是我不可能直接在服务器上写代码啊，那样太不方便了，所以就决定把项目托管到[github](https://github.com/rason/HelloPython)上，然后就可以用自己的本地机器`git clone`下来愉快地写代码了。好了，下面开始介绍怎么把项目托管到github上。

## 步骤

**1.安装git**
 
 新购买的阿里云服务器没有git，所有首先得安装git。

`sudo apt-get install git-all`

**2.初始化仓库**

我们进入教程一中创建的工程文件夹，将该文件夹作为仓库。

`cd ebookloader/`

`git init`

初始完之后，我们使用`ls -a`命令查看一下，发现`.git`文件夹，此文件夹就为我们保存了版本控制的信息。

```
root@iZ94hd1izoaZ:/ebookloader# ls -a
.  ..  booksystem  ebookloader  .git  manage.py

```

**3.将工程加入版本控制**

我们上一步初始化的仓库，默认是没有把`ebookloader`文件夹中的内容加入到版本控制中的，所以执行

`git add .`

把文件夹中的所有内容暂存起来，但是此时内容还没有真正保存到仓库中，我们可以使用

`git status`

查看状态，结果如下

<!-- more -->

```
On branch master

Initial commit

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)

	new file:   booksystem/__init__.py
	new file:   booksystem/admin.py
	new file:   booksystem/apps.py
	new file:   booksystem/migrations/__init__.py
	new file:   booksystem/models.py
	new file:   booksystem/tests.py
	new file:   booksystem/views.py
	new file:   ebookloader/__init__.py
	new file:   ebookloader/__init__.pyc
	new file:   ebookloader/settings.py
	new file:   ebookloader/settings.pyc
	new file:   ebookloader/urls.py
	new file:   ebookloader/views.py
	new file:   ebookloader/wsgi.py
	new file:   manage.py

```
此时，git提示我们需要commit。所以，我们执行一下

`git commit -m "init project"`

操作结果如下

```
*** Please tell me who you are.

Run

  git config --global user.email "you@example.com"
  git config --global user.name "Your Name"

```
原来我们新安装的git还没有配置邮箱和名称，没有配置的话就无法跟踪代码是谁提交的了，多人协助的时候就没法辨认了，这显然是我们不希望发生的，所以我们默认要配置邮箱和名称。

**4.git配置**

根据上面的提示，我们执行以下命令

`git config --global user.email "rason2008@gmail.com`

`git config --global user.name "rason`

当然，用户名和名称要替换成你自己的。

配置完之后我们再次执行`git commit -m "init project"`,结果如下

```
[master (root-commit) 002e6cc] init project
 15 files changed, 200 insertions(+)
```
这就代表我们提交到仓库成功了。当然，这只是提交到本地仓库，还没有提交到远程的github仓库中。

**5.在github上新建一个仓库**
 
要将代码托管在github中，我们需要在[github](https://github.com/)上点击`New repository`按钮新建一个仓库。

我的将其命名为`HelloPython`,通过SSH链接的地址是`git@github.com:rason/HelloPython.git`。


**6.添加远程仓库**

我们先执行一下`git remote -v`查看远程仓库信息，发现什么都没输出。因为我们现在还没有添加任何远程仓库，所以是空白的。

然后执行

`git remote add origin git@github.com:rason/HelloPython.git`

把我们上一步创建的仓库作为我们的远程仓库，因为我们需要把代码托管到这个仓库中。此命令的格式为：

```
$ git remote add <主机> <网址>
```

此时，我们再执行一下`git remote -v`会看到以下信息

```
origin	git@github.com:rason/HelloPython.git (fetch)
origin	git@github.com:rason/HelloPython.git (push)
```
说明我们添加远程仓库成功。


**7.将代码推送到远程仓库**

添加完远程仓库之后，我们就可以直接把代码推送到远程仓库中了。执行

`git push -u origin master`

然而，结果好像是没成功，输出以下信息

```
The authenticity of host 'github.com (192.30.252.129)' can't be established.
RSA key fingerprint is 16:27:ac:a5:76:28:2d:36:63:1b:56:4d:eb:df:a6:48.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'github.com,192.30.252.129' (RSA) to the list of known hosts.
Permission denied (publickey).
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```
根据提示可以看出，是因为没有权限而被拒绝了。的确，因为github根本不知道我是谁啊？那么该怎么办？

**8.在github上配置SSH keys**

为此我们需要生成一对公私密钥，并将公钥配置在github的`setting>SSH keys`中。我们可以使用`ls -al ~/.ssh`查看机器中有没有密钥，因为是新购的服务器，显然是没有的。

执行

`ssh-keygen -t rsa -b 4096 -C "rason2008@gmail.com"`

生成密钥对。

再执行

`ssh-add ~/.ssh/id_rsa`

将密钥添加到ssh中。

生成之后使用`vi ~/.ssh/id_rsa.pub `命令打开公钥文件，拷贝其中的内容，在[github settings](https://github.com/settings/keys)中的SSH keys栏目中点击`New SSH key`,将内容粘贴进去，点击保存即可。

关于SSH keys的完整操作过程可见[官方教程](https://help.github.com/articles/generating-an-ssh-key/)。

**9.重新将代码推送到远程仓库**

配置完毕之后，我们再次执行

`git push -u origin master`

将代码推送到远程仓库，输出以下信息

```
root@iZ94hd1izoaZ:/ebookloader# git push -u origin master
Warning: Permanently added the RSA host key for IP address '192.30.252.130' to the list of known hosts.
Counting objects: 18, done.
Compressing objects: 100% (16/16), done.
Writing objects: 100% (18/18), 4.67 KiB | 0 bytes/s, done.
Total 18 (delta 0), reused 0 (delta 0)
To git@github.com:rason/HelloPython.git
 * [new branch]      master -> master
Branch master set up to track remote branch master from origin.
```

这说明我们的推送成功了。在浏览器中刷新一下[仓库地址](https://github.com/rason/HelloPython)，即可以看到提交上来的代码了。

此处，`git push`命令的格式为：

```
$ git push <远程主机名> <本地分支名>:<远程分支名>
```

**10.将远程仓库的代码克隆到本地机器**

好了，通过以上的操作，我们已经将阿里云服务器中的工程代码提交到github上了。现在只要我们把github的仓库克隆到自己的电脑，即可以在自己的电脑上进行开发工作了。

在自己的机器终端上，使用

`git clone git@github.com:rason/HelloPython.git`

即可把代码克隆下来。然后愉快地进行开发，开发完把代码提交到远程仓库，然后在服务器上将新提交的代码拉取下来，重启Apache服务就可以看到新提交的功能了。

## 总结

本篇文章主要是温习一下git的简单使用，将工程代码托管在github上，以便进行版本控制。当然自己一个人开发的话，可以不用版本控制，直接使用工具将本机的代码上传到阿里云服务器中即可。

