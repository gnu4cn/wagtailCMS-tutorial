# 嵌入的内容

Wagtail支持从到外部服务提供商，如Youtube或Twitter等上的内容的URLs，生成嵌入代码。默认Wagtail将直接从相关提供商站点，使用 [oEmbed协议](https://oembed.com/) 获取嵌入的代码。

Wagtail有着一个内建的大多数常见服务商的清单，且该清单可通过 [一项设置](#customising-embed-providers) 进行修改。Wagtail还支持使用 [Embedly](#embedly) 与 [定制的嵌入发现器](#custom-embed-finders) 来获取嵌入代码。

<a name="embedding-content-on-your-site"></a>
## 在站点上嵌入代码

对于大多数内容提供商来说，Wagtail的内容嵌入模块应可以直接开箱即用。可使用下列的任意方式来调用此模块：

### 富文本方式

Wagtail的默认富文本编辑器，有着一个允许将嵌入代码放置于富文本中的“媒体”图标。不必做任何设置来开启此特性；只要确保该富文本字段的内容，在魔板中是通过`|richtext`过滤器进行传递的即可，因为该过滤器正式对嵌入代码模块进行调用，而进行获取并将嵌入代码加以嵌入的。

### `StreamField`块的`EmbedBlock`类型

该`EmbedBlock`块类型允许将嵌入代码置于某个`StreamField`中。

比如：

```python
from wagtail.embeds.blocks import EmbedBlock

class MyStreamField(blocks.StreamBlock):
    ...

    embed = EmbedBlock()
```

### `{% embed %}`标签

语法：`{% embed <url> [max_width=<max width>] %}`

可通过传入URL与一个可选的`max_width`参数到`{% embed %}`标签，而将嵌入代码嵌套进某个魔板。

其中的`max_width`参数是在获取嵌入代码时，发送给内容提供商的：

{% raw %}
    
    {% load wagtailembeds_tags %}

    {# 这里嵌入一个Youtube视频 #}
    {% embed 'https://www.youtube.com/watch?v=SJXMTtvCxRo' %}

    {# 此标签也可从某个变量获取URL #}
    {% embed page.video_url %}
{% endraw %}

# 从Python嵌入

还可调用内部的`get_embed`函数，该函数取一个URL字符串，返回一个`Embed`对象（参见下面的模型文档）。其也取一个可选的、在获取嵌入代码时发送到服务提供商处的`max_width`关键字参数。

```python
from wagtail.embeds.embeds import get_embed
from wagtail.embeds.exceptions import EmbedException

try:
    embed = get_embed('https://www.youtube.com/watch?v=5CgB9gwXoxM')

    print(embed.html)

except EmbedException:
    # 无法找到嵌入代码
    pass
```

<a name="configuring-embed-finders"></a>
## 配置嵌入代码的“查找器”

__Configuring embed "finders"__

所谓嵌入代码查找器，是指Wagtail中负责从某个URL产生嵌入代码的那些模块。

嵌入代码查找器是通过使用`WAGTAILEMBEDS_FINDERS`设置，进行配置的。该设置是一个查找器配置的清单，米格查找器配置依次运行，直到其中之一成功返回一个嵌入代码为止：

默认配置为：

```python
WAGTAILEMBEDS_FINDERS = [
    {
        'class': 'wagtail.embeds.finders.oembed'
    }
]
```

### oEmbed（默认的）

该默认嵌入代码发现器使用oEmbed协议，直接从内容提供商处获取嵌入代码。Wagtail有着一个内建的服务提供商清单，此清单默认全部开启。可在下面的链接找到该内容提供商清单：

https://github.com/wagtail/wagtail/blob/master/wagtail/embeds/oembed_providers.py

__对嵌入代码提供商清单的进行定制__

可通过指定在查找器配置中的提供商清单，来限制用到哪些提供商。

比如下面的配置，就只允许来自Vimeo与Youtube的内容才能进行嵌套。同时还加入了一个定制的服务提供商：

```python
from wagtail.embeds.oembed_providers import youtube, vimeo

# 加入一个定制的提供商
# 为了令到嵌入代码可以工作，定制提供商必须支持oEmbed协议。应可以
# 在提供商的文档中找到这些技术细节。
# - `endpoint` 时 Wagtail将调用的 oEmbed 端点
# - `urls` 指定和何种模式

my_custom_provider = {
    'endpoint': 'https://customvideosite.com/oembed',
    'urls': [
        '^http(?:s)?://(?:www\\.)?customvideosite\\.com/[^#?/]+/videos/.+$',
    ]
}

WAGTAILEMBEDS_FINDERS = [
    {
        'class': 'wagtail.embeds.finders.oembed',
        'providers': [youtube, vimeo, my_custom_provider],
    }
]
```

__对单个的提供商进行定制__

可将多个查找器链接在一起（multiple finders can be chained together）。此特性可用于对某个提供商的配置进行定制，而不会影响到其他提供商。

比如下面就是告诉Youtube返回HTTPS形式的视频（对于Youtube必须显式地这样做）：

```python
from wagtail.embeds.oembed_providers import youtube

WAGTAILEMBEDS_FINDERS = [
    # 获取YouTube视频，但要在调用Youtube的oEmbed端点时，将`?scheme=https`放在 `GET` 参数中

    {
        'class': 'wagtail.embeds.finders.oembed',
        'providers': [youtube],
        'options': {'scheme': 'https'}
    }，

    # 以默认方式处理其他所有oEmbed提供商
    {
        'class': 'wagtail.embeds.finders.oembed',
    }
]
```

__Wagtail使用多个嵌入代码查找器的方式__

在有着多个提供商都能处理某个URL时（比如使用上面的配置时的请求的一个Youtube视频），那么最顶上的那个查找器将被选作执行本次请求。

就算选中的查找器并未返回一个嵌入内容，Wagtail也将不会尝试运行其他查找器。

### Embed.ly

Embed.ly是一项并未实现oEmbed协议，但也能够提供站点嵌入内容的付费服务。

他们也提供了一些诸如赋予嵌入内容一种持久外观，与在站点允许视频在不同服务提供商处留存，且需要对这些视频实现定制控件时，一个有用常见视频回访API。

Wagtail具有内建的从Embed.ly获取嵌入代码的支持。可通过将一个使用了`wagtail.embeds.finders.oembed`类，并传入了Embed.ly API秘钥的嵌入代码发现器，加入到`WAGTAILEMBEDS_FINDERS`设置，就可使用该项付费服务了：

```python
WAGTAILEMBEDS_FINDERS = [
    {
        'class': 'wagtail.embeds.finders.embedly',
        'key': '你的embed.ly秘钥'
    }
]
```

### 对嵌入代码发现器类进行定制

可创建一个定制的查找器类，以实现对发现器完全的控制。

下面是一个存根发现器类，可用作其他类的框架（here is a stub finder class that could be used as a skeleton）；请阅读代码中的那些 `docstrings` 来了解各个方法具体干些什么：

```python
from wagtail.embeds.finders.base import EmbedFinder

class ExampleFinder(EmbedFinder):
    def __init__(self, **options):
        pass

    def accept(self, url):

        """
        在本发现器知道如何获取某个URL的嵌入代码时返回 True。
        这样做应不会有任何弊端（尚无到外部服务器的请求）
        """
        pass

    def finder_embed(self, url, max_width=None):

        """
        取用一个URL 与最大宽度参数，并返回一个有个将用于嵌入到站点的内容的信息字典

        此方法就是即将向外部APIs发起请求的部分。
        """

        # TODO: 完成请求

        return {
            'title': '内容的标题',
            'author_name': '内容作者名称',
            'provider_name': '提供商名称 (比如 YouTube, Vimeo, etc)',
            'type': "要么是'photo', 'video', 'link', or 'rich'之一",
            'thumbnail_url': '到缩略图的URL',
            'width': width_in_pixels,
            'height': height_in_pixels,
            'html': "<h2>嵌入的HTML</h2>"
        }
```

在实现了所有这些方法后，就只需将定制的查找器类添加到`WAGTAILEMBEDS_FINDERS`设置即可：

```python
WAGTAILEMBEDS_FINDERS = [
    {
        'class': 'path.to.your.finder.class.here',
        # 所有其他选项，将作为提供给`__init__`方法的 `kwargs` 加以传递
    }
]
```

## `Embed`模型

+ `class wagtail.embeds.models.Embed`

    嵌入代码将只获取一次，并存储在数据库中，故其后对某个嵌入代码的访问，并不会再去调用该嵌入代码的查找器了。

    - `url`

        （文本）
        此嵌入代码的原始内容的URL。

    - `max_width`

        （整数，非空值）
        所请求的最大宽度。

    - `type`

        （文本）
        嵌入代码的类型。该字段可以是 “video”、“photo”、“link”或“rich”。

    - `html`

        （文本）
        应放置在页面商的嵌入代码的HTML内容。

    - `title`

        （文本）
        所嵌入内容的标题。

    - `author_name`

        （文本）
        所嵌入内容的作者名称。

    - `provider_name`

        （文本）
        所嵌入内容的提供商名称。
        比如： Youtube, Vimeo等

    - `thumbnail_url`

        （文本）
        一个到所嵌入内容的缩略图的URL。

    - `width`

        （整数，非空值）
        嵌入代码的宽度（仅适用于图片与视频）。

    - `height`

        （整数，非空值）
        嵌入代码的高度（仅适用于图片与视频）。

    - `last_updated`

        （`datetime`值）
        此嵌入代码上一次获取到的日期/时间。


## 删除嵌入代码

只要嵌入代码方面的配置未被破坏，那么删除`Embed`模型中的条目将很安全地完成。Wagtail将自动重新生成站点商正在使用的那些记录。

之所以要删除嵌入代码，无非是要从 oEmbed 协议转到 Embed.ly服务，或反过来，因为他们二者所生成的代码略有不同，从而导致站点上前后矛盾。
