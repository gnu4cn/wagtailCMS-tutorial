# 页面模型

**Page models**

Wagtail中的每种页面类型（又名“内容类型”），都是由一个Django的模型所表示的（each page type(a.k.a. content type) in Wagtail is represented by a Django model）。所有页面模型，都必须从`wagtail.core.models.Page`类继承。

> **注** Django模型，就是一个类的数据结构，对应着数据库中的一个表。Python类与数据库之间存在一个对象关系模型的抽象，对页面的修改，经由 migrate 命令，最终反映到数据库中。

因为所有页面类型都是Django模型，那么就可以使用Django所提供的所有字段类型。请参阅[模型字段参考](https://docs.djangoproject.com/en/stable/ref/models/fields/)，了解所有可用字段清单。此外，Wagtail还提供了一个`RichTextField`字段，该字段提供了一个可用于编辑富文本内容的所见即所得的编辑器。

**关于Django的模型**

如对Django模型尚不熟悉，那么请快速浏览以下链接，以获得基本的掌握：

+ [创建模型](https://docs.djangoproject.com/en/stable/intro/tutorial02/#creating-models)
+ [模型的语法](https://docs.djangoproject.com/en/stable/topics/db/models/)


## Wagtail页面模型示例

下面的示例表示一个典型的博客文章：

```python
from django.db import models

from modelcluster.fields import ParentalKey

from wagtail.core.models import Page, Orderable
from wagtail.core.fields import RichTextField
from wagtail.admin.edit_handlers import FieldPanel, MultiFieldPanel, InlinePanel
from wagtail.images.edit_handlers import ImageChooserPanel
from wagtail.search import index

class BlogPost(Page):

    # 数据库字段

    body = RichTextField()
    date = models.DateField("发布日期")
    feed_image = models.ForeignKey(
        'wagtailimages.Image',
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
        related_name="+"
    )

    # 搜索功能索引的配置

    search_fields = Page.search_fields + [
        index.SearchField('body'),
        index.FilterField('date'),
    ]

    # 编辑器面板的配置

    content_panels = Page.content_panels + [
        FieldPanel('date'),
        FieldPanel('body', classname="full"),
        InlinePanel('related_links', label="相关链接"),
    ]

    promote_panels = [
        MultiFieldPanel(Page.promote_panels, "一般性页面配置"),
        ImageChooserPanel('feed_image'),
    ]

    # 父页面/子页面类型规则

    parent_page_types = ['blog.BlogIndex']
    subpage_types = []

class BlogPageRelatedLink(Orderable):
    page = ParentalKey(BlogPage, on_delete=models.CASCADE, related_name='related_links')
    name = models.CharField(max_length=255)
    url = models.URLField()

    panels = [
        FieldPanel('name'),
        FieldPanel('url'),
    ]
```

> **要点** 请确保字段名称与类的名称保持不同。那会因为Django处理对象关系的机制（[了解更多有关此方面的信息](https://github.com/wagtail/wagtail/issues/503)），而导致出错。在上面的示例中已经通过将`Page`追加到各个模型名称之后，避免了这个问题。

<a name="writing-page-models"></a>
## 编写页面模型

这里将对上面的实力中的各个部分进行逐一讲解，以帮助建立自己的页面模型。

<a name="database-fields"></a>
### 数据库字段

每种Wagtail页面类型，都是一个Django模型，在数据库中表现为单个表。

每个页面类型，都可以有他自己的一套字段。比如一篇新闻报道可能具有文章主题文本与发布日期，而某项活动则可能需要单独的活动地点及开始/结束时间字段。

在Wagtail中，可以使用所有的Django字段类。同时由第三方应用所提供的绝大多数字段类同样可以工作。

Wagtail还提供了一些他自身的字段类：

+ `RichTextField` -- 用于富文本内容
+ `StreamField` -- 一种基于块的内容字段（参见：[使用`StreamField`的自由格式页面内容](topics/streamfield.md)）

对于标签化特性，Wagtail完全支持[django-taggit](https://django-taggit.readthedocs.org/en/latest/)，因此建议使用该现成的库。

<a name="search"></a>
## 搜索功能

页面模型中的`search_fields`属性，定义了哪些字段要被加入到搜索索引，以及他们如何被索引。

该属性影视一个`SearchField`与`FilterField`对象的清单。其中`SearchField`将某个字段加入到全文搜索。`FilterField`则是将某个字段加入到结果过滤中（`FilterField` adds a field for filtering the results）。字段可被`SearchField`与`FilterField`同时索引（但只能每个索引只能有一个实例）。

在上面的示例中，将`body`字段进行了全文搜索索引，将`date`字段做了过滤索引。

这些字段类型所接受的参数，在[对额外字段进行索引](topics/search/indexing.md#wagtailsearch-indexing-fields)中进行了文档说明。

### 编辑器面板

对于页面的这些字段如何在页面编辑器界面的安排，有着几个属性：

+ `content_panels` -- 用于内容，比如文章主题文本
+ `promote_panels` -- 用于元数据，比如标签、缩略图及搜索引擎优化的标题等
+ `settings_panels` -- 用于相关设置，比如发布日期等

每个这些属性，都被设置到一个`EditorHandler`对象的清单，该清单定义了那些字段将出现在哪个分页上，以及这些字段在各个分页上的结构方式。


