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

+ `pageurl()`

生成某个页面实例的URL：

```html
    <a href="{{ pageurl(page.more_information) }}">更多信息</a>
```

请参考 [`pageurl`](https://wagtail.xfoss.com/topics/writing_templates.md#pageurl-tag) 了解更多信息。

+ `slugurl()`

生成某个带有别名的页面（a Page with a slug）的URL：

```html
 <a href="{{ slugurl("about") }}">关于我们</a>
```

请参考 [`slugurl`](https://wagtail.xfoss.com/topics/writing_template.md#slugurl-tag) 了解更多信息。

+ `image()`

对某个图片进行缩放，并输出一个 `<img>` 标签：

```html
    {# 输出一个图片标签 #}
    {{ image(page.header_image), "fill-1024x200", class="header-image" }}

    {# 缩放某个图片 #}
    {% set background=image(page.background_image, "max-1024x1024") %}
    <div class="wrapper" style="background-image: url({{ background.url }})"></div>
```

请参阅 [在模板中使用图片](https://wagtail.xfoss.com/topics/images.md#image-tag) 了解更多信息。

+ `|richtext`

对 Wagtail 的内部HTML表示进行转换，将一些内部引用扩展为页面与图片。

```html
    {{ page.body|richtext }}
```

请参阅 [富文本（过滤器）](https://wagtail.xfoss.com/writing_templates.md#rich-text-filter) 以了解更多信息。

+ `wagtailuserbar()`

输出前端上用于对页面进行编辑的 Wagtail 上下文弹出式菜单

```html
    {{ wagtailuserbar() }}
```

请参考 [Wagtail的用户栏](https://wagtail.xfoss.com/writing_templates.md#wagtailuserbar-tag) 以获取更多信息。


+ `{% include_block %}`

将流式内容（the stream content）作为一个整体，输出他的HTML表示，对于各个单独块也是作为整体输出为HTML表示的。

允许将模板上下文传递给`StreamField`的模板。

```html
    {% include_block page.body %}
    {% include_block page.body with context %} {# 与上面的写法相同 #}
    {% include_block page.body without context %}
```


请参考 [`StreamField` 模板的渲染](https://wagtail.xfoss.com/topics/streamfield.md#streamfield-template-rendering)  以了解更多信息。


> __注意__ `{% include_block %}`标签设计用于严格遵循 Jinja 的 `{% include %}` 标签的语法与行为，因此其并没有实现 Django 版本的仅传递指定变量到上下文中的特性。
