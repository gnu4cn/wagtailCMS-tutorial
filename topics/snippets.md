# 内容块（Snippets）

所谓片段（Snippets），是指小片的、没有充分理由渲染成为一个完整网页的内容。他们可用于次级内容，诸如页面头部、页脚以及侧边栏等的制作，在Wagtail管理界面可进行编辑。片段是一些没有从`Page`基类进行继承的Django模型，并因此没有组织到Wagtail树中。但他们仍可通过赋予其面板，而成为可编辑的内容，并经由`register_snippet`类装饰器，将某个模型标识为片段。

片段缺少页面的很多特性，比如在Wagtail管理界面可被排序，或拥有一个定义的URL。要仔细区分打算构建为片段的内容类型，是否更适合构建为一个页面。

## 片段模型

下面是一个片段模型的示例：

```python
from django.db import models

from wagtail.admin.edit_handlers import FieldPanel
from wagtail.snippets.models import register_snippet

...
@register_snippet
class Advert(models.Model):
    url = models.URLField(null=True, blank=True)
    text = models.CharField(max_length=255)

    panels = [
        FieldPanel('url'),
        FieldPanel('text'),
    ]

    def __str__(self):
        return self.text
```

该`Advert`模型使用了基本的Django模型类，并定义了两个属性：`text`与`URL`。编辑节目与提供给派生自`Page`的类非常接近，有着在`panels`属性中所指派的几个字段。片段不会用到多个分栏的字段，同时也不提供“保存为草稿”或“提交审核”特性。

`@register_snippet`告诉Wagtail将该模型作为一个片段进行处理。`panels`清单定义了在该片段的编辑页面上展示的字段。经由`def __str__(self):` 提供一个表示该类的字符串也很重要，这样做才能在片段于Wagtail管理界面中被列出时，片段对象才有意义。

## 在模板标签中将片段包含进去

