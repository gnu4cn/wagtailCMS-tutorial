# 动态的图片提供视图

__Dynamic image serve view__

在多数场合，想要在Python中生成图片副本的开发者，都应使用`get_rendition()`方法，请参考[以Python方式生成图片转写](renditions.md)。

在需要为某种诸如博客或移动应用这样的 _外部_ 系统生成图片的一些版本时，Wagtail提供了一种通过调用独特的URL，来动态生成图片副本的视图。

此种视图获取到URL中的图片id、过滤器规范与安全性签名。在这些参数为有效的情况下，将提供一个符合条件的图片文件。

与`{% image %}`标签一样，该图片副本是在第一次调用时生成的，后续调用将从缓存中予以提供。

## 设置

将该视图的一个条目，添加到站点URLs配置中：

```python
# app/urls.py

from wagtail.images.views.serve import ServeView

urlpatterns = [
    ...

    url(r'^images/([^/]*)/(\d*)/([^/]*)/[^/]*$', ServeView.as_view(), name='wagtailimages_serve'),

    ...

    # 确保该 wagtailimages_serve 行，出现在默认Wagtail页面服务路由之上
    # Ensure that the wagtailimages_serve line appears above the default Wagtail page serving route

    url(r'', include(wagtail_urls)),
]
```

## 用法

### 图片URL生成器的UI

在开启了动态提供图片特性之后，在管理界面中的图片URL生成器将自动变为可用。可经由任意图片的编辑页面，通过点击右侧的“URL生成器”按钮，访问到图片URL生成器。

该界面令到站点编辑可以生成到图片裁剪版本的URLs。

### 以Python方式，生成动态的图片URLs

也可使用Python代码来生成动态的图片URLs，并经由API加以提供，或直接在模板中进行使用。

在模板中使用动态的图片URLs的一个优势，就是在渲染时不会像`{% image %}`标签那样阻塞到初次响应。

`wagtail.images.views.serve`中的`generate_image_url`函数，是生成动态的图片URL的一种方便的方法。

下面是在某个视图中该函数的使用示例：

```python
def display_image(request, image_id):
    image = get_object_or_404(Image, id=image_id):

    return render(request, 'display_image.html', {
        'image_url': generated_image_url(image, 'fill-100x100')
    })
```

对图片的操作，可通过将这些操作以`|`字符连接起来的方式，链接起来：

```python
return render(request, 'display_image.html', {
    'image_url': generate_image_url(image, 'fill-100x100|jpegquality-40')
})
```

在模板中：

{% raw %}

    {% load wagtailimages_tags %}
    ...

    <!-- 获取所放到宽度400像素的图片的url -->
    {% image_url page.photo "width-400" %}

    <!-- 再获取一次，但这次作为一个方形的缩略图： -->
    {% image_url page.photo "fill-100x100|jpegquality-40" %}

    <!-- 这次使用定制的图片提供视图 -->
    {% image_url page.photo "width-400" "mycustomview_serve" %}
{% endraw %}

可传递一个将用于经由其进行图片提供图片的可选视图名称。默认的视图名称为`wagtailimages_serve`。
