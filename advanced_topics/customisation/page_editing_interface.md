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

但`RichTextField`的模板输出较为特殊，而需要加以过滤以保留所嵌入的内容。关于这个问题，请参阅[富文本（过滤器）](https://wagtail.xfoss.com/topics/writing_templates.html#rich-text-filter)。

### 对富文本字段中的特性加以限制

默认该富文本编辑器提供给用户相当多的文本格式化与插入诸如图片等嵌入式内容的选项。然而可能希望将某个富文本字段限制到更为受限的特性集合 -- 比如：

+ 该字段可能打算作为短文本的内容片段，比如将在索引页面上拉出来的一个概要，那样的话嵌入图片或视频就不适当了；

+ 在页面内容是通过使用 [`StreamField`](https://wagtail.xfoss.com/topics/streamfield.html#streamfield)进行定义的时，诸如标题、图片及视频等元素，就通常赋予了其自身的块类型，同时为普通段落文本使用某种富文本的块类型；在此情况下，再允许标题及图片存在于富文本内容，就会显得累赘（并导致设计上无法保持一致）。

可通过将带有希望使用的特性标识符的清单的一个`features`关键字参数，传递给`RichTextField`函数，来完成富文本字段特性的限制：

```python
body = RichTextField(features=['h2', 'h3', 'bold', 'italic', 'link'])
```

默认Wagtail安装所提供了以下这些特性标识符：

+ `h1`、`h2`、`h3`、`h4`、`h5`、`h6` -- 标题元素
+ `bold`、`italic` -- 粗体/斜体文本
+ `ol`、`ul` -- 有序/无序列表
+ `hr` -- 水平线
+ `link` -- 页面、外部网页与电子邮件链接
+ `document-link` -- 到文档的链接
+ `image` -- 嵌入图片
+ `embed` -- 嵌入媒体（请参阅 [嵌入式内容](https://wagtail.xfoss.com/advanced_topics/embeds.html#embedded-content)）

还有几个额外的特性标识符。他们默认没有开启，但可在标识符清单中使用他们。这些额外标识符如下所示：

+ `code` -- 内联代码
+ `superscript`、`subscript`、`strikethrough` -- 文本格式化
+ `blockquote` -- 引用块（blockquote）

以下页面对创建新特性的流程，进行了讲解：

+ [富文本的内部元素](rich_text_internals.md)
+ [对Draftail编辑器进行扩展](extending_draftail.md)
+ [对Hallo编辑器进行扩展](extending_hallo.md)

### 富文本编辑器中的图片格式问题

在Wagtail加载时，他将搜寻任何带有`image_formats.py`文件的应用，并执行其中的代码。这就提供了一种在`RichTextField`中插入图片时，对暴露给编辑器的格式选项进行定制的途径。

比如加入一个“缩略图”的格式：

```python
# image_formats.py

from wagtail.images.formats import Format, register_image_format

register_image_format(Format('thumbnail', 'Thumbnail', 'richtext-image thumbnail', 'max-120x120'))
```

这里是以`Format`类、`register_image_format`函数，以及可选的`unregister_image_format`函数的导入开始的。要注册一个新的`Format`，就要以`Format`对象作为参数，调用`register_image_format`。类`Format`则要取以下的构造器参数：

+ `name`

    用于标识该格式的唯一性键。在解除此格式的注册时，就要以这个字符串，作为唯一的参数来调用`unregister_image_format`。

+ `label`

    将图片插入到`RichTextField`中时，在选择器表单中使用的标签。

+ `classnames`

    指派给所生成的`<img>`标签的`class`属性的字符串。

    > __注意__ 所提供的任何CSS类名称，都必须有着单独编写的CSS规则，作为前端CSS代码的一部分与其匹配。指定一个`left`值的`classnames`，将只能确保在生成的标记语言代码中输出一个`left`的CSS类，而不会导致该图片左对齐。
    

+ `filter_spec`

    用于创建图片副本的字符串规格。更多信息，请参考[在模板中使用图片](https://wagtail.xfoss.com/topics/images.html#image-tag)。

要解除注册，使用该`Format`的`name`字符串作为唯一参数，调用`unregister_image_format`即可。

> __注意__ 解除`Format`对象的注册，将导致那些对这些`Format`进行应用的查看或编辑页面报出错误。


## 生成表单的定制


[`class wagtail.admin.forms.WagtailAdminModelForm`](#wagtail.admin.forms.WagtailAdminModelForm)

[`class wagtail.admin.forms.WagtailAdminPageForm`](#wagtail.admin.forms.WagtailAdminPageForm)

Wagtail使用模型上所配置的面板，来自动生成表单。默认该表单对`WagtailAdminModelForm`，或页面的`WagtailAdminPageForm`进行子类化。可通过对所有模型上的`base_form_class`属性的设置，来配置定制基本表单类。内容片段的定制表单，必须是对`WagtailAdminModelForm`的子类化，而页面的定制表单，则必须是`WagtailAdminPageForm`的子类化。

此特性可用于将非模型字段，添加到表单，从而自动生成字段内容，或为模型加入定制的验证逻辑：

```python
from django import forms
import geocoder # 该库不是Wagtail中的，这里仅用于示例 -- http://geocoder.readthedocs.io/
from wagtail.admin.edit_handler import FieldPanel
from wagtail.admin.forms import WagtailAdminPageForm
from wagtail.core.models import Page

class EventPageForm(WagtailAdminPageForm):
    address = form.CharField()

    def clean(self):
        cleaned_data = super().clean()

        # 这里确保活动的开始日期是在结束日期之前
        start_date = cleaned_data['start_date']
        end_date = cleaned_data['end_date']
        if start_date and end_date and start_date > end_date:
            self.add_error('end_date', '结束日期必须是在开始日期之后')

        return cleaned_data


    def save(self, commit=True):
        page = super().save(commit=False)

        # 根据提交的两个日期，对活动时长字段进行更新
        page.duration = (page.end_date = page.start_date).days

        # 通过对地址进行地理计算，来获取定位数据
        page.location = geocoder.arcgis(self.cleaned_date['address'])

        if commit:
            page.save()

        return page

class EventPage(Page):
    start_date = models.DateField()
    end_date = models.DateField()
    duration = models.IntegerField()
    location = models.CharField(max_length=255)

    content_panels = [
        FieldPanel('title'),
        FieldPanel('start_date'),
        FieldPanel('end_date'),
        FieldPanel('address'),
    ]

    base_form_class = EventPageForm
```

Wagtail将生成该模型的表单的一个新的子类，把`panels`与`content_panels`中的定义的所有字段都加入进来。模型上已经定义的所有字段，都不会被这些自动生成添加的字段所覆写，因此某个模型字段的表单字段，可通过将其添加到定制表单而加以覆写（Any fields already defined on the model will not be overridden by these automatically added fields, so the form field for a model field can be overridden by adding it to the custom field）。
