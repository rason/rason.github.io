title: 【Django现学现卖1】在新购的阿里云上搭建Django的HelloWorld
categories: Django
date: 2016-03-03 15:46:41
tags: Django
description: 阿里云部署Django
---

## 前言

听说Python这门语言很牛逼，也听说`Django在手，天下我有`。所以抽了两天时间大概看了下Python的语法和Django框架，按照`learning by doing`的思想，就立马购买了一个月阿里云服务器，动手搭建第一个Django应用，初学者按照以下操作就可以搭建一个Django的`Hello, world.`了。

## 步骤

**1.基本准备**

Ubuntu系统电脑一台（当然Unix系其它也行）和阿里云服务器一台（自行购买）。

**2.登录阿里云服务器**

使用`ssh user@ip`命名登录，user一般为root，当然你也可用其它用户，如果有的话；ip即为你购买的阿里云服务器ip。

**3.更新服务器**

`sudo apt-get update`

**4.安装apache**

`sudo apt-get install apache2 libapache2-mod-wsgi`

**5.安装pip**

`sudo apt-get install python-pip`

<!-- more -->

**6.安装Django**

`pip install Django`

**7.安装Mysql**

`sudo apt-get install mysql-server mysql-client`

**8.创建数据库**

先使用命令`mysql -u root -p`登录数据库，再使用命令`create database ebookloader default charset utf8 collate utf8_general_ci;`创建一个数据库，然后`exit;`退出。

**9.安装 MySQLdb**

`pip install mysql-python`

如果安装过程有问题，可先安装下述软件包，然后再安装 MySQLdb

```
sudo apt-get install python-setuptools
sudo apt-get install libmysqld-dev
sudo apt-get install libmysqlclient-dev
sudo apt-get install python-dev
```

**10.创建Django项目**

`django-admin startproject ebookloader`

为了简单起见并没有创建Python虚拟环境，ebookloader为项目名称，会创建相应的ebookloader目录，我的是保存在根目录下。

**11.配置Apache2**

先删除默认的配置文件`rm /etc/apache2/sites-enabled/000-default.conf `。

再使用命令`sudo vi /etc/apache2/sites-available/sitename.conf`创建网站配置文件，文件内容可参照：

```
<VirtualHost *:80>
  ServerName www.ebookloader.com
  ServerAlias ebookloader.com
  ServerAdmin rason
  Alias /media/ /ebookloader/media/
  Alias /static/ /ebookloader/static/
  <Directory /ebookloader/media>
    Require all granted
  </Directory>
  <Directory /ebookloader/static>
    Require all granted
  </Directory>
  WSGIScriptAlias / /ebookloader/ebookloader/wsgi.py
  <Directory /ebookloader/ebookloader>
  <Files wsgi.py>
    Require all granted
  </Files>
  </Directory>
</VirtualHost>

```

上面的配置中写的 `WSGIScriptAlias / /ebookloader/ebookloader/wsgi.py` 把apache2和你的网站联系起来，但为了让脚本找到Django项目的位置，得再修改下 `/ebookloader/ebookloader/wsgi.py` 文件为以下内容：

```
import os
import sys

from django.core.wsgi import get_wsgi_application
from os.path import join,dirname,abspath

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "ebookloader.settings")
PROJECT_DIR = dirname(dirname(abspath(__file__)))
sys.path.insert(0,PROJECT_DIR)
application = get_wsgi_application()

```

**12.启用网站**

执行`sudo a2ensite sitename.conf`，根据提示需要执行`service apache2 reload`重启apache。

**13.浏览器输入你的服务器IP**

当显示以下内容，恭喜你成功了。如果不是，呵呵哒，自行Google排查问题咯。

![成功页面](/image/python-first_django_app.png)

好了，Django项目搭建完成，现在可以开始搭建我们的App了。

**14.创建App**

进入项目的中包含`manage.py`文件的文件夹，执行`python manage.py startapp booksystem`创建应用，完成之后文件结构如下：

```
root@iZ94hd1izoaZ:/ebookloader# tree booksystem/
booksystem/
├── admin.py
├── apps.py
├── __init__.py
├── migrations
│   └── __init__.py
├── models.py
├── tests.py
└── views.py

booksystem就是我们的应用名字，保存在项目文件夹ebookloader下。

```
**15.注册App**

编辑 settings.py 文件，在 `INSTALLED_APPS` 中添加 booksystem

```
# Application definition

INSTALLED_APPS = ( 
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'booksystem',
)
```

**16.编辑url**

`root@iZ94hd1izoaZ:/ebookloader/ebookloader# vi urls.py `

在 urlpatterns 中添加：

`url(r'^$', 'ebookloader.views.home', name='home'),`

修改后文件完整内容如下：

```
from django.conf.urls import url
from django.contrib import admin

urlpatterns = [
    url(r'^admin/', admin.site.urls),
    url(r'^$', 'ebookloader.views.home', name='home'),
]

```

**17.编辑view**

`root@iZ94hd1izoaZ:/ebookloader/ebookloader# vi views.py`

内容如下：

```
from django.http import HttpResponse

def blog_index(request):
    return HttpResponse("Hello, world.")
```

**18.重启Apache**

执行`service apache2 reload`，在浏览器输入服务器IP，请求成功后即可看到 `Hello, world.`

**大功造成！**

## 附加内容

上面安装了数据库，实际`Hello world`应用还没有用到Mysql数据库，安装只是为了日后使用。

**1.配置数据库**

看一下`settings.py` 中数据库的配置，可以看出默认的数据库即为 sqlite3

```
# Database
# https://docs.djangoproject.com/en/1.8/ref/settings/#databases

DATABASES = { 
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
    }   
}
```

我们将其修改数据库为 MySQL

```
DATABASES = { 
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'ebookloader',
        'USER': 'root',
        'PASSWORD': 'password',
        'HOST': '127.0.0.1',
    }   
}
```

**2.语言和时区**

同样是在`settings.py`文件中,修改为中文和本地市区

```
LANGUAGE_CODE = 'zh-CN'
TIME_ZONE = 'Asia/Shenzhen'
```

## 总结

**动起手来吧，问题不是你想象中那么的困难**

