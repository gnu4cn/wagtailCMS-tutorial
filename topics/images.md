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

