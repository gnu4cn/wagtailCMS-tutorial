# Wagtail下对Django的配置

要从零开始安装Wagtail，就要创建一个Django项目，并在那个项目中建立一个应用。有关这些任务的教程，请参见[编写第一个Django应用](https://docs.djangoproject.com/en/stable/intro/tutorial01/)。项目的目录将像下面这样：

```sh
myproject/
    myproject/
        __init__.py
        settings.py
        urls.py
        wsgi.py
    myapp/
        __init__.py
        models.py
        tests.py
        admin.py
        views.py
    manage.py
```

可安全地从应用目录中将`admin.py`与`views.py`移除，因为Wagtail将为模型提供到此功能。对Django进行加载Wagtail的配置，涉及将一些模块与变量加入到`settings.py`，以及将URL的配置加入到`urls.py`。请参考 [Django设置](https://docs.djangoproject.com/en/stable/topics/settings/)与[Django的URL调度器](https://docs.djangoproject.com/en/stable/topics/http/urls/)，以获取对这些文件中定义了什么的全面了解。

下面的内容，是略过了许多的Django样板性设置的部分设置参考。如此时只想要尽快安装好Wagtail，而不想纠缠于这些设置，那么请参见[已准备好使用示例配置文件](#complete-example-config)。

<a name="middleware"></a>
## 关于中间件（`settings.py`）

<a name="apps"></a>
## 关于应用（`settings.py`）

<a name="settings-vaiables"></a>
## 设置变量（`settings.py`）

## URL模式

<a name="complete-example-config"></a>
## 已准备好使用示例配置文件

