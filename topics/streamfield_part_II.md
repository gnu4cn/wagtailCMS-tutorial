# `StreamField`第二部分

[`StreamField`第一部分](streamfield_part_I.md)


## 模板的渲染

`StreamField`特性为流式内容作为整体的HTML表示，也为各个单独的块提供了HTML表示（`StreamField` provides an HTML representation for the stream content as a whole, as well as for each individual block）。而要将此HTML包含进页面中，就要使用`{% include_block %}` 标签。

{% raw %}
    {% load wagtailcore_tags %}

    ...

    {% include_block page.body %}
{% endraw %}

在默认的渲染中，该流的各个块是包围在`<div class="block-my_block_name">`元素中的（其中`my_block_name`就是在该`StreamField`定义中所给的块名称）。若要提供自己的HTML标记，可对该字段的值进行迭代，并依次在各个块调用`{% include_block %}`：

{% raw %}
    ...

    <article>
        {% for block in page.body %}
            <section>{% include_block block %}</section>
        {% endfor %}
    </article>
{% endraw %}

为实现对特定块类型渲染的更多控制，各个块对象都提供了`block_type`与`value`属性：

{% raw %}
    ...

    <article>
        {% for block in page.body %}
            {% if block.block_type == 'heading' %}
                <h1>{{ block.value }}</h1>
            {% else %}
                <section class="block-{{ block.block_type }}">
                    {% include_block block %}
                </section>
            {% endif %}
        {% endfor %}
    </article>
{% endraw %}

默认各个块都是使用简单的、最小的HTML标记，或完全不使用HTML进行渲染的。比如`CharBlock`就是作为普通文本进行渲染的，而`ListBlock`则会将其子块输出到一个`<ul>`包装器中。如要用定制的HTML渲染方式来覆写此行为，可将一个`template`参数传递给该块，从而给到一个要进行渲染的模板文件名。这样做对于一些从`StructBlock`派生的定制块类型，有为有用：

```python
('person', blocks.StructBlock(
    [
        ('first_name', blocks.CharBlock()),
        ('surname', blocks.CharBlock()),
        ('photo', ImageChooserBlock()),
        ('biography', blocks.RichTextBlock()),
    ],
    tempalte='myapp/blocks/person.html',
    icon='user'
))
```

或在将其定义为`StructBlock`的子类时：

```python
class PersonBlock(blocks.StructBlock):
    first_name = blocks.CharBlock()
    surname = blocks.CharBlock()
    photo = ImageChooserBlock(required=False)
    biography = blocks.RichTextBlock()

    class Meta:
        template = 'myapp/blocks/person.html'
        icon = 'user'
```

在模板中，块的值可以变量`value`进行访问：

{% raw %}
    {% load wagtailimages_tags %}

    <div class="person">
        {% image value.photo width-400 %}
        <h2>{{ value.first_name }} {{ value.surname }}</h2>
        {{ value.biography }}
    </div>
{% endraw %}

因为`first_name`、`surname`、`photo`与`biography`都是以其自己地位作为块进行定义的，所以这也可写为下面这样：

{% raw %}
    {% load wagtailimages_tags wagtailcore_tags %}

    <div>
        {% image value.photo width-400 %}
        <h2>{% include_block value.first_name %} {% include_block value.surname %}</h2>
        {% include_block value.biography %}
    </div>
{% endraw %}


`{{ myblock }}` 的写法大致与 `{% include_block my_block %}`等价，但短的形式限制更多，因为其没有将来自所调用模板的变量，比如`request`或`page`，加以传递；因为这个原因，只建议在一些不会渲染其自己的HTML的简单值上使用这种短的形式。比如在`PersonBlock`使用了如下模板时：

{% raw %}
    {% load wagtailiamges_tags %}

    <div class="person">
        {% image value.photo width-400 %}
        <h2>{{ value.first_name }} {{ value.surname }}</h2>

        {% if request.user.is_authenticated %}
            <a href="#">联系此人</a>
        {% endif %}

        {{ value.biography }}
    </div>
{% endraw %}

那么这里的`request.user.is_authenticated`测试，在经由`{{ ... }}`这样的标签进行渲染时便不会工作：

