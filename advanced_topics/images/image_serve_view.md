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

可传递一个将用于经由其提供图片的可选视图名称。默认的视图名称为`wagtailimages_serve`。

## 高级配置

### 令到视图重定向而非直接提供

__Making the view redirect instead of serve__

默认下该视图将直接提供图片文件。此行为可修改为一个`301`重定向，在图片留存于外部主机时，这样做会有用处。

要开启重定向，就要在urls配置中，将`action='redirect'`传递给`ServeView.as_view()`方法：

```python
from wagtail.images.views.serve import ServeView

urlpatterns = [
    ...

    url(r'^images/([^/*])/(\d*)/[^/*]/[^/]*$', ServeView.as_view(action='redirect'), name='wagtailimages_serve'),
]
```

### 与`django-sendfile`的集成

[`django-sendfile`](https://github.com/johnsensible/django-sendfile) 可将图片数据传输工作交给web服务器，而不是直接由Django应用直接提供图片数据。在站点有着很多同时被下载的图片，却又无法使用[带有缓存的代理服务器](performance.md#caching-proxy)或CDN的情况下，这样做可极大地减低服务器负载。

首先要安装和配置好`django-sendfile`，并将web服务器配置好使用`django-sendfile`。如尚未准备好这些，请参考该[安装文档](https://github.com/johnsensible/django-sendfile#django-sendfile)。

可使用`SenfFileView`类，来以`django-sendfile`方式进行图片的提供。此视图可开箱即用：

```python
from wagtail.images.views.serve import SendFileView

urlpatterns = [
    ...

    url(r'^images/([^/]*)/(\d*)/([^/]*)/[^/]*$', SendFileView.as_view(), name='wagtailimages_serve'),
]
```

可对其加以定制，以对`SENDFILE_BACKEND`中定义的后端进行覆写：

```python
from wagtail.images.views.serve import SendFileView
from project.sendfile_backends import MyCustomBackend

class MySendFileView(SendFileView):
    backend = MyCustomBackend
```

还可将其定制为发送私有文件。比如在要求仅为需为认证用户（例如对于Django >= 1.9）：

```python
from django.contrib.auth.mixins import LoginRequiredMixin
from wagtail.images.views.serve import SendFileView

class PrivateSendFileView(LoginRequiredMixin, SendFileView):
    raise_exception = True
```
