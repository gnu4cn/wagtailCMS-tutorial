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

<a name="parent-page-subpage-type-rules"></a>
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

在需要完整的URL（包含协议与域名）时，可使用`Page.get_full_url(request)`方法。同样应尽可能包含`request`参数，为的是启用站点级别的URL信息的历次请求的缓存。有关此方面的更多信息，请参阅[wagtail.core.models.Page.get_full_url()](reference/pages/model_reference.md#wagta      il.core.models.Page.get_full_url)

## 模板的渲染

可以给予每个页面模型一个HTML的模板，在用户浏览到站点前端的某个页面时，模板将被渲染出来。这就是将Wagtail内容给到终端用户的最简单也是最常见方式（这不是唯一的方式哦）。

### 将某个页面模型的模板添加进来

Wagtail将基于应用级别与模型类的名称，自动选择模板的名称。

格式为：`<app_label>/<model_name(蛇姓字母的)>.html`

> **注** 蛇形字母写法，snake cased，参见[Snake case, wikipedia](https://en.wikipedia.org/wiki/Snake_case)

比如上面的博客页面的模板将是： `blog/blog_page.html`

只需在某个可以此名称访问到的地方，创建一个模板即可。

### 模板上下文

**Template context**

Wagtail 将以绑定到正在渲染的页面实力上的`page`变量，来进行模板的渲染。使用该变量来访问页面的内容。比如要获得当前页面的标题，就使用`{{ page.title }}`。所有由 [上下文处理器](https://docs.djangoproject.com/en/stable/ref/templates/api/#subclassing-context-requestcontext) 提供的变量，也都是可用的。

** 模板上下文的定制 **

所有页面都有一个`get_context()`方法，在渲染模板时，都会调用到该方法，其返回值为绑定到模板变量的一个字典。

可通过覆写该方法，来讲更多变量加入到模板上下文：

```python
class BlogIndexPage(Page):
    ...

    def get_context(self, request):
        context = super().get_context(request)

        # 加入额外变量，并返回更新后的上下文
        context['blog_entries'] = BlogPage.objects.child_of(self).live()
        # 注意上面就是一个 QuerySet
        return context
```

此时这些变量就可以在模板中加以使用了：

```python
{{ page.title }}

{% for enry in blog_entries %}
    {{ entry.titlte }}
{% endfor %}
```

### 修改模板

通过在类上设置 `template` 属性，来使用一个不同的模板文件：

```python
class BlogPage(Page):
    ...

    template = 'other_tempalte.html'
```

**动态地选择模板**

通过在页面类上定义`get_template()`方法，可以给每个实例指定不同的模板。在每次页面被渲染时，都会调用该方法：

```python
class BlogPage(Page):
    ...

    use_other_template = models.BooleanField()

    def get_template(self, request):
        if self.use_other_template:
            return 'blog/other_blog_page.html'

        return 'blog/blog_page.html'
```

在本示例中，那些设置了`use_other_template`字段的页面，将使用`blog/other_blog_page.html`模板。所有其他页面都将使用默认的`blog/blog_page.html`。

### 在页面渲染上的更多控制

所有页面类，都有一个`serve()`方法，该方法内部调用了`get_context`与`get_template`方法，进而对模板进行渲染。此方法与Django的视图函数类似，取的是一个Django `Request` 对象，返回的是一个 Django `Response` 对象。

为实现对页面渲染的完全掌控，此方法可被重写。

比如下面就是一种让页面以一个表示其自身的JSON方式进行响应的方法：

```python
from django.http import JsonResponse

class BlogPage(Page):
    ...

    def serve(self, request):
        return JsonResponse({
            'title': self.title,
            'body': self.body,
            'date': self.date,

            # 将图片缩放到宽度为 300px 并取得到图片的URL
            'feed_image': self.feed_image.get_rendition('width-200').url,
        })
```

## 内联模型

**Inline models**

Wagtail可在页面中嵌套其他模型的内容。对于创建那些诸如相关链接，或要以旋转木马方式显示的重复字段，此特性是有用的。内联模型的内容，与其他页面内容一样，保存在版本变更中。

每个内联模型，都有以下要求：

+ 必须继承自`wagtail.core.models.Orderable`
+ 必须要有一个到父模型的`ParentalKey`

> **注意** `django-modelcluster`与`ParentalKey`的区别
> 模型的内联特性，是由[`django-modelcluster`](https://github.com/torchbox/django-modelcluster)所提供的，同时必须要从这里将`ParentalKey`字段类型导入进来：
> `from modelcluster.fields import ParentalKey`
> `ParentalKey` 是 Django 的`ForeignKey`的一个子类，并取的是同样的参数。


比如下面的内联模型就可用于将相关链接（一个名称-url对的的列表）加入到`BlogPage`模型：

```python
from django.db import models
from modelcluster.fields import ParentalKey
from wagtail.core.models import Orderable

class BlogPageRelatedLink(Orderable):
    page = ParentalKey(BlogPage, on_delete=models.CASCADE, realted_name='related_links')
    name = models.CharField(max_length=250)
    url = models.URLField()

    panels = [
        FieldPanel('name'),
        FieldPanel('url'),
    ]
```

接着使用`InlinePanel`编辑面板类，将其加入到管理界面：

```python
content_panels = [
    ...

    InlinePanel('related_links', label="相关链接"),
]
```

`InlinePanel`类构造函数的第一个参数，必须与`ParentalKey`的`related_name`属性一致。

## 处理多个页面

**Working with pages**

Wagtail使用了Django的 [多表继承](https://docs.djangoproject.com/en/stable/topics/db/models/#multi-table-inheritance) 特性，来实现在相同的树中使用多个页面模型。

每个页面都会被同时加入到Wagtail内建的`Page`模型，以及其用户定义的模型（比如早先所创建的`BlogPage`模型）。

相应地，页面可以两种形式的Python代码存在，`Page`基类的示例，或页面模型的示例。

在一并处理多个页面类型时，通常使用的是Wagtail的 `Page` 模型，其不会给予到对特定于这些页面类型的那些字段的访问。

```bash
# 获取数据库中的所有页面
>>> from wagtail.core.models import Page
>>> Page.objects.all()
[<Page: Homepage>, <Page: About us>, <Page: Blog>, <Page: A Blog post>, <Page: Another Blog post>]
```

> **注** 通过 `python manage.py shell` 即可进入到上面的命令行界面

在处理单个的页面类型时，就可以对用户定义的模型实例进行处理了。这样的模型实例，给予到对`Page`中所有可用字段的访问，以及对该特定类型的用户定义字段的访问。

```bash
# 获取数据库中所有的博客条目
>>> from blog.models import BlogPage
>>> BlogPage.objects.all()
[<BlogPage: A Blog post>, <BlogPage: Another Blog post>]
```

使用`.specfific`属性，就可以将某个`Page`对象，转换为其更为具体的用户定义的等价实体。然而这会引发一次额外的数据库查询。

```bash
>>> page = Page.objects.get(title='A Blog Post')
>>> page
<Page: A Blog post>

# 注意：该博客文章是 Page 的一个实例，因此不能访问到文章主体、日期或首页图片

>>> page.specific
<BlogPage: A Blog post>
```

## 技巧

### 友好的模型名称

运用带有 `verbose_name` 属性的Django的内部`Meta`类，可令到模型名称对用户更为友好，比如：

```python
class HomePage(Page):

    ...

    class Meta:
        verbose_name = "homepage"
```

默认情况下，在用户选择创建要创建的页面时，页面类型清单是通过将模型名称按照大写字符拆分，而生成的。那么`HomePage`模型就将被命名为“Home Page”，显然这个名字有点笨拙。在如同上面的示例中那样定义了 `verbose_name`后，就会将名称修改为“Homepage”，这样就更为习以为常一点。

> verbose: 详细

### 页面QuerySet的排序

由基类`Page`所派生的模型，是 *无法* 通过使用标准的加入 `ordering` 属性到模型内部的`Meta`类上，这种标准的Django 方法，而赋予给他一种默认排序的。

```python
Class NewsItemPage(Page):
    
    publication_date = models.DateField()
    ...

    class Meta:
        ordering = ('-publication_date', ) # 不会生效
```

这是因为`Page`是强制以路径对QuerySet进行排序的。 因此在构造一个QuerySet时，就必须显示地进行排序：

```python
news_items = NewsItemPage.objects.live().order_by('-publication_date')
```

### 对页面管理器进行定制

可将定制的`Manager`类，添加到`Page`类。所有定制管理器，都应继承自`wagtail.core.models.PageManager`：

```python
from django.db import models

from wagtail.core.models import Page, PageManager

class EventPageManager(PageManager):
    """ 活动页面的定制管理器  """

class EventPage(Page):

    start_date = models.DateField()

    objects = EventPageManager()
```

而在仅需加入额外的 `QuerySet` 方法时，还可以从`wagtail.core.models.PageQuerySet` 继承， 并调用`from_queryset()`来构建一个定制的 `Manager` 类：

```python
from django.db import models
from django.utils import timezone

from wagtail.core.models import Page, PageManager, PageQuerySet

class EventPageQuerySet(PageQuerySet):

    def future(self):
        today = timezone.localtime(timezone.now()).date()
        return self.filter(start_date__gte=today)

EventPageManager = PageManager.from_queryset(EventPageQuerySet)

class EventPage(Page):
    
    start_date = models.DateField()

    objects = EventPageManager()
```