{% raw %}
    {# 错误的写法： #}

    {% for block in page.body %}
        {% if block.block_type == 'person' %}
            <div>{{ block }}</div>
        {% endif %}
    {% endfor %}

    {# 正确的写法： #}

    {% for block in page.body %}
        {% if block.block_type == 'person' %}
            <div>{% include_block block %}</div>
        {% endif %}
    {% endfor %}
{% endraw %}

与Django的`{% include %}`标签类似，`{% include_block %}` 也允许通过`{% include_block with foo="bar" %}`语法，将额外变量传递给所包含的模板：

{% raw %}
    {# 在页面模板中： #}

    {% for block in page.body %}
        {% if block.block_type == 'person' %}
            {% include_block block with classname="important" %}
        {% endif %}
    {% endfor %}

    {# 在PersonBlock的模板中： #}
    <div class="{{ classname }}"></div>
{% endraw %}


还支持 `{% include_block my_block with foo="bar" only %}`语法，以指明除了来自父模板的`foo`变量外，无其他变量传递给子模板。

除了从父模板进行变量传递外，块子类也可通过对`get_context`方法进行重写，传递他们自己额外的模板变量：

```python
import datetime

class EventBlock(blocks.StructBlock):

    title = blocks.CharBlock()
    date = blocks.DateBlock()

    def get_context(self, value, parent_context=None):
        context = super().get_context(value, parent_context=parent_context)
        context['is_happening_today'] = (value['date'] == datetime.date.today())
        return context

    class Meta:
        template = 'myapp/blocks/event.html'
```

在此示例中：变量`is_happening_today`将在该块的模板中成为可用。在该块是经由某个`{% include_block%}`标签进行渲染时，`parent_context`关键字参数会是可用的，且他将是一个从调用该块的模板中传递过来的变量的字典。

## `BoundBlocks`与值

所有块类型，而不仅是`StructBlock`，都接受一个用于确定他们将如何在某个页面上进行渲染的`template`参数。但对于那些处理基本Python数据类型的块，比如`CharBlock`与`IntegerBlock`，在于何处模板生效上有着一些局限，因为这些内建类型（`str`、`int`等等）无法就他们的模板渲染进行“干预”。作为此问题的一个示例，请思考一下的块定义：

```python
class HeadingBlock(blocks.CharBlock):
    class Meta:
        template = 'blocks/heading.html'
```

其中`block/heading.html`的构成是：

```html
<h1>{{ value }}</h1>
```

这就给到一个与普通文本字段一样表现的块，但在其被渲染时，是将其输出封装在`h1`标签中的：

```python
class BlogPage(Page):
    body = StreamField([
        # ...
        ('heading', HeadingBlock()),
        # ...
    ])
```

{% raw %}
    {% load wagtailcore_tags %}

    {% for block in page.body %}
        {% if block.block_type == 'heading' %}
            {% include_block block %} {# 此块将输出他自己的 <h1>...</h1> 标签。 #}
        {% endif %}
    {% endfor %}
{% endraw %}

此种安排 -- 一个期望表示普通文本字符串，但在某个模板上有着其自己的定制HTML表示 -- 通常将是以Python达成的非常糟糕的事，不过在这里将奏效，因为在对某个`StreamField`进行迭代是所获取到的条目，并非这些块的真实“原生”值。相反，每个条目都是作为一个`BoundBlock` -- 一个表示值与值的块定义的对，的实例而加以返回的。`BoundBlock`通过对块定义保持跟踪，而始终知道要进行渲染的模板。而要获取到底层值 -- 在本例中，就是标题的文本内容 -- 就需要访问`block.value`。实际上，如在页面中输出`{% include_block block.value %}`，将发现他是以普通文本进行渲染的，而不带有`<h1>`标签。

（更为准确地说，在对某个`StreamField`进行迭代时，其所返回的条目，是`StreamChild`类的实例，`StreamChild`类提供了`block_type`与`value`两个属性）

有经验的Django开发者可能会发现，将这个与Django的表单框架中，表示表单字段值与其相应的表单字段定义对的`BoundField`类，进行比较而有所帮助，从而明白是怎样将值作为HTML表单字段进行渲染的。

大多数时候，都无需担心这些内部细节问题；Wagtail将在期望使用模板渲染的任何地方，而进行模板渲染。不过在某些此种设想并不完整的情况下 --也就是说，在访问`ListBlock`或`StructBlock`的子块时。在这些情况下，就没有`BoundBlock`的封装器，进而其条目就无法依赖于获悉其自己的渲染模板。比如，请考虑以下设置，其中的`HeadingBlock`是`StructBlock`的一个子块：

```python
class EventBlock(blocks.StructBlock):
    heading = HeadingBlock()
    description = blocks.TextBlock()
    # ...

    class Meta:
        template = 'blocks/event.html'
```

在 `blocks/event.html`:

{% raw %}
    {% load wagtailcore_tags %}
    <div class="event {% if value.heading == "聚会！" %}lots-of-ballons{% endif %} ">
        {% include_block value.bound_blocks.heading %}
        - {% include_block value.description %}
    </div>
{% endraw %}

在具体实践中，在`EventBlock`的模板中把`<h1>`标签显式地写出来，将更为自然且更具可读性：

{% raw %}
    <div class="event {% if value.heading == "聚会！"%}lots-of-balloons{% endif %}">
        <h1>{{ value.heading }}</h1>
        - {% include_block value.description %}
{% endraw %}

这种局限性并不存在于作为`StructBlock`子块的`StructBlock`与`StreamBlock`，因为Wagtail是将他们作为知悉其自己的渲染模板的复杂对象，就算在没有封装在一个`BoundBlock`中，而加以实现的。比如在一个`StructBlock`嵌套于另一个`StructBlock`中事：

```python
class EventBlock(blocks.StructBlock):
    heading = HeadingBlock()
    description = blocks.TextBlock()
    guest_speaker = blocks.StructBlock([
        ('first_name', blocks.CharBlock()),
        ('surname', blocks.CharBlock()),
        ('photo', ImageChooserBlock()),
    ], template='blocks/speaker.html')
```

那么在`EventBlock`的模板中，将如预期的那样，从`blocks/speaker.html`拾取渲染模板。

总的来说，`BoundBlock`s 与普通值之间的互动，遵循以下规则：

1. 在对`StreamField`或`StreamBlock`的值进行迭代时（就像在`{% for block in page.body %}`中那样），将获取到一系列的`BoundBlock`s。

2. 在有着一个`BoundBlock`实例时，可以`block.value`访问到其普通值。

3. 对`StructBlock`子块的访问（比如在`value.heading`中那样），将返回一个普通值；而要获取到`BoundBlock`的值，就要使用`value.bound_blocks.heading`语法。

4. `ListBlock`的值，是一个普通的Python清单；对`ListBlock`的迭代，将返回普通的子元素值。

5. 与`BoundBlock`不同，`StructBlock`与`StreamBlock`的值，总是知道如何去渲染他们自己的模板，就算仅有着普通值。

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
