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
    {% image page.photo height-480 %}
    {% endraw %}

将图片的高度，缩放到所指定的尺寸。
