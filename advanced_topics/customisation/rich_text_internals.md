# 富文本内部元素

初一看Wagtail的富文本功能，貌似给到了站点编辑对一块HTML内容的完全控制。而事实上，给予站点编辑一个相对最终的HTML输出，已移除了多个步骤的富文本内容的一种表示，是有必要的，理由如下：

+ 编辑界面需要将那些不期望的标签过滤掉；这些不期望的标签包括恶意脚本、从外部字处理软件粘贴来的字体样式，以及那些可能破坏站点设计的有效性或一致性的元素（比如，通常页面会保留`<h1>`作为页面标题，因此允许用户经由富文本而插入他们自己额外的`<h1>`元素就不恰当了）

+ 富文本字段可指定`features`参数，来进一步限制在该字段中允许使用的元素（参见 [限制富文本字段中的功能](page_editing_interface.md#rich-text-feature)）

+ 强制使用HTML的子集，有助于将呈现类标记符号排除在数据库之外，从而令到站点更易于维护，且令到对站点内容的重新规划更为容易（包括潜在的生成诸如 [LaTeX](https://www.latex-project.org/) 这类非HTML的输出）

+ 诸如页面链接及图片这样的元素，需要保留像是页面或图片ID这类元数据，这些元数据在最终HTML呈现中是不会出现的

这就要求该富文本内容经历一系列的验证与转换步骤；这些验证与转换步骤，不仅存在于编辑界面与数据库中保存的版本之间，也存在于数据库的表示与最终渲染出的HTML之间。

因为这个原因，对Wagtail的富文本处理过程进行扩展，以支持到新的元素，远非简单地讲一句“开启`<blockquote>`元素”（举个例子）那么简单，因为Wagtail的多个组件 -- 同时涉及客户端与服务端 --都需要就如何处理那个功能达成一致，包括其如何暴露在编辑界面中、在数据库中应如何表示，以及在前端渲染时应如何被翻译出来（在功能适合渲染时）。

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

## 功能注册

项目中的所有应用，都可定义对Wagtail的富文本处理过程的扩展，比如新的`linktype`与`embedtype`规则。名为 _功能注册_ 的对象，是作为有关富文本应如何运作的核心事实来源而加以提供的（An object known as the _feature registry_ serves as a central source of truth about how rich text should behave）。该对象可通过`register_rich_text_features`钩子进行访问，而该钩子是在启动时进行调用的，调用时会收集与富文本有关的所有定义：

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

    必需的。方法`expand_db_attributes`期望从某个数据库的富文本`<a>`标签（对于`EmbedHandler`就是`<embed>`标签）取得一个属性的字典，并使用该字典来生成有效的前端HTML。
    比如`PageLinkHandler.expand_db_attributes`将收到`{'id': 123}`，将使用其来获取有着 ID `123`的页面，并渲染出一个到该页面URL链接，类似于`<a href="/path/to/page/123">`。

+ `get_model()`

    可选的。该静态`get_model()`方法仅用在那些用来渲染与Django模型有关的内容。此方法允许处理器将那些知道如何进行处理的内容的类型暴露出来（this method allows handlers to expose the type of content that they know how to handle）。
    比如`PageLinkHandler.get_model`将返回Wagtail类`Page`。

    那些与Django模型无关的处理器，可将此方法留为未定义状态，此时如对其进行调用，则将引发`NotImplementedError`错误。

+ `get_instance(attrs)`

    可选的。该静态或类方法（classmethod）`get_instance`同样只应用于那些用来渲染与Django模型有关的处理器。该方法期望获取一个来自数据库的富文本`<a>`标签（或`EmbedHandler`的`<embed>`标签）的属性字典，并使用该字典来返回特定所引用的Django模型实例。

    比如`PageLinkHandler.get_instance`就将收到`{'id': 123}`，并返回Wagtail的有着ID `123`的`Page` 类的实例。

    在该方法未定义时，该方法的一个默认实现，将查询由使用提供的`id`属性下，`get_model`方法所返回的类上的`id`模型字段（if left undefined, a default implementation of this method will query the `id` model field on the class returned by `get_model` using the provided `id` attribute）；在自己的处理器中，如打算使用其他模型字段，则可对此默认行为进行覆写。

以下是一个定制的重写处理器的示例，其中实现了加入对用户邮箱地址的富文本链接支持的上述方法。该重写处理器支持对类似`<a linktype="user" username="wagtail">`这样的富文本标签，到类似`<a href="mailto:hello@wagtail.io">`的有效HTML的转换。此示例假定等价的前端功能已被添加，从而允许用户将这类链接插入到富文本编辑器中。

```python
from django.contrib.auth import get_user_model
from wagtail.core.rich_text import LinkHandler

class UserLinkHandler(LinkHandler):

    identifier = 'user'

    @staticmethod
    def get_model():
        return get_user_model()

    @classmethod
    def get_instance(cls, attrs):
        model = cls.get_model()
        return model.objects.get(username=attrs('username'))

    @classmethod
    def expand_db_attributes(cls, attrs):
        user = cls.get_instance(attrs)
        return '<a href="mailto:%s">' % user.email
```

## 注册重写处理器

重写处理器也必须通过`register_rich_text_features`钩子，使用功能注册而加以注册。对于链接处理器与嵌入处理器的注册，分别提供了独立的方法。

+ [`FeatureRegistry.register_link_type(handler)`](#FeatureRegistry.register_link_type)

    该方法允许对一个派生自`wagtail.core.rich_text.LinkHandler`的定制处理器进行注册，并将其加入到富文本转换期间的链接处理器清单中。

```python
# my_app/wagtail_hooks.py

from wagtail.core import hooks
from my_app.handlers import MyCustomLinkHandler

@hooks.register('register_rich_text_features')
def register_link_handler(features):
    features.register_link_type(MyCustomLinkHandler)
```

虽然Wagtail内建的`external`与`email`链接并没有预定义的`linktype`，但为他们定义链接重写处理器也是可能的。比如在出于搜索引擎优化目的，而想要外部链接有着一个`rel="nofollow"`属性时：

```python
from django.utils.html import escape
from wagtail.core import hooks
from wagtail.core.rich_text import LinkHandler

class NoFollowExternalLinkHandler(LinkHandler):
    identifier = 'external'

    @classmethod
    def expand_db_attributes(cls, attrs):

        href = attrs["href"]
        return '<a href="%s" rel="nofollow">' % escape(href)

@hooks.register('register_rich_text_features')
def register_external_link(features):
    features.register_link_type(NoFollowExternalLinkHandler)
```

与此类似，可使用`email`的链接类型，来加入一个定制的用于邮箱链接的重写处理器（比如用来对富文本中的电邮进行混淆）。


+ [`FeatureRegistry.register_embed_type(handler)`](#FeatureRegistry_register_embed_type)

    此方法允许注册一个派生自`wagtail.core.rich_text.EmbedHandler`的定制处理器，并将其添加到富文本装换期间可用的嵌入处理器清单。

```python
# my_app/wagtail_hooks.py

from wagtail.core import hooks
from my_app.handlers import MyCustomEmbedHandler

@hooks.register('register_rich_text_features')
def register_embed_handler(features):
    features.register_embed_type(MyCustomEmbedHandler)
```

_版本 2.5 中的新功能_：在之前的版本中，`register_link_type`与`register_embed_type`都接受两个参数：一个时链接或嵌入类型标识符，以及一个用于执行重写的函数（与`expand_db_attributes`方法等价）。

## 编辑器小部件

用于富文本字段的编辑器界面，可以`WAGTAILADMIN_RICH_TEXT_EDITORS`设置项而加以配置。Wagtail提供了两个编辑器实现：`wagtail.admin.rich_text.DraftailRichTextArea`（也就是基于 [Draft.js](https://draftjs.org/) 的 [Draftail](https://www.draftail.org/) 编辑器），以及`wagtail.admin.rich_text.HalloRichTextArea`（已放弃，基于 [Hallo.js](http://hallojs.org/)）。

创建自己的富文本编辑器实现是可能的。在最小状态，一个富文本编辑器就是一个Django的`Widget`子类，其构造器接受一个`options`关键字参数（一个特定于编辑器的配置选项的字典，其来源为`WAGTAILADMIN_RICH_TEXT_OPTIONS`中的`OPTIONS`字段），同时其要消费与制造上面所讲到的类HTML格式中的字符串数据（and which consumes and produces string data in the HTML-like format described above）。

通常一个富文本部件也会接收一个自`RichTextField`/`RichTextBlock`或`WAGTAILADMIN_RICH_TEXT_EDITORS`的`features`选项，传递过来的`features`的清单，该清单定义了在该编辑器的那个实例中可用的功能（参见 [对富文本字段中的功能进行限制](page_editing_interface.md#rich-text-features)）。要开启功能支持，就对小部件类上的`accepts_features = True`属性做设置；随后小部件构造器将以关键字参数`features`接收到功能清单。

在 [限制富文本字段中的功能](page_editing_interface.md#rich-text-features) 下列出的一套标准的可识别功能标识符，但这并不是一个最终清单；功能标识符并不仅是依照惯例进行定义的，还取决于各个编辑器小部件而确定那些功能将为这些小部件所识别，并据此适应其行为。单个编辑器部件可能实现相较于默认功能集更少或更多的功能，既可作为内建功能形式，或在编辑器部件有着某种插件机制时，以插件的方式实现。

比如某个第三方的Wagtail扩展要引入`table`作为一种新的富文本元素，并提供了Draftail与Hallo编辑器（二者都提供了插件机制）的实现。在此情况下，该第三方扩展将对定制的编辑器部件一无所知，因此该部件就不知道如何处理`table`功能标识符。编辑器部件将静默地忽略所有他们不能识别的功能标识符。

功能注册表中的`default_features`属性，是一个在`RichTextField`/`RichTextBlock`或`WAGTAILADMIN_RICH_TEXT_EDITORS`中没有给出显式的功能清单时，使用到的功能标识符清单。该清单可在`register_rich_text_features`钩子中加以修改，以令到新的功能得以默认开启，并通过调用`get_default_features()`而获取到。

```python
@hooks.register('register_rich_text_features')
def make_h1_default(features):
    features.default_features.append('h1')
```

在`register_rich_text_features`钩子之外 -- 比如在某个部件类中 -- 该功能注册表可作为 `wagtail.core.rich_text.features` 对象而加以导入。下面是带有功能支持的富文本编辑器的可行起点：

```python
from django.forms import widgets
from wagtail.core.rich_text import features

class CustomRichTextArea(widgets.TextArea):
    accepts_features = True

    def __init__(self, *args, **kwargs):
        self.options = kwargs.pop('options', None)

        self.features = kwargs.pop('features', None)
        if self.features is None:
            self.features = features.get_default_features()

        super().__init__(*args, **kwargs)
```

## 编辑器插件

<a name="FeatureRegistry.register_editor_plugin"></a>
+ `FeatureRegistry.register_editor_plugin(editor_name, feature_name, plugin_definition)`

    富文本编辑器通常提供了某种插件机制，以允许对编辑器扩展出新的功能。`register_editor_plugin`方法就为`register_rich_text_features`钩子提供了一种标准化的方法，用于在某个富文本功能开启时将拉入到编辑器的插件的定义。

    `register_editor_plugin`所传入的编辑器名称（用于对编辑器部件进行标识的一个唯一字符串 -- Wagtail使用`draftail`与`hallo`作为其内建的编辑器）、功能标识符及插件定义对象。该对象时特定于编辑器部件的，并可以为任意值，通常会包含一个引用了该插件的JavaScript代码的 [Django的表单媒体](https://docs.djangoproject.com/en/stable/topics/forms/media/)定义，JavaScript代码随后就将被融入到编辑器部件自己的媒体定义中 -- 以及在编辑器实例化时要进行传递的全部相关配置选项。

<a name="FeatureRegistry.get_editor_plugin"></a>
+ `FeatureRegistry.get_editor_plugin(editor_name, feature_name)`

在编辑器部件内，某个给定功能的插件定义可通过`get_editor_plugin`方法获取到，传入的是编辑器自身的标识符与功能的标识符。在没有已注册的匹配插件时，其将返回`None`。

有关Wagtail内建编辑器插件格式的详细信息，请参见 [对Draftail编辑器进行扩展](extending_draftail.md) 与 [对Hallo编辑器进行扩展](extending_hallo.md)。

## 关于格式转换器

编辑器部件通常无法直接工作于Wagtail的富文本格式上，从而需要转换为其自己的原生格式。对于Draftail，原生格式为一种基于JSON的、叫做 `ContentState`（参见 [Draft.js是如何表示富文本数据的](https://medium.com/@rajaraodv/how-draft-js-represents-rich-text-data-eeabb5f25cf2)）的格式。Hallo.js与其他编辑器则是基于HTML 的`contentEditable`机制，此机制要求有效的HTML，因此Wagtail使用了一种被称为“编辑器HTML”的约定，在这种约定中，链接及嵌入元素所需要的额外数据，保存在`data-`属性中，比如：

```html
<a href="/contact-us" data-linktype="page", data-id="3">联系我们</a>
```

Wagtail提供了两个工具类，`wagtail.admin.rich_text.converters.contentstate.ContentstateConverter`与`wagtail.admin.rich_text.converters.editor_html.EditorHTMLConverter`，用于完成富文本格式与编辑器原生格式之间的转换。这些类是独立于各个编辑器部件的，且在将富文本渲染到模板上时，也是根据所发生的重写过程而有所区别。

两个类都接受一个`features`清单，作为他们的构造器的参数，并实现以下两个方法：方法`from_database_format(data)`将Wagtail的富文本数据转换到编辑器的格式，方法`to_database_format(data)`，将编辑器数据转换到Wagtail的富文本格式。

对于编辑器的插件，转换器类的行为可根据传递给他的功能清单，而有所不同。尤其是他可以执行白名单规则，来确保输出仅包含那些对应于当前活动功能集的HTML元素。功能注册表提供了`register_converter_rule`方法，来允许`register_rich_text_features`钩子，定义出在开启了给定功能时将要激活的转换规则。

<a name="FeatureRegistry.register_converter_rule"></a>
+ `FeatureRegistry.register_converter_rule(converter_name, feature_name, rule_definition)`

传递给`register_converter_plugin`的是一个转换器名称（对转换器类进行唯一标识的一个字符串 -- Wagtail使用了`contentstate`与`editorhtml`两个标识符）、一个功能标识符与一个规则定义对象。该对象是特定于转换器的，且可以是任意值。

有关`contentstate`与`editorhtml`两个转换器的规则定义格式的详细信息，请分别参阅 [对Draftail编辑器进行扩展](extending_draftail.md) 与 [对Hallo编辑器进行扩展](extending_hallo.md)。

<a name="FeatureRegistry.get_converter_rule"></a>
+ `FeatureRegistry.get_converter_rule(converter_name, feature_name)`

在某个转换器类的内部，可通过`get_converter_rule`方法，获取到给定特性的规则定义，传递给该方法的参数为转换器自身的标识符字符串与功能标识符。在匹配的规则未有注册时，该方法将返回`None`。
