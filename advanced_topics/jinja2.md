# Jinja2 模板的支持

Wagtail支持 Jinja2 模板的全部前端特性。更多有关以下模板标签的信息，可在 [编写模板](https://wagtail.xfoss.com/topics/writing_templates.md#writing_templates) 文档中找到。

## 对Django进行配置

需要对 Django 进行配置，以支持到 Jinja2 模板。因为 Wagtail 的管理界面是以常规 Django 模板进行编写的，因此就必须将Django配置为同时使用两张模板引擎。请将下面的配置添加到应用的 `TEMPLATE` 设置项：

```python
TEMPLATES = [
    # ...

    {
        'BACKEND': 'django.template.backends.jinja2.Jinja2',
        'APP_DIRS': True,
        'OPTIONS': {
            'extensions': [
                'wagtail.core.jinja2tags.core',
                'wagtail.admin.jinja2tags.userbar',
                'wagtail.images.jinja2tags.images',
            ]
        }
    }
]
```

Jinja 的模板，必须放在应用的 `jinja2/` 目录中。比如`events` 应用中的 `EventPage` 模型的模板，就应被创建于 `events/jinja2/events/event_page.html`。

默认下，Jinja 的环境并没有任何的 Django 函数或过滤器。Django文档中有着有关 [为 Django 而对 Jinja 进行配置](https://docs.djangoproject.com/en/stable/topics/templates/#django.template.backends.jinja2.Jinja2)的更多信息。

## 模板中的 `self`

在 Django 的模板中，`self` 可用于对当前页面、流式块（stream block）或字段面板进行引用。但在 Jinja 中，`self`则是保留作内部用途的。在编写Jinja模板时，使用 `page` 来引用页面，使用`value`来引用流式块，同时使用`field_panel`来引用字段面板。

## 模板标签、函数与过滤器


