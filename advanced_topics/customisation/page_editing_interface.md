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

## 生成表单的定制

