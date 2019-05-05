# 编辑界面的定制

## 分页界面的定制

标准情况下，Wagtail将页面的面板，组织为三个分页：“内容”、“Promote”与“设置”。对于内容片段，Wagtail则是将所有面板放到一个分页中。依据站点的需求，可能希望对特定页面或内容片段下的此种默认做法进行定制 -- 比如为侧边栏内容而添加一个额外的分页。这可通过在页面或内容片段模型上指定一个`edit_handler`属性来实现。比如：

```python
from wagtail.admin.edit_handler import TabbedInterface, ObjectList

class BlogPage(Page):
    # 此处省略了字段定义部分

    content_panels = [
        FieldPanel('title', classname="full title"),
        FieldPanel('data'),
        FieldPanel('body', classname="full"),
    ]

    sidebar_content_panels = [
        SnippetChooserPanel('advert'),
        InlinePanel('related_links', label="相关链接")
    ]

    edit_handler = TabbedInterface([
        ObjectList(content_panels, heading="内容"),
        ObjectList(sidebar_content_panels, heading="侧边栏内容"),
        ObjectList(Page.promote_panels, heading="Promote"),
        ObjectList(Page.settings_panels, heading="设置项", classname="settings"),
    ])
```


## 关于富文本（HTML）

Wagtail提供了一个通用目的、用于创建富文本内容（HTML）及将诸如图片、视频与文档等媒体文件加以嵌入的所见即所得编辑器。在定义某个模型字段时，使用`RichTextField`函数，就可以将该编辑器包含到模型中：

```python
from wagtail.core.fields import RichTextField
from wagtail.admin.edit_handlers import FieldPanel

class BookPage(Page):
    book_text = RichTextField()

    content_panels = Page.content_panels + [
        FieldPanel('body', classname="full"),
    ]
```

`RichTextField`继承了Django的基本`TextField`字段，因此可像使用普通Django字段那样，将任意字段参数传入到`RichTextField`。该字段并不需要特别的面板，同时可使用`FieldPanel`进行定义。

但`RichTextField`的模板输出较为特殊，而需要加以过滤以保留所嵌入的内容。关于这个问题，请参阅[富文本（过滤器）](https://wagtail.xfoss.com/topics/writing_templates.html#rich-text-filter)

## 生成表单的定制

