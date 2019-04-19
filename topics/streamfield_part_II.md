# `StreamField`第二部分

[`StreamField`第一部分](streamfield_part_I.md)

## 定制`StructBlock`的编辑界面

要对呈现在页面编辑器中的`StructBlock`的样式进行定制，可为其指定一个`form_classname`的属性（既可以作为`StructBlock`构造器的一个关键字参数，也可放在某个子类的`Meta`中），以覆写`struct-block`这个默认值：

```python
class PersonBlock(blocks.StructBlock):
    first_name = blocks.CharBlock()
    surname = blocks.CharBlock()
    photo = ImageChooserBlock()
    biography = blocks.RichTextBlock()

    class Meta:
        icon = 'user'
        form_classname = 'person-block struct-block'
```

此时便可为该块提供定制的CSS了，以该指定的CSS类名称为目标，通过 [`insert_editor_css`钩子](reference/hooks.html#insert-editor-css)。

> **注意** Wagtail的编辑器样式机制，有着一些`struct-block`类及其他相关元素的内建样式。在制定了`form_classname`的值时，将覆写已经应用到`StructBlock`那些CSS类，因此必须记得要同时要指定`struct-block`CSS类。

而对于那些需要修改HTML标记的更具扩展性的定制，则可在`Meta`中覆写`form_template`属性，以制定自己的模板路径。此种模板中支持以下这些变量：

+ `children`

    所有构成该`StructBlock`的子块的一个`BoundBlock`s的`OrderedDict`；通常`StructBlock`指定的模板，将调用这些`OrderedDict`上的`render_form`方法。

+ `help_text`

    如有制定`help_text`, 则为该块的帮助文本。

+ `classname`

    以`form_classname`所传递的CSS类的名称（默认为`struct-block`）。

+ `block_definition`

    定义此块的`StructBlock`实例。

+ `prefix`

    该块实例的用在表单字段上的前缀，确保了在整个表单范围表单字段的唯一性。

可通过覆写块的`get_form_context`方法，来加入一些额外的变量：

```python
class PersonBlock(blocks.StructBlock):

    first_name = blocks.CharBlock()
    surname = blocks.CharBlock()
    photo = ImageChooserBlock()
    biography = blocks.RichTextBlock()

    def get_form_context(self, value, prefix='', errors=None):
        context = super().get_form_context(value, prefix=prefix, errors=errors)
        context['suggested_first_name'] = ['John', 'Paul', 'George', 'Ringo']
        return context

    class Meta:
        icon = 'user'
        form_template = 'myapp/block_forms/person.html'
```

## 对`StructBlock`的值类进行定制

**Custom value class for `StructBlock`**

可通过指定一个`value_class`的属性（即可作为`StructBlock`构造器的一个关键字参数，也可放在某个子类的`Meta`中），来对`StructBlock`子块的值如何加以准备，而实现对`StructBlock`值的可用方法的定制。

而该`value_class`必须是`StructValue`基类的一个子类，所有额外方法，都可以从子块经由该块在`self`上的键（比如`self.get('my_block')`），访问到该子块的值。

比如：

```python
from wagtail.core.models import Page
from wagtail.core.blocks import (
    CharBlock, PageChooserBlock, StructValue, StructBlock, TextBlock, URLBlock)

class LinkStructValue(StructValue):
    def url(self):
        external_url = self.get('external_url')
        page = self.get('page')
        if external_url:
            return external_url
        elif page:
            return page.url

class QuickLinkBlock(StructBlock):
    text = CharBlock(label='链接文本', required=True)
    page = PageChoooserBlock(label='页面', required=False)
    external_url = URLBlock(label='外部URL', required=False)

    class Meta:
        icon = 'site'
        value_class = LinkStructValue

class MyPage(Page):
    quick_links = StreamField([('链接', QuickLinkBlock())], blank=True)
    quotations = StreamField([('引用'， StructBlock([
        ('quote', TextBlock(required=True)),
        ('page', PageChooserBlock(required=False)),
        ('external_url', URLBlock(required=False)),
    ], icon='openquote', value_class=LinkStructValue))], blank=True)

    content_panels = Page.content_panels + [
        StreamFieldPanel('quick_links'),
        StreamFieldPanel('quotations'),
    ]
```

此时所扩展的值类方法，就在模板中可用了：

{% raw %}

    {% load watailcore_tags %}
    <ul>
        {% for link in page.quick_links %}
            <li><a href="{{ link.value.url }}">{{ link.value.text }}</a></li>
        {% endfor %}
    </ul>

    <div>
        {% for quotation in page.quotations %}
            <blockquote cite="{{ quotation.value.url }}">
                {{ quotation.value.quote }}
            </blockquote>
        {% endfor %}
    </div>
{% endraw %}

## 对块类型进行定制

在需要实现某个定制UI，或要处理某种Wagtail内建的块类型所未提供（且无法作为既有字段的一个结构而构建出来）的数据类型，就要定义自己的定制块类型了。
