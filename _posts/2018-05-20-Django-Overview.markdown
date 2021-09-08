---
layout: post
title: Django之 概述
date: 2018-05-20 23:32:24.000000000 +09:00
---

这段时间学习django ，就为了做一个自己的运维系统，看来看去后端还是使用django更加合适，主要是考虑到上手比较简单。首先有个概念，就是project , 项目。比如说我打算做个运维系统，那我创建个项目就叫 operating system， 但是这个项目创建出来只是一个框架。然后我们就需要 application , 应用来提供服务，比如说我这个运维系统里面要提供监控，要提供CMDB ，所以我还会创建几个应用叫 “监控” ， “CMDB” 之类的。

安装 Django 比较简单，使用pip3 install django 就可以了，然后使用 django-admin startproject projectName 就可以创建一个新的项目了，执行完这个命令可以看到当前目录下多出了一个跟项目名称一样的目录，里面会是这样一个结构：

{% highlight ruby %}

nick@nick-ThinkPad-T61:~/PycharmProjects$ django-admin startproject projectName

nick@nick-ThinkPad-T61:~/PycharmProjects$ tree projectName
projectName
├── manage.py
└── projectName
    ├── __init__.py
    ├── settings.py
    ├── urls.py
    └── wsgi.py

1 directory, 5 files
{% endhighlight %}

manage.py 这个文件可以看成这个项目的一个管理文件，比如说运行这个文件时可以看到有这样的选项：

{% highlight ruby %}

nick@nick-ThinkPad-T61:~/PycharmProjects/projectName$ python3 manage.py 

Type 'manage.py help <subcommand>' for help on a specific subcommand.

Available subcommands:

[auth]

……

[contenttypes]
 ……

[django]
    ……

[sessions]
    ……

[staticfiles]
    ……

{% endhighlight %}

看，这里主要就是我们这个项目的各种管理功能。然后再看看这个文件的内容，

```python
if __name__ == "__main__":
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "projectName.settings")
    try:
        from django.core.management import execute_from_command_line
    except ImportError as exc:
        raise ImportError(
            "Couldn't import Django. Are you sure it's installed and "
            "available on your PYTHONPATH environment variable? Did you "
            "forget to activate a virtual environment?"
        ) from exc
    execute_from_command_line(sys.argv)
```

其实这么几行最主要的就是加载了我们的 projectName.settings 文件，这个文件里主要是这个项目的配置，比如说使用的应用，样本，数据库，语言等等的配置。 django就是根据这个文件去寻找对应的各种功能配置，所以以后新建应用的时候就需要在这里声明添加应用，如果添加了样本的目录，就要在这里的样本对应的地址里添加目录地址。

urls.py 这个文件里面主要是有这么一个tuple , 

```python
from django.contrib import admin
from django.urls import path

urlpatterns = [
    path('admin/', admin.site.urls),
]
```

这个是配置一个路由信息，根据不同的 url 地址来相应不同的应用，后面再另起一篇文章仔细说说这个。

到了这里，还需要创建一个应用来提供服务。

{% highlight ruby %}

nick@nick-ThinkPad-T61:~/PycharmProjects/projectName$ django-admin startapp app
nick@nick-ThinkPad-T61:~/PycharmProjects/projectName$ ls
app  manage.py  projectName
nick@nick-ThinkPad-T61:~/PycharmProjects/projectName$ tree app
app
├── admin.py
├── apps.py
├── __init__.py
├── migrations
│   └── __init__.py
├── models.py
├── tests.py
└── views.py

1 directory, 7 files

{% endhighlight %}

这样我们就创建了一个名为 app 的应用，只要我们在 settings 里面声明了这个应用，并在 urls.py 里添加对应的相应地址，用户就可以通过 url 来使用我们这个应用了。这里面有两个很重要的文件，首先就是 views.py 了， 这个文件定义了我们如何相应客户的请求;然后就是 models.py 了，这个是处理数据库的文件，这些都值得另起文章讲述。

说到这里，我就想说说设计模式了， MTV 可能对于大众来说更常见的可能是 MVC 。相信做开发或者运维的朋友们都知道松耦合这个概念 ， 松耦合的出现主要是为了解决功能的分离，这样不同的功能模块可以分开同时开发，主要留出了统一的接口，各种模块怎么发展都无所谓。就好比说办身份证要去派出所，但是派出所后面的流程怎么改变，人员怎么调动，我们都不需要管。只要需要办理身份证相关的事情，我们就去派出所，当然准备文件提前预约什么的这个我们就不说了，但是大概可以这么理解。换到我们这个网站开发上面来说，就是前端跟后端分离，前端再怎么改变都好，只要接口没有变动，我后端还是一样的提供服务不会因为前端的变化而跟着改变。所以传统的 MVC 的概念就是这样，全称是 module , view , controller .

module , 是模块，就是各种独立的功能模块，主要是跟数据库打交道的东西。比如说我运维系统里面可能有监控，有CMDB ， 有用户管理……那我就可以分别有监控模块，CMDB模块，用户管理模块等等。各功能模块可以分开独立开发，其间的联系就看接下来的

controller , 这里主要解决的是商业逻辑，比如说用户想查询一下线上有多少机器，那可能需要输入几个限制条件，比如说机器是否在线，机器的操作系统，机器的各种配置等等。获取到所需信息之后再跟module 去查询数据库，那查询到信息之后怎么返回给用户呢，就是接下来的

view ， 解决的就是返回的信息如何更好的展示给用户，这里主要就是前端的工作了。

根据这三个分类就可以把网站开发给分开了，达到了一个松耦合的目的。而我们的django 有所不同，提出的设计模式是 MVT 

这里 M 还是 module , 主要解决数据库的交互问题;然后 V 就是 view , django 里面的管理逻辑都放到这个里面，比如说我们上面说的应用的文件框架里面有一个 views.py ，这个文件解决的就是如何相应用户的请求;最后最大的区别就是这个 T 了， template , django提供了一个很强大的模板系统 ， 开发者可以通过定义模板来便捷的生成用户界面 。

Django的大致介绍就到这里了，后续再聊聊各种组件的应用。
