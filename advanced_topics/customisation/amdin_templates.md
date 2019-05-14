# 对管理模板进行定制

在Wagtail项目中，可能希望在管理界面中，使用自己的品牌元素，来替换默认的Wagtail徽标等元素。可经由Django的继承机制，实现此目标。

需要在某个应用目录下，创建一个`templates/wagtailadmin/`的文件夹 -- 这既可以是一个既有的文件夹，也可以是为此目的而新建的文件夹，比如 `dashboard`。该应用必须是已在 `INSTALLED_APPS` 中，于`wagtail.admin`之前已注册的应用：

```python
INSTALLED_APPS = (
    # ...
    'dashboard',

    'wagtail.core',
    'wagtail.admin',

    # ...
)
```


## 品牌的定制

可用于对管理界面中的品牌加以定制的模板块包含以下这些：

+ `branding_logo`

    要替换默认的徽标，就要创建一个对默认的`branding_logo`块进行覆写的模板文件 `dashboard/templates/wagtailadmin/base.html`：

{% raw %}

    {% extends "wagtailadmin/base.html" %}
    {% load static %}

    {% block branding_logo %}
        <img src="{% static 'images/custom-logo.svg' %}" alt="定制项目" width="80">
    {% endblock %}

{% endraw %}


+
+