将片段暴露给模板的最简单方式，就是通过使用模板标签。这主要是通过普通的Django实现的，因此最好复习一下Django的[django定制模板标签](https://docs.djangoproject.com/en/stable/howto/custom-template-tags/)文档，那将更有助益。这里仍将回顾到基础知识，还将指出一些对于Wagtail需要考虑的地方（We'll go over the basics, though, and point out any considerations to make for Wagtail）。

首先将一个新的Python文件加入到应用中的`templatetags`文件夹 -- 比如`myproject/demo/templatetags/demo_tags.py`。这里将装入一些Django的模块，以及应用的模型，并准备好`register`装饰器：

```python
from django import template
from demo.models import Advert

register = template.Library()

...

# Advert 片段
@register.inclusion_tag('demo/tags/advert.html', takes_context=True)
def adverts(context):
    return {
        'adverts': Advert.objects.all(),
        'request': conext['request'],
    }
```

`@register.inclusion_tag()` 需要两个变量：一个模板与一个该模板是否要传入一个请求上下文的逻辑值。将请求上下文包含在定制模板标签中，是种好的做法，因为某些特定于Wagtail的模板标签，如`pageurl`，就需要上下文才能正确工作。模板标签函数可取一些参数，并对该`adverts`进行过滤，以返回一个特定模型，这里因为简要而仅使用了`Advert.objects.all()`。

下面是使用到模板标签所使用的模板：

```html
{% for advert in adverts %}
    <p>
        <a href="{{ advert.url }}">{{ advert.text }}</a>
    </p>
{% endfor %}
```

随后在页面模板中，据可以这样来将该片段模板标签包含进来了：

{% raw %}
    {% load wagtailcore_tags demo_tags %}

    ...
    {% block content %}
        ...

        {% adverts %}
    {% endblock %}
{% endraw %}

## 将页面绑定到片段

在上述示例中，`adverts`的清单是一个固定清单，其显示是独立于页面内容的。这种形式对于某个侧边栏中的普通面板可能是预期的效果，但在其他场景下，可能希望对某个页面内容中的特定片段进行引用（this might be what you want for a common panel in a sidebar, say -- but in other scenarios you may wish to refer to a particular snippet from within a page's content）。这可通过在页面模型中定义一个到片段模型的外键，并将一个`SnippetChooserPanel`到添加到页面的`content_panels`来实现。比如在打算指定某个`advert`要出现在`BookPage`上时：

```python
from wagtail.snippets.edit_handlers import SnippetChooserPanel

# ...

class BookPage(Page):
    
    advert = models.ForeignKey(
        'demo.Advert',
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
        related_name='+'
    )

    content_panels = Page.content_panels + [
        SnippetChooserPanel('advert'),
        # ...
    ]
```

随后该片段就可以在模板中作为`page.advert`进行访问了。

要将多个`adverts`附加到某个页面，就可将`SnippetChooserPanel`放置在`BookPage`的某个内联子对象上，而不要在`BookPage`上。下面的子模型被命名为了`BookPageAdvertPlacement`（之所以这样命名，是因为对于每次将`advert`放置在`BookPage`上时，都有一个这样的对象，here this child model is named `BookPageAdvertPlacement`(so called because there is on such object for each time that an advert is placed on a BookPage)）。

```python
from django.db import models

from wagtail.core.models import Page, Orderable
from wagtail.snippets.edit_handlers import SnippetChooserPanel

from modelcluster.fields import ParentalKey

...

class BookPageAdvertPlacement(Orderable, models.Model):

    page = ParentalKey('demo.BookPage', on_delete=models.CASCADE, related_name='advert_placements')
    advert = models.ForeignKey('demo.Advert', on_delete=models.CASCADE, related_name='+')

    class Meta:
        verbose_name = "广告位"
        verbose_name_plural = "广告位"

    panels = [
        SnippetChooserPanel('advert'),
    ]

    def __str__(self):
        return self.page.title + " -> " + self.advert.text

class BookPage(Page):
    ...

    content_panels = Page.content_panels + [
        InlinePanel('advert_placements', label="广告"),
        # ...
    ]
```

现在这些子对象就可经由页面的`advert_placements`属性访问到了，且从那里可以`advert`访问到链接的`Advert`。在`BookPage`的模板中，可包含以下代码：

```html
{% for advert_placement in page.advert_placements.all %}
    <p><a href="{{ advert_placement.advert.url }}">{{ advert_placement.advert.text }}</a></p>
{% endfor %}
```

## 令到片段可被搜索

在片段模型继承了[对定制模型进行索引](search.html#indexing-custom-models)所降到的`wagtail.search.index.Indexed`时，Wagtail将自动把一个搜索框添加到那个片段类型的选择器界面上。比如该`Advert`片段可像下面这样做成可搜索的：

```python
...

from wagtail.search import index
...

@register_snippet
class Advert(index.Indexed, models.Model):
    url = models.URLField(null=True, blank=True)
    text = models.CharField(max_length=255)

    panels = [
        FieldPanel('url'),
        FieldPanel('text'),
    ]

    search_fields = [
        index.SearchField('text', partial_match=True),
    ]
```

## 给片段打上标签

将标签添加到片段，与将标签添加到页面非常类似。唯一差别在于应在`ClusterTaggableManager`处使用`taggit.manager.TaggableManager`。

```python
from modelcluster.fields import ParentalKey
from modelcluster.models import ClusterableModel
from taggit.models import TaggedItemBase
from taggit.managers import TaggableManager

class AvertTag(TaggedItemBase):
    content_object = ParentalKey('demo.Advert', on_delete=models.CASCADE, related_name='taggged_items')

@register_snippet
class Advert(ClusterableModel):
    
    ...
    tags = TaggableManager(through=AdvertTag, blank=True)

    panels = [
        ...
        FieldPanel('tags'),
    ]
```

关于更多有关在视图中使用标签的知识，请参阅[给页面打标签的文档](reference/pages.html#tagging)。
