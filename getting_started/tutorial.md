# 第一个Wagtail站点

> **注意** 此教程讲的是有关建立一个全新Wagtail项目的内容。如要将Wagtail加入到某个既有Django项目，请参考[将Wagtail集成到Django项目](integrating_into_django.md)。

1. 安装 Wagtail 与其依赖：

```sh
$ pip install wagtail
```

2. 开始你的站点：

```sh
$ wagtail start mysite
$ cd mysite
```

Wagtail提供了与`django-admin.py startproject` 类似的 `start` 命令。在项目中运行 `wagtail start mysite`将生成一个新的、有着几个特定于 Wagtail 的附加文件的 `mysite` 文件夹，这些文件包括了：

+ 所需的项目设置
+ 一个有着空白的 `HomePage` 模型的“主页”应用
+ 一些基础模板
+ 一个简单的“搜索”应用。

3. 安装项目依赖

```sh
$ pip install -r requirements.txt
```

此步骤确保刚创建的项目具有相关版本的Django

4. 创建数据库

```sh
$ ./manage.py migrate
```

在没有更新项目设置时，数据库将是项目目录中的一个 SQLite 数据库文件。

5. 创建出一个管理员用户

```sh
$ ./manage.py createsuperuser
```

6. 启动服务器

```sh
$ ./manage.py runserver
```

如没有什么错误的话，访问`http://127.0.0.1:8000`就可以看到一个欢迎页面了：

![Wagtail欢迎页面](../images/tutorial_1.png)


