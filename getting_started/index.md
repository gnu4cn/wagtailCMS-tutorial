# 入门

Wagtail 是构建于 [Django web 框架](https://www.djangoproject.com/) 之上的，因此本文档假定已经安装好了必要的软件包。如尚未安装这些软件包，那么就要安装以下必要的软件/程序：

+ [Python](https://www.python.org/downloads/)
+ [pip](https://pip.pypa.io/en/latest/installing.html)（请注意 `pip` 已默认包含在Python 3.4及以后的发布中）


同时建议使用 Virtualenv, 其提供了隔离的Python环境：

+ [Virtualenv](https://virtualenv.pypa.io/en/latest/installation.html)


> **要点** 在安装 Wagtail 之前，有必要安装 `libjpeg` 与 `zlib` 两个库，他们提供了处理 JPEG、PNG 与 GIF 图像的支持（通过 Python 的`Pillow`库）。而安装这两个库的方式，则根据平台的不同而有所区别 -- 请参阅 Pillow 的 [特定平台安装说明](http://pillow.readthedocs.org/en/latest/installation.html#external-libraries)。


在安装了上述库之后，安装Wagtail的最快方式就是：

*若使用了 Virtualenv, 那么请运行*

```sh
$python3 -m venv .venv
```

```sh
$ mkdir wagtail-demo && cd wagtail-demo
$. ~/.venv/bin/activate
$ pip install wagtail
```

在安装完毕后，Wagtail提供了一个类似于 Django 的 `django-admin startproject`命令，用于生成一个新的站点/项目（stubs out a new site/project）：

```sh
$ wagtail start demo
```

这将基于一个包含了满足起步所有需要的模板，而建立一个新的文件夹 `demo`。更多有关该模板的信息，请移步 [这里](getting_started/project_template.md)。

此时在 `demo` 文件夹中，只要运行一下对于所有Django项目来说都需要的必要几步：

```sh
$ pip install -r requirements.txt
$ ./manage.py migrate
$ ./manage.py createsuperuser
$ ./manage.py runserver
```

现在就可以在`http://localhost:8000`访问到该站点了，同时在`http://localhost:8000/admin`出可以访问到管理后端。

这些步骤建立起来一个新的单机化的Wagtail项目。如要将Wagtail加入到某个既有的Django项目，则请参阅[将Wagtail集成到Django项目中](getting_started/integrating_into_django.md)。

有一些可选`pip`包没有包含到默认安装，但推荐使用他们来提高性能或赋予Wagtail某些特性，包括：

+ [Elasticsearch](advanced_topics/performance.md)
+ [特性发现](advanced_topics/images/feature_detection.md)
