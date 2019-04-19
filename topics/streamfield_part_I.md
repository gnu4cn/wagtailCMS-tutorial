# 采用StreamField特性的自由格式页面内容

**Freeform page content using StreamField**

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


[前往`StreamField`第二部分](streamfield_part_II.html)
