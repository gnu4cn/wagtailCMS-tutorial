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

每个这些属性，都被设置为了一个`EditHandler`对象的清单，该清单定义了哪些字段将出现在哪个分页上，以及这些字段在各个分页上的结构方式。

下面是这些Wagtail自带的`EditHandler`类的简介。请参阅[可用的面板类型](reference/pages/panels.md)，了解他们的完整介绍。

### 基本面板（Basic）

基本面板实现模型字段的编辑。`FieldPanel`类会基于字段类型，来选择适当的小部件，而`StreamField`字段则就需要使用特殊面板类。

+ `FieldPanel`
+ `StreamFieldPanel`

### 结构化面板（Structural）

这类面板用于对界面中的字段进行结构组织。

+ `MultiFieldPanel` -- 用于将相似字段分组到一起
+ `InlinePanel` -- 用于将子模型置于行内化（For inlining child models）
+ `FieldRowPanel` -- 用于将多个字段组织为单个的行

### 选择器面板（Chooser）

到某些确切模型的`ForeignKey`字段，可以使用以下的那些 `ChooserPanel` 类之一。这些面板把美观的模态选择器对话框界面加入进来，同时图片/文档选择器还允许在不离开页面编辑器的情况下，进行新文件的上传。

+ `PageChooserPanel`
+ `ImageChooserPanel`
+ `DocumentChooserPanel`
+ `SinippetChooserPanel`

> **注意** 要使用这些选择器，字段所链接到的模型，就必须是某个页面、图片、文档或内容块。
> 对于链接到其他模型类型的情况，就应使用`FieldPanel`了，这时将创建出一个下拉选择框。

## 对页面编辑器界面进行定制

页面编辑器可被深度定制，请参阅 [定制编辑接口](advanced_topics/customisation/page_editing_interface.md)。


## 父页面/子页面的类型规则

**Parent page / subpage type rules**

下面的两个属性，可实现在站点中何处使用何种页面类型的控制。允许定义出如同“博客条目只能创建于博客首页下”这样的规则。

两条属性都取的是模型类或模型名称的清单。模型名称的格式为 `app_label.ModelName`。在省略了 `app_label`时，假定就是同样的应用。

+ `parent_page_types` 限制本类型可在哪些页面类型之下创建
+ `subpage_types` 限制在本类型下可以创建哪些页面类型

默认情况下，在所有页面类型之下，可以创建任意的页面类型，而如果要的就是这种效果的话，就没有必要设置这些属性。

将`parent_page_types`设置为空的清单，是一种阻止在编辑器界面创建出某个特定页面类型的好做法。


## 页面模型的URL模式的定制

通常不会直接调用`Page.get_url_parts(request)`，但为了给某个给定的页面模型定义定制的URL路由，是可以覆写该方法的（the `Page.get_url_parts(request)` method will not typically be called directly, but may be overridden to define custion URL routing for a given page model）。该方法会返回一个`(site_id, root_url, page_path)`的元组，该元组为`get_url`与`get_full_url`（下面可以看到）用来构造他和给定的页面URL类型。

在覆写`get_url_parts()`方法时，需接受`*args, **kwargs`:

```python
def get_url_parts(self, *args, **kwargs):
```

并在调用`super`（如有 `super`）上的`get_url_parts`方法处，将`*args, **kwargs`传递过去，比如：

```python
super().get_url_parts(*args, **kwargs)
```

虽然可以只传递`request`关键字参数，但将所有这些参数按原样进行传递，可确保在以后这些方法签名发生改变时保持兼容性。

有关更多此方面的信息，请参阅 [`wagtail.core.models.Page.get_url_parts()`](reference/pages/model_reference.md#wagtail.core.models.Page.get_url_parts)。

### 获取页面实例的URLs

在需要某个页面的URL时，可调用`Page.get_url(request)`方法。在该方法能够探测到页面位于当前站点（通过 `request.site`）的情况下，其默认返回的是本地URLs（不包含协议或域名）；否则将返回一个包含了协议与域名的完整URL。为了开启站点级别的URL信息的每次请求缓存，以及为了实现本地URLs的生成，应尽可能将可选参数`request`包含进去（whenever possible, the optional `request` argument should be included to enable per-request caching of site-level URL information and facilitate the generation of local URLs）。

`get_url(request)`的一个常见用例，就是在项目中包含了用于生成导航菜单的所有定制模板标签中。在编写这样的定制模板标签时，要确保这些标签包含有`takes_context=True`，及确保他们都用到了`context.get('request')`，从而安全地传递上请求，或在上下文中不存在请求时，要传递`None`（A common use case for `get_url(request)` is in any custom template tag your project may include for generating navigation menus. When writing such a custom template tag, ensure that it includes `takes_context=True` and use `context.get('request')` to safely pass the request or `None` if no request exists in the context）。

有关此方面的更多信息，请参阅[wagtail.core.models.Page.get_url()](reference/pages/model_reference.md#wagtail.core.models.Page.get_url)。



