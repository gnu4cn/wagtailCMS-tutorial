# 在模板中使用图片

标签`image`将一个兼容XHTML的`img`元素，插入到页面中，并设置其`src`、`width`、`height`与`alt`属性。请参阅[对`img`标签的更多控制](#image-tag-alt)。

该标签的语法是这样的：

    {% image [image] [resize-rule] %}

**图片与缩放规则，都必须传递给该模板标签**。

示例：

    {% load wagtailimages_tags %}
    ...

    <!---->
