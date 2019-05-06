# 富文本内部元素

初一看Wagtail的富文本功能，貌似给到了站点编辑对一块HTML内容的完全控制。而事实上，给予站点编辑一个相对最终的HTML输出，已移除了多个步骤的富文本内容的一种表示，是有必要的，理由如下：

+ 编辑界面需要将那些不期望的标签过滤掉；这些不期望的标签包括恶意脚本、从外部字处理软件粘贴来的字体样式，以及那些可能破坏站点设计的有效性或一致性的元素（比如，通常页面会保留`<h1>`作为页面标题，因此允许用户经由富文本而插入他们自己额外的`<h1>`元素就不恰当了）

+ 富文本字段可指定`features`参数，来进一步限制在该字段中允许使用的元素（参见 [限制富文本字段中的特性](page_editing_interface.md#rich-text-feature)）

+ 强制使用HTML的子集，有助于将呈现类标记符号排除在数据库之外，从而令到站点更易于维护，且令到对站点内容的重新规划更为容易（包括潜在的生成诸如 [LaTeX](https://www.latex-project.org/) 这类非HTML的输出）

+ 诸如页面链接及图片这样的元素，需要保留像是页面或图片ID这类元数据，这些元数据在最终HTML呈现中是不会出现的

这就要求该富文本内容经历一系列的验证与转换步骤；这些验证与转换步骤，不仅存在于编辑界面与数据库中保存的版本之间，也存在于数据库的表示与最终渲染出的HTML之间。

因为这个原因，对Wagtail的富文本处理过程进行扩展，以支持到新的元素，远非简单地讲一句“开启`<blockquote>`元素”（举个例子）那么简单，因为Wagtail的多个组件 -- 同时涉及客户端与服务端 --都需要就如何处理那个特性达成一致，包括其如何暴露在编辑界面中、在数据库中应如何表示，以及在前端渲染时应如何被翻译出来（在特性适合渲染时）。

Wagtail中富文本处理过程涉及到的组件有下面这些。

## 数据格式

富文本数据（由 [`RichTextField`](page_editing_interface.md#rich-text) 进行处理，而在 [`StreamField`](https://wagtail.xfoss.com/topics/streamfield.html) 中则是由 `RichTextBlock`进行处理），是以类似于HTML，但并不完全相同的方式，存储于数据库中的。比如到某个页面的链接，可能被存储为：

```html
<p><a linktype="page" id="3">联络我们</a> 获取更多信息。</p>
```

这里的`linktype`属性，表示了对该标签进行重写应使用的规则。在经由`|richtext`过滤器（参见 [富文本（过滤器）](https://wagtail.xfoss.com/topics/writing_templates.html#rich-text-filter)）在某个模板上进行渲染时，这就会被转换成为一个有效的HTML:

```html
<p><a href="/contact-us/">联系我们</a> 获取更多信息。</p>
```

在`RichTextBlock`情况下，块的值为一个`RichText`对象，在作为一个字符串而渲染时，将自动完成此转换，因此过滤器`|richtext`就无必要了。

与此类似，富文本内容中的某个图片，将像下面这样存储起来：

```html
<embed embedtype="image" id="10" alt="A pied wagtail" format="left" />
```

渲染时这将渲染为一个`<img>`元素：

```html
<img alt="A pied wagtail" class="richtext-image left" height="294" src="media/images/pied-wagtail.width-500_ENyKffb.jpg" width="500" />
```

再次，这里的`embedtype`属性表示了重写该标签所用的规则。除了`<a linktype="...">`与`<embed embedtype="..." />`之外的所有其他标签，在转换后的HTML中都将保持不变。

有着一些应用到`<a linktype="...">`与`<embed embedtype="..." />`的约束，他们的作用是实现经由字符串替换，而有效地完成转换：

+ 标签名称与属性必须是小写的
+ 属性值必须用双引号括起来
+ `embed`元素必须使用XML的自闭标签语法（XML self-closing tag syntax, 也就是以`/>`，而非`</embed>`结束）
+ 属性值中仅允许这些HTML实体符号：`&lt;`、`&gt;`、`&amp;`与`&quot;`

## 特性注册

项目中的所有应用，都可定义对Wagtail的富文本处理过程的扩展，比如新的`linktype`与`embedtype`规则。名为 _特性注册_ 的对象，是作为有关富文本应如何运作的核心事实来源而加以提供的（An object known as the _feature registry_ serves as a central source of truth about how rich text should behave）。该对象可通过`register_rich_text_features`钩子进行访问，而该钩子是在启动时进行调用的，调用时会收集与富文本有关的所有定义：

```python
# my_app/wagtail_hooks.py

from wagtail.core import hooks

@hooks.register('register_rich_text_features')
def register_my_feature(features):
    # 这里将新的定义，添加到 “features”
```

## 关于重写处理器

重写处理器，是一些知道如何将富文本标签的内容，诸如`<a linktype="...">`与`<embed embedtype="..." />`，转换为前端HTML的类。比如`PageLinkHandler`类，就知道怎样将富文本标签`<a linktype="page" id="123">`，转换为HTML标签`<a href="/path/to/page/123">`。

重写处理器还能提供到其他一些有关富文本标签的信息。比如对于一个给定的适当标签，`PageLinkHandler`就可用于提取出将引用哪个页面的信息。这对于可能需要那些富文本中所引用对象信息的下游代码，将是有用的。

可创建对之前定制的新`linktype`与`embedtype`进行支持的重写处理器。新的处理器必须是继承自`wagtail.core.richtext.LinkHandler`或`wagtail.core.richtext.EmbedHandler`的Python类。新的类应至少对下面的部分方法进行重写（这里列出的时`LinkHandler`的方法，不过`EmbedHandler`的这些方法也有着相同的签名）：

[`class LinkHandler`](#LinkHandler)

    + `identifier`
        
        必需的。`identifier`属性是一个表示哪个富文本标签应由该处理器进行处理的字符串。
        比如`PageLinkHandler.get_identifier`返回的字符串就是`"page"`，表明所有有着`<a linktype="page">`的富文本标签，都将由其加以处理。

    + `expand_db_attributes(attrs)`
    + `get_model()`
    + `get_instance(attrs)`
