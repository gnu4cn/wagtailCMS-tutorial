# 在模板中使用图片

标签`image`将一个兼容XHTML的`img`元素，插入到页面中，并设置其`src`、`width`、`height`与`alt`属性。请参阅[对`img`标签的更多控制](#image-tag-alt)。

该标签的语法是这样的：

    {% image [image] [resize-rule] %}

**图片与缩放规则，都必须传递给该模板标签**。

示例：

{% raw %}
    {% load wagtailimages_tags %}
    ...

    <!-- 显示一张缩放到 400 像素宽度的图片 -->
    {% image page.photo width-400 %}

    <!-- 再次显示该图片，但这次作为方形的缩略图 -->
    {% image page.photo fill-80x80 %}
{% endraw %}

在上面的语法示例中，`[image]` 是一个对图片的Django对象引用。在页面模型定义了一个名为“photo”的字段时，那么`[image]`就会是`page.phoho`。其中的`[resize-rule]`定义了图片在插入到页面时，将被怎样进行缩放。支持的缩放方式有数种，分别用于不同的用例（如首页上横跨整个页面的头图，或被裁剪为固定尺寸的缩略图）。

请注意有一个空格将`[image]`与`[resize-rule]`隔开，但缩放规则却不能包含空格。缩放规则中的图片宽度，始终是在高度之前。在没有使用`fill`规则时，被缩放的图片将保持其原始宽高比，`fill`规则将导致图片中的某些像素被裁剪。

下面是可用的图片调整的一些方式：

+ `max`
    
（要取两个尺寸值）

    {% raw %}
    {% iamge page.photo max-1000x500 %}
    {% endraw %}


调整到给定的尺寸 **里面**。

图片四个边中最长的那个边，将降低到所指定的匹配尺寸。比如一张宽为 `1000` 高为 `2000`的纵向图片，在以`max-1000x500`规则（一个横向布局）进行处理时，将生成一张被压缩的图片，那么最后的 *高度* 就是 `500` 像素，宽度为 `250` 像素了。

![图片的 `max` 过滤器](images/image_filter_max.png)

*示例：图片将保持其宽高比，但将被缩放到所能提供的最大尺寸（Example: The image will keep its proportions but fit within the max(green line) dimensions provided）*


+ `min`

（要取两个尺寸值）

    {% raw %}
    {% image page.photo min-500x200 %}
    {% endraw %}

**覆盖** 所给定的尺寸。

此图片过滤器将导致图片些许 **大于** 所制定的尺寸。对于一个方形的宽为 `2000` 高为`2000`的图片，在以`min-500x200`规则下，将令到其高度与宽度均为 `500` 像素，也就是要匹配调整规则的 *宽度*，但要所得图片高度要大于规则中的高度。

![图片过滤器 `min`](images/image_filter_min.png)

*示例：图片将保持其宽高比，但将缩放到所能提供的最小尺寸（Example: The image will keep its proportions while filling at least the min(green line) dimensions provided）*

+ `width`
    
（只取一个尺寸值）

    {% raw %}
    {% image page.photo width-640 %}
    {% endraw %}

将图片的宽度，缩放到所指定的尺寸。

+ `height`

（只取一个尺寸值）

    {% raw %}
    {% image page.photo height-480 %}
    {% endraw %}

将图片的高度，缩放到所指定的尺寸。

+ `scale`

（取一个百分数）

    {% raw %}
    {% image page.photo scale-50 %}
    {% endraw %}

将图片的高度，缩放到所指定的百分比。

+ `fill`

（取两个尺寸值，以及一个可选的`-c`参数）

    {% raw %}
    {% image page.photo fille-200x200 %}
    {% endraw %}

将图片进行调整及 **裁剪**，以填充到所给定的精确尺寸。

此图片过滤器在那些需要任意图片的方形缩略图的网站上，尤为有用。比如对于一张宽度为 `2000` 高度为 `1000`的图片，在以 `fill-200x200`规则进行处理是，将把其高度缩小为 `200`，随后在将其宽度（此时为`400`）缩小到 `200`。

在设置了图片焦点时，该调整规则将裁剪到图片的焦点。在没有设置焦点是，将默认裁剪到图片的中心点。

![图片过滤器`fill`](images/image_filter_fill.png)

*示例：图片将先被缩放，并被裁剪（红线），以取得所提供的尺寸下的尽可能大的区域（Example: The image is scaled and also cropped(red line) to fit as much of the image as possible within the provided dimensions）*

### 对于无法进行放大的图片

有可能带有`fill`尺寸下所请求的图片在没有放大的情况下，无法支持。比如一张带有 `fill-400x400 `图片过滤器请求的宽度 `400`高度`200`图片。在这种情况下， 将匹配 *所请求的`fill`的宽高比*，但尺寸将不再匹配。因此示例的 `400x200` 图片（比例为 `2：1`），将变为 `200x200`（比例为 `1：1`，匹配了该调整规则）。

### 更接近于焦点进行裁剪

默认下，Wagtail将改变图片的宽高比，到足够匹配到所给的调整规则的比列方式，进行裁剪。

但在某些情况下（比如缩略图），就更希望以更接近与图片焦点的方式，进行裁剪，以使得图片的主题更为突出一些。

那么可通过将`-c<percentage>`追加到调整规则的后面，实现此目的。比如可通过加入`-c100`，而将图片以更可能靠近其焦点的方式，进行裁剪：

    {% raw %}
    {% image page.photo fille-200x200-c100 %}
    {% endraw %}

这就会将图片以尽可能多的方式进行裁剪，而不会裁剪到焦点中。

如发现 `-c100` 过于靠近焦点，那么可尝试 `-c75` 或 `-c50`。`-c`可接受任何从 `0` 到 `100` 的整数。

![有焦点的图片`fill`过滤器](images/image_filter_fill_focal.png)

*示例：焦点被设置为了偏离中心点，因此该图片被缩放了，同时像`fill`那样进行了裁剪，但裁剪的中心点已被防止在了靠近焦点的地方（Example: The focal point is set off centre so the image is scaled and also cropped like fill, however the center point of the crop is positioned closer the focal point）*

![带有`-c75` 的 `close` 参数的焦点图片`fill`过滤器](images/image_filter_fill_focal_close.png)

*示例：在设置了 `-c75` 之后，最终裁剪将更靠近焦点（Example: With `-c75` set, the final crop will be closer to the focal point）*

+ `original`

（不取尺寸）


    {% raw %}
    {% image page.photo original %}
    {% endraw %}

将图片以其原始尺寸渲染出来。

> **注意** Wagtail并不允许对图片做变形或拉伸处理。图片尺寸比例将始终保持不变。Wagtail 也 *不支持图片的放大*。若强制以较大尺寸来显示较小的图片，那么将以其原生尺寸进行 “最大化（max out）”。

<a name="more-control-over-the-img-tag"></a>
## 对`img`标签的更多控制

Wagtail提供了两个赋予到对`img`元素更多控制的捷径：

### 1. 给 `{% image %}` 标签加上属性

可使用`attribute="value"` 语法，来制定一些额外的属性：

```html
{% image page.photo width-400 class="foo" id="bar" %}
```

通过此种方法，可设置更具相关性的 `alt` 属性，来覆写自动从图片标题生成的那个`alt`值。在必要的情况下，图片的 `src`、`width`及`height`都可以进行覆写。

### 2. 从图片生成图片模板变量，从而访问其诸多属性

**Generating the image "as foo" to access individual properties**

```html
{% image page.photo width-400 as tmp_photo %}

<img src="{{ tmp_photo.url }}" width="{{ tmp_photo.width }}" 
    height="{{ tmp_photo.height }}" alt="{{ tmp_photo.alt }}" class="my-custom-class" />
```

此种语法将所采用的图片转写（the underlying image Redition, `tmp_photo`），暴露给开发者。而“转写”则包含了所请求的、使用调整规则对图片进行格式化的特定信息，也就是尺寸与图片的源URL。

在站点使用 `AstractImage` 定义了一种定制图片模型时，那么添加到某个图片的所有额外字段（比如版权所有者），都 **不会** 包含在转写中。

因此就要将一个 `author` 字段，加入到上面示例中的 `AbstractImage`，而通过`{{ page.photo.author }}`，而非`{{ tmp_photo.author }}` 来访问该字段。

（由于转写图片与其父图片在数据库中的链接的原因，也 **可** 以`{{ tmp_photo.image.author }}`来访问该字段，但这样降低了代码的可读性）

> **注意** 图片用于`img`元素 `src` 属性的对象属性，实际上是`image.url`，而非`image.src`。


## 关于 `attrs` 捷径

**The `attrs` shortcut**

也可使用`attrs` 属性，作为一并输出`src`、`width`、`height`与`alt`等属性的快捷方式：

```html
<img {{ tmp_photo.attrs }} class="my-custom-class" />
```

## 嵌入到富文本中的图片

以上的信息，与经由模型中定义的、特定于图片的字段有关。然而图片还可以通过页面编辑器，嵌入到富文本中的任意位置（参考[富文本（HTML）](advanced_topics/customisation/page_editing_interface.html#rich-text)）。

嵌入到富文本中的图片，就不能由模板开发者所轻易控制到了。对于这样的图片，没有可供处理的图片对象，因此不能使用`{{ image }}`模板标签。而是由站点编辑，在要文本编辑框中插入图片的地方，从一些图片“格式”中选取图片。

Wagtail带有三种预定义的图片格式，但开发者可以Python方式，定义更多的图片格式。这三种格式如下：

+ `Full width`
    
    该格式使用`width-800`创建一个图片的撰写，并赋予了该格式下`<img>`标签 `full-width` 的CSS类

+ `Left-aligned`

    该格式使用`width-500`创建一个图片的转写，并赋予了该格式下`<img>`标签 `left` 的CSS类

+ `Right-aligned`
    

    该格式使用`width-500`创建一个图片的撰写，并赋予了该格式下`<img>`标签 `right` 的CSS类

> **注意** 所添加到图片的CSS类， **并未** 相应样式表，或内联样式与其对应。比如默认 `left` 类就不会生效。开发者需要将这些类加入到他们的前端 CSS文件中，以准确地定义他们想要`left`、`right`或`full-width`意味着什么。

有关更多有关图片格式的信息，包括如何建立自己的图片格式，请参阅[富文本中的图片格式](advanced_topics/customisation/page_editing_interface.html#rich-text-image-formats)。


## 输出图片的格式

在对图片加以处理时，Wagtail可能会自动地改变某些图片的格式：

+ PNG与JPEG的图片不会改变格式
+ 不带有动画的GIF图片，被装换成PNG格式
+ BMP的图片，被转换成PNG格式

也可通过在调整规则之后，使用`format`过滤器，在每个图片基础上，覆写输出格式的设定。

比如使用`format-jpeg`来令到`image`标签总是将图片转换成JPEG:

```html
{% image page.photo width-400 format-jpeg %}
```

也可使用`format-png`或`format-gif`。


## 图片背景色

PNG与GIF图片两种格式，都支持透明，但在打算将图片转换成JPEG格式时，透明部分就需要使用某种固定的背景色来替换。

默认下Wagtail会将背景色设为白色。但如果白色背景不能满足设计是，可使用`bgcolor`过滤器，指定某种颜色。

此过滤器仅取一个参数，为CSS的3位或6位的十六进制代码，表示想要使用的颜色：

```html
{# 将图片背景色设为黑色 #}
{% image page.photo width-400 bgcolor-000 format-jpeg %}
```

## JPEG图片的质量

Wagtail的JPEG图片质量默认设置为了 `85`（已经相当高了）。对此可以全局地修改或基于各个标签进行修改。

### 全局修改JPEG图片质量

在 `settings.py`中使用`WAGTAILIMAGES_JPEG_QUALITY`设置，来改变全局默认JPEG质量：

```python
# settings.py

# 生成低质量但更小的图片
WAGTAILIMAGES_JPEG_QUALITY = 40
```

请注意此种修改不会影响所有先前已经生成的图片，因此可能要删除所有图片的转写，从而以新的设置重新生成这些图片转写。而这可以从Django的命令行完成：

```bash
# 如使用了其他定制的图片转写模型，那么就要使用该图片转写模型
>>> from wagtail.images.models Rendition
>>> Rendition.objects.all().delete()
```

### 依各个标签进行修改

通过使用`jpegquality`过滤器，也可在单个标签基础上，有着不同的JPEG质量。这样总是会覆写默认设置：

```html
{% image page.photo width-400 jpegquality-40 %}
```

请注意这不会在PNG或GIF文件上生效。在想要所有文件都成为低质量，就要将该过滤器与`format-jpeg`一起使用（这强制将所有图片，输出为JPEG格式）。

```html
{% image page.photo width-400 format-jpeg jpegquality-40 %}
```

## 以Python方式生成图片的转写

上面提到所有的图片转换，都可以直接以Python代码方式使用。请参阅[以Python方式生成图片](advanced_topics/images/renditions.html#image-renditions)。
