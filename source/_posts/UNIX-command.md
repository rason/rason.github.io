title: Linux常用命令
categories: Linux
date: 2015-10-14 16:01:48
tags: Linux命令
description: Linux常用命令
---

简单记录一下自己最常用到的几个命令

## vi

**注意：以下命令大小写敏感**

### 进入离开vi

| 命令 | 作用 |
| :-----------: | :-------: |
| vi filename | 打开或新建文件 |
| i | 进入编辑模式 |
| Esc | 退出编辑模式 |
| EscEsc | 不清楚当前处于哪种模式，回到指令模式 |
| :q | 没有修改退出 |
| :q! | 不保存退出 |
| :wq | 保存退出 |
| ZZ | 保存退出|
| :set nu | 在左侧显示行号|

<!-- more -->

### 光标移动

| 命令 | 作用 |
| :-----------: | :-------: |
| k | 向上移动一行 |
| j| 向下移动一行 |
| h | 光标左移一个字符 |
| l | 光标右移一个字符 |
| G | 光标移到最后一行 |
| 1G | 光标移到第一行 |
| nG | 光标移到第n行 |
| :n | 光标移到第n行|
| 0 或 l | 光标回到行首 |
| $ | 光标回到行末 |
| nG | 光标移到第n行 |
| :n | 光标移到第n行|

### 编辑文本

| 命令 | 作用 |
| :-----------: | :-------: |
| i | 当前位置前插入文本 |
| I | 当前行首插入文本 |
| a | 当前位置后插入文本 |
| A | 当前行末尾插入文本 |
| o | 在光标位置下方新建一行来输入文本 |
| x | 删除光标位置下的字符 |
| dd | 删除一整行 |
| yy | 复制当前行|
| yw | 复制当前字 |
| p | 在光标后粘贴 |
| P | 在光标前粘贴 |

### 搜索文本

| 命令 | 作用 |
| :-----------: | :-------: |
| / | 从上往下搜索 |
| ？ | 从下往上搜索 |

搜索命令后跟正则表达式，输入n 和 N 指令用于以同样或相反的方向重复上述搜索指令，在搜索指令中，一些字符串拥有特殊的意义，就需要使用转意符（\）。

## ftp

登录ftp服务器：

```
$ftp hostname or ip-address
```

操作命令:

| 命令 | 作用 |
| :-----------: | :-------: |
| put filename	| 从本地往远程服务器上传文件 | 
| get filename	| 从远程服务器往本地下载文件 | 
| mput file list	| 从本地往远程服务器批量上传文件 | 
| mget file list	| 从远程服务器往本地批量下载文件 | 
| prompt off	| 关闭文件提醒,在 mput 与 mget 时不会每操作一个文件就询问一次。| 
| prompt on	| 开启文件提醒 | 
| dir	| 列出远程服务器上当前目录下的所有文件 | 
| cd dirname	| 切换本地主机上的目录到指定目录下 | 
| lcd dirname	| 切换远程服务器上的目录到指定目录下 | 
| quit	| 注销当前登陆 | 

需要注意的是，上传和下载文件时的本地主机目录都是当前目录。如果用户希望上传或下载文件的目录为特定的目录，那么用户需要先将当前目录切换到指定目录后再进行上传或下载操作。

## ssh

登录远程主机：
```
ssh user@ip
```

## 进程

| 命令 | 作用 |
| :-----------: | :-------: |
| ps | 显示当前处于执行状态的进程 | 
| ps -f	| 同上，显示更详细内容 | 
| kill PID	| 终止进程 | 
| kill -9 PID	| 强制终止进程 | 
| top	| 显示以不同条件排序进程，和相关进程的如下信息：物理内存、虚拟内存、CPU 利用率、负载率| 

## 文件权限

### 权限表示符

```
rason@rason-ThinkPad-T430:~$ ls -l
total 40
drwxr-xr-x  2 rason rason 4096 Sep 18 08:23 Desktop
drwxr-xr-x 11 rason rason 4096 Oct 10 17:28 Documents
drwxr-xr-x  3 rason rason 4096 Oct 13 14:48 Downloads

```

输出的第一列表示的是与文件或者目录相关的访问模式或者权限。
权限被分为三组，组中的每个位置代表一个特定的权限，这个顺序是：读(r)、写(w)和执行(x):
- 前三个字符 (2-4) 表示文件的所有者的权限。例如 -rwxr-xr-- 代表，文件的所有者拥有读 (r)、写 (w) 和执行 (x) 的权限。
- 第二组的三个字符 (5-7) 包含了该文件所属组的权限。例如 -rwxr-xr-- 表示了所属组拥有读 (r) 和执行 (x) 的权限，但没有写权限。
- 最后一组三个字符 (8-10) 代表其他人的权限。例如 -rwxr-xr-- 代表其他人只有读 (r) 的权限。

### 改变权限

改变文件或目录的权限，您可以使用 chmod(change mode)命令。有两种方法可以使用 chmod：符号模式和绝对模式。

| Chmod 操作符| 描述 |
| :-----------: | :-------: |
| + | 给文件或者目录添加指定的权限。 | 
| -	| 删除文件或者目录的权限。 | 
| =	| 设置指定的权限 | 

```
$chmod o+wx testfile
$ls -l testfile
-rwxrwxrwx  1 amrood   users 1024  Nov 2 00:10  testfile
$chmod u-x testfile
$ls -l testfile
-rw-rwxrwx  1 amrood   users 1024  Nov 2 00:10  testfile
$chmod g=rx testfile
$ls -l testfile
-rw-r-xrwx  1 amrood   users 1024  Nov 2 00:10  testfile
```

## 其它命令

| 命令 | 作用 |
| :-----------: | :-------: |
| pwd | 打印当前工作目录 | 
| mkdir dirname	| 创建目录 | 
| mkdir -p	| 创建父目录 | 
| ls -l	| 列出的文件的详细信息 | 
| ls -a | 可列出隐藏文件 | 
| cat -b filename	| 查看文件的内容并显示行号 | 
| wc filename	| 统计文件的字数 | 
| passwd	| 修改系统登录密码 | 


原创文章，欢迎转载，无需注明出处。
