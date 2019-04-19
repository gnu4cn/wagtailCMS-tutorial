# 采用StreamField特性的自由格式页面内容

__Freeform page content using StreamField__

Wagtail的StreamField特性，提供了一个适合于那些并不遵循固定结构 -- 诸如博客文章或新闻报道 -- 的一类页面的内容编辑模型，在这样的页面中，文本可能穿插有子标题、图片、拉取引用及视频等元素。此种内容编辑模型也适合于那些更为专用的内容类型，比如地图或图表（或编程博客、代码片段等）等。在该模型中，这些各异的内容类型是以序列的“块”来表示的，这些块可以重复并以任意顺序进行安排。

有关StreamField特性的更多背景知识，以及为什么要在文章主体使用StreamField，而不使用富文本字段的原因，请参阅博客文章[Rich text fields and faster horses](https://torchbox.com/blog/rich-text-fields-and-faster-horses/)。

StreamField还提供到一个丰富的，用于定义从简单的子块集合（比如由姓、名及相片组成的`person`），到带有自己的编辑界面的、完全定制化组件的定制块类型的API。在数据库中，StreamField内容是作为JSON进行存储的，确保了该字段的全部信息内容都得以保留，而不仅是其HTML的表现形式。

## 使用StreamField

`StreamField`是一个可像所有其他字段一样，在页面模型中进行定义的模型字段：

```python
from django.db import models

from wagtail.core.models import Page
from wagtail.core.fields import StreamField
from wagtail.core import blocks
from wagtail.admin.edit_handlers import FieldPanel, StreamFieldPanel
from wagtail.images.blocks import ImageChooserPanel

class BlogPage(Page):
    author = models.CharField(max_length=255)
    date = models.DateField("发布日期")
    body = StreamField([
        ('heading', blocks.CharBlock(classname="full title")),
        ('paragragh', blocks.RichTextBlock()),
        ('image', ImageChooserBlock()),
    ])

    content_panels = Page.content_panels + [
        FieldPanel('author'),
        FieldPanel('date'),
        StreamField('body'),
    ]
```

注意：StreamField并不向后兼容诸如`RichTextField`这样的其他字段类型。如需将某个既有字段迁移到StreamField，请参考[将RichTextFields迁移到StreamField](#migrating-richtext)。

`StreamField`构造函数的参数，是一个`(name, block_type)`的元组的清单。`name`用于在模板与内部的JSON表示中对块类型进行标识（同时应遵循Python变量名的约定：小写字母与下划线、没有空格），而`block_type`就应是一个下面所讲到的块定义的对象。（此外，`StreamField`也可传入单个的`StreamBlock`实例 -- 请参阅[结构化的块类型](#structural-block-types)）

这样就定义了可在该字段里使用的一套可用块类型。该页面的作者可自由使用这些块，以任意顺序，想用几次就用几次。

`StreamField`还接受可选的关键字参数`blank`，该参数默认为`False`；在其为`False`时，就必须为该字段提供至少一个的块，方能视为该字段有效。

## 基本的块类型

所有块类型，都接受一下的可选关键字参数：

+ `default`

    应接受到的一个新的“空”块的默认值。

+ `label`

    在引用到此块时，编辑器界面所显示的标签 -- 默认为该块名称的一个美化了的版本（或在莫个上下文中没有指定名称时--比如在某个`listBlock`里 -- 的空字符串）。

+ `icon`

    在可用块类型菜单用于显示该块类型的图标的名称。可通过在项目的`INSTALLED_APPS`中，加入`wagtail.contrib.styleguide`，来开启图标名称的清单，该清单的更多信息，请参阅[Wagtail样式手册]()。
+ `template`

    到将用于在前端上渲染此块的Django模板的路径。请参考[模板渲染](#template-rendering)。

+ `group`

    用于对此块进行分类的组，即所有有着同样组名称的块，将在编辑器界面中，以该组名称作为标题显示在一起。

Wagtail提供了以下一些基本块类型：

### `CharBlock`

`wagtail.core.blocks.CharBlock`

一种单行的文本输入。接受一下关键字参数：

+ `required` （默认值：`True`）

    在为`True`时，该字段不能留空。

+ `max_length`, `min_length`

    确保字符串至多或至少有给定的长度。

+ `help_text`

    显示于该字段旁边的帮助文本。

### `TextBlock`

`wagtail.core.blocks.TextBlock`

一个多行的文本输入。与`CharBlock`一样，接受关键字参数关键字`required`（默认值：`True`）、`max_length`、`min_length`与`help_text`。

### `EmailBlock`

`wagtail.core.blocks.EmailBlock`

一个单行的email输入，会验证email字段是一个有效的Email地址。接受关键字参数`required`（默认值：`True`）与`help_text`。

### `IntegerBlock`

`wagtail.core.blocks.IntegerBlock`

一个单行的整数输入，会验证该整数是一个有效的整数。接受关键字参数`required`（默认值：`True`）、`max_value`、`min_value`与`help_text`。

### `FloatBlock`

`wagtail.core.blocks.FloatBlock`

一个单行的浮点数输入，会验证该值是一个有效的浮点数。接受关键字参数`required`（默认值：`True`）、`max_value`与`min_value`。

### `DecimalBlock`

`wagtail.core.blocks.DecimalBlock`

一个单行的小数输入，会验证该整数是一个有效的小数。接受关键字参数`required`（默认值：`True`）、`help_text`、`max_value`、`min_value`、`max_digits`与`decimal_places`。

有关`DecimalBlock`的用例，请参阅[示例：`PersonBlock`](#personblock-example)。

### `RegexBlock`

`wagtail.core.blocks.RegexBlock`

一个单行的文本输入，会将该字符串与一个正则表达式进行比对。用于验证的正则表达式，必须作为第一个参数，或一关键字参数`regex`进行提供。为了对用于表示验证错误的消息文本进行定制，就要将一个包含了键`required`（用于不显示消息）或`invalid`（用于在不匹配值时显示的消息）的字典，作为关键字参数`error_messages`加以传入。

```python

    blocks.RegexBlock(regex=r`^[0-9]{3}$`, error_messages={
        'invalid': "不是一个有效的图书馆卡编号"
    })
```

接受`regex`、`help_text`、`required`（默认值：`True`）、`max_length`、`min_length`与`error_messages`关键字参数。


### `URLBlock`

`wagtail.core.blocks.URLBlock`

一个单行的文本输入，会验证其字符串为一个有效的URL。接受关键字参数`required`（默认值：`True`）、`max_length`、`min_length`与`help_text`。

### `Boolean_Block`

`wagtail.core.blocks.BooleanBlock`

一个复选框。接受关键字参数`required`与`help_text`。与Django的`BooleanField`一样，一个`required=True`（默认的）值表明必须勾选该复选框才能继续。对于一个即可勾选也可不勾选的复选框，就必须显式的传入`required=False`。

### `DateBlock`

`wagtail.core.blocks.DateBlock`

一个日期选择器。接受`required`（默认值：`True`）、`help_text`与`format`关键字参数。

`format`（默认值：`None`）

日期格式。该参数必须是在[`DATE_INPUT_FORMATS`](https://docs.djangoproject.com/en/stable/ref/settings/#std:setting-DATE_INPUT_FORMATS)设置项中能识别的格式之一。在没有指定的该参数时，Wagtail将使用`WAGTAIL_DATE_FORMAT`的设置，而回滚到`%Y-%m-%d`的格式。

> **译者注** 此块类型为何没有`start_date`、`end_date`这样的关键字参数呢？


### `TimeBlock`

`wagtail.core.blocks.TimeBlock`

一个时间拾取器。接受关键字参数`required`（默认值：`True`）与`help_text`。

### `DateTimeBlock`

`wagtail.core.blocks.DateTimeBlock`

一个结合了日期/时间的拾取器。接受关键字参数`required`（默认值：`True`）、`help_text`与`format`。

`format`（默认值：`None`）

日期格式。该参数必须是在[`DATETIME_INPUT_FORMATS`](https://docs.djangoproject.com/en/stable/ref/settings/#std:setting-DATETIME_INPUT_FORMATS)设置项中能识别的格式之一。在没有指定的该参数时，Wagtail将使用`WAGTAIL_DATETIME_FORMAT`的设置，而回滚到`%Y-%m-%d %H:%M`的格式。

### `RichTextBlock`

`wagtail.core.blocks.RichTextBlock`

一个用于创建包含链接、粗体/斜体等内容的格式化文本的所见即所得的文本编辑器。接受关键字参数`features`，用于制定所允许的特性集合（请参阅[在富文本字段中对特性进行限制](advanced_topics/customisation.html#rich-text-features)）。

### `RawHTMLBlock`

`wagtail.core.blocks.RawHTMLBlock`

一个用于输入原始HTML的文本编辑区域，这些原始HTML将在页面输出中，进行转义渲染。接受关键字参数`required`（默认值：`True`）、`max_length`、`min_length`与`help_text`。

>  *警告* 在使用此种块时，没有防止站点编辑将恶意脚本，包括那些可能在有另一名管理员查看该页面时，允许当前管理员寻求获取到管理员权限的脚本，插入到页面的机制。所以除非能够充分信任站点编辑，那么请不要使用此种块类型。

### `BlockQuoteBlock`

`wagtail.core.blocks.BlockQuoteBlock`

一个文本字段，其内容将以一个HTML的`<blockquote>`标签对包围起来。接受关键字参数`required`（默认值：`True`）、`max_length`、`min_length`与`help_text`。

### `ChoiceBlock`

`wagtail.core.blocks.ChoiceBlock`

一个用于从选项清单中进行选择的下拉式选择框。接受以下关键字参数：

+ `choices`

    一个选项清单，以所有Django模型字段的`choices`参数所能接受的格式；或者一个可返回此种清单的可调用元素。


+ `required`（默认：`True`）

    在为`True`时，该字段不能留空。


+ `help_text`

    显示于该字段旁边的帮助文本

`ChoiceBlock`也可以被子类化，而生成一个有着同样的、在所有地方都用到的选项清单的可重用块。比如下面这个块的定义：

```python
blocks.ChoiceBlock(choices=[
    ('tea': '茶'),
    ('coffee': '咖啡')，
], icon="cup")
```

就可以被重写为`ChoiceBlock`的一个子类：

```python
class DrinksChoiceBlock(blocks.ChoiceBlock):

    choices = [
        ('tea': '茶'),
        ('coffee': '咖啡')，
    ]

    class Meta:
        icon = 'cup'
```

此时`StreamField`的那些定义就可以在完整的`ChoiceBlock`定义处，对`DrinksChoiceBlock()`加以引用了。请注意这仅在`choices`是一个固定清单，而非可调用元素时，才能工作。

### `PageChooserBlock`

`wagtail.core.blocks.PageChooserBlock`

一个用于选择页面对象的控件，使用了Wagtail的页面浏览器（a control for selecting a page object, using Wagtail's page browser）。 接受以下关键字参数：

+ `required`（默认：`True`）
    
    在为`True`时，该字段不能留空

+ `target_model`（默认：`Page`）

    将选择限制到一个或更多的特定页面类型。此关键字参数接受某个页面模型类、模型名称（作为字符串），或他们的一个清单或元组。

+ `can_choose_root`（默认：`False`）

    在为`True`时，站点编辑可将Wagtail树的根，选作页面。正常情况下这样做是不可取的，因为Wagtail树的根绝不会是一个可用的页面，但在某些特殊场合，这样做却是恰当的。比如在某个块提供了相关文章种子时，就会使用一个`PageChooserBlock`，来选择由哪个站点文章子板块，来作为相关文章种子来源，而树根就对应“所有板块”（for example, a block providing a feed of related articles could use a `PageChooserBlock` to select which subsection of the site articles will be taken from, with the root corresponding to 'everywhere'）。

### `DocumentChooserBlock`

`wagtail.core.block.DocumentChooserBlock`

一个允许网站编辑选择某个既有文档对象，或上传新的文档对象的控件。接受关键字参数`required`（默认：`True`）。

### `ImageChooserBlock`

`wagtail.core.blocks.ImageChooserBlock`


一个允许网站编辑选择某个既有图片，或上传新的图片的控件。接受关键字参数`required`（默认：`True`）。

### `SnippetChooserBlock`

一个允许站点编辑选取内容片段对象的控件。需要一个位置性参数：应从哪个内容片段类处选取。接受关键字参数`required`（默认：`True`）。

### `EmbedBlock`

`wagtail.core.blocks.EmbedBlock`

用于站点编辑输入一个到媒体条目（比如Youtube视频）URL，而作为页面上嵌入的媒体的控件。接受关键字参数`required`（默认：`True`）、`max_length`、`min_length`与`help_text`。

### `StaticBlock`

`wagtail.core.blocks.StaticBlock`

这是一个不带有字段的块，因此在渲染其模板时，不会传递特定的值给其模板。这在需要站点编辑插入某些任何时候都不变的内容，或无需在页面编辑器中进行配置的内容时，比如某个地址、来自第三方服务的嵌入代码，或一些在模板使用到模板标签时的更为复杂的代码等，尤为有用。

默认将在编辑器界面显示一些文本（在传入了`label`关键字参数时，就是该关键字参数），因此该块看起来并不是空的。但可通以关键字参数`admin_text`传入一个文本字符串，而对其进行整个的定制：

```python
blocks.StaticField(
    admin_text='最新文章：无需配置。',
    # 或 admin_text=mark_safe('<b>最新文章</b>：无需配置。'),
    template='latest_posts.html'
)
```

`StaticBlock`也可以进行子类化，而生成一个带有在任何地方都可使用的某些配置的一个可重用块：

```python
class LatestPostsStaticBlock(blocks.StaticBlock):

    class Meta:
        icon = 'user'
        lable = '最新文章'
        admin_text = '{label}: 在其他地方配置'.format(label=label)
        template = 'latest_posts.html'
```

## 结构化的块类型

**Structural block types**

除了上面的这些基本块类型外，定义一些由子块所构成的新的块类型也是可行的：比如由姓、名与照片构成的`person`块，或由不限制数量的图片块构成的一个`carousel`块。这类结构可以任意深度进行嵌套，从而令到某个结构包含块清单，或者结构的清单。

### `StructBlock`

`wagtail.core.blocks.StructBlock`

一个由在一起显示的子块的固定组别所构成的块。取一个`(name, block_definition)`的元组，作为其首个参数：

```python
('person', blocks.StructBlock([
    ('first_name', blocks.CharBlock()),
    ('surname', blocks.CharBlock()),
    ('photo', ImageChooserBlock(required=False)),
    ('biography', blocks.RichTextBlock()),
], icon='user'))
```

此外，子块清单也可在某个`StructBlock`的子类中加以提供：

```python
class PersonBlock(blocks.StructBlock):

    first_name = blocks.CharBlock()
    surname = blocks.CharBlock()
    photo = ImageChooserBlock(required=False)
    biography = blocks.RichTextBlock()

    class Meta:
        icon = 'user'
```

该`Meta`类支持属性`default`、`label`、`icon`与`template`，这些属性与将他们传递给该块的构造器时，有着同样的意义。

上面的代码将`PersonBlock()`定义为了一个可在模型定义中想重用多少次都可以的块类型。

```python
body = StreamField([
    ('heading', blocks.CharBlock(classname="full title")),
    ('paragraph', blocks.RichTextBlock()),
    ('image', ImageChooserBlock()),
    ('author', PersonBlock()),
])
```

更多有关对页面编辑器中的`StrucBlock`的显示进行定制的选项，请参阅[定制`StructBlock`的编辑界面](#custom-editing-interfaces-for-structblock)。

同时还可对如何将`StructBlock`的值加以准备，以在模板中使用而进行定制 -- 请参阅[定制`StructBlock`的值类](#custom-value-class-for-structblock)。

### `ListBlock`

`wagtail.core.blocks.ListBlock`

由许多同样类型的子块所构成的块。站点编辑可将不限数量的子块添加进来，并对其进行重新排序与删除。取子块的定义作为他的首个参数：

```python
('ingredients_list', blocks.ListBlock(
    blocks.CharBlock(label='营养成分')
))
```

可将所有块类型作为子块的类型，包括结构化块类型：

```python
('ingredients_list', blocks.ListBlock(
    blocks.StructBlock([
        ('ingredient', blocks.CharBlock()),
        ('amount', blocks.CharBlock(required=False)),
    ])
))
```


### `StreamBlock`

`wagtail.core.blocks.StreamBlock`

一种由一系列不同类型的子块构成的快，这些子块可混合在一起，并依意愿进行重新排序。作为`StreamField`本身的整体机制而进行使用，也可在其他结构化块类型加以嵌套或使用。将一个`(name, block_definition)`元组清单，作为其首个参数：

```python
('carousel', blocks.StreamField([
    ('image', ImageChooserBlock()),
    ('quotation', blocks.StuctBlock([
        ('text', blocks.TextBlock()),
        ('author', blocks.CharBlock()),
    ])),
    ('video', EmbedBlock()),
], icon='cogs'))
```

与`StructBlock`一样，子块清单也可作为`StreamBlock`的子类加以提供：

```python
class CarouselBlock(blocks.StreamBlock):

    image = blocks.ImageChooserBlock()
    quotation = blocks.StructBlock([
        ('text', blocks.TextBlock()),
        ('author', blocks.CharBlock()),
    ])
    video = EmbedBlock()

    class Meta:
        
        icon = 'cogs'
```

因为`StreamField`在块类型清单处接受了一个`StreamBlock`的实例作为参数，这就令到在不重复定义的情况下，重复使用一套通用的块类型成为可能（since `StreamField` accepts an instance of `StreamBlock` as a parameter, in place of a list block types, this makes it possible to re-use a common set of block types without repeating definitions）：

```python
class HomePage(Page):

    carousel = StreamField(CarouselBlock(max_num=10, block_counts={'video': {'max_num': 2}}))
```

`StreamBlock`接受以下选项，作为关键字参数或`Meta`的属性：

+ `required`（默认：`True`）

    在为`True`时，就要至少提供一个子块。这在将`StreamBlock`作为某个`StreamField`的顶级块使用时被忽略；在此情况下，该`StreamField`的`blank`属性优先。

+ `min_num`

    该`StreamBlock`至少应有的子块数量。

+ `max_num`

    该`StreamBlock`最多应有的子快数量。

+ `block_counts`

    指定各个子块类型下最小与最大数量，是以子块名称到可选的`min_num`与`max_num`字典的映射字典。


<a name="personblock-example"></a>
## 示例`PersonBlock`

本示例对如何将上面讲到的基本块类型，结合到一个更为复杂的基于`StructBlock`的块类型中：

```python
from wagtail.core import blocks

class PersonBlock(blocks.StructBlock):

    name = blocks.CharBlock()
    height = blocks.DecimalBlock()
    age = blocks.IntegerBlock()
    email = blocks.EmailBlock()

    class Meta:
        
        template = 'blocks/person_block.html'
```

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

`1`. 在对`StreamField`或`StreamBlock`的值进行迭代时（就像在

{% raw %}
    
    {% for block in page.body %}
{% endraw %}
中那样），将获取到一系列的`BoundBlock`s。

`2`. 在有着一个`BoundBlock`实例时，可以`block.value`访问到其普通值。

`3`. 对`StructBlock`子块的访问（比如在`value.heading`中那样），将返回一个普通值；而要获取到`BoundBlock`的值，就要使用`value.bound_blocks.heading`语法。

`4`. `ListBlock`的值，是一个普通的Python清单；对`ListBlock`的迭代，将返回普通的子元素值。

`5`. 与`BoundBlock`不同，`StructBlock`与`StreamBlock`的值，总是知道如何去渲染他们自己的模板，就算仅有着普通值。

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

__Custom value class for `StructBlock`__

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

在需要实现某个定制UI，或要处理某种Wagtail内建的块类型所未提供（且无法作为既有字段的一个结构而构建出来）的数据类型时，就要定义自己的定制块类型了。请参考Wagtail的内建块类的源代码，以获取更详细的说明。

对于那些简单地将既有Django表单字段进行封装的块类型，Wagtail提供了一个抽象类`wagtail.core.blocks.FieldBlock`作为助手类（a helper(class)）。那些子类就只需设置一个返回该表单字段对象的`field`属性即可：

```python
class IPAddressBlock(FieldBlock):
    def __init__(self, required=True, help_text=None, **kwargs):
        self.field = forms.GenericIPAddressField(required=required, help_text=help_text)
        super().__init__(**kwargs)
```

## 迁移方面

### 在数据库迁移内的`StreamField`定义

__StreamField definitions within migrations__

就如同Django中的所有模型字段一样，所有会对`StreamField`造成影响的对模型定义的修改，都将造就一个包含该字段定义的“冻结”副本的数据库迁移文件。因为一个`StreamField`定义比一个典型的模型定义更为复杂，所以就会存在来自导入到数据库迁移的项目的增加了可能性的定义 -- 而这就会在后期这些定义被移动或删除时，导致一些问题出现（as with any model field in Django, any changed to a model definition that affect a StreamField will result in a migration file that contains a 'frozen' copy of that field definition. Since a StreamField definition is more complex that a typical model field, there is an increased likelihood of definitions from your project being imported into the migration -- which would cause problems later on if those definitions are moved or deleted）。
