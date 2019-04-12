# 编写模板

Wagtail使用了Django的模板语言。若刚接触Django，请从Django自己的模板文档开始：[模板](https://docs.djangoproject.com/en/stable/topics/templates/)。

对与那些刚接触Django/Wagtail的Python程序员，可能偏好看更具技术性的文档：[Django模板语言：写给Python程序员](https://docs.djangoproject.com/en/stable/ref/templates/api/)。

在继续本文之前，应熟悉Django模板原理方面的基础知识。

<a name="templates"></a>
## 关于模板

在Wagtail中的所有页面或“内容类型”，都是在一个叫做`models.py`的文件中，被定义为某个“模型”的（every type of page or "content type" in Wagtail is defined as a "model" in a file called `models.py`）。当站点有个博客应用时，那么就会有个`BlogPage`模型，以及另一个命名为`BlogPageListing`的模型。模型的名称，是有Django开发者决定的。

对于`models.py`中的每个页面模型，Wagtail都假定存在（多数情况下）一个同样名称的HTML模板文件。前端开发者可能需要自己负责创建出这些模板，他们通过参考`models.py`，从该文件中所定义的模型，来推断出模板的名称。

Wagtail将把驼峰式的模型类名，转换成蛇形名称，以找出一个恰当的模板。因此对于`BlogPage`页面模型，就期望得到一个`blog_page.html`的模板。必要时可覆写模型对应的模板文件名称。

默认模板文件位于此处：

```bash
name_of_project/
    name_of_app/
        templates/
            name_of_app/
                blog_page.html
        models.py
```

有关此方面的更多信息，请参阅Django文档的 [应用目录的模板加载器](https://docs.djangoproject.com/en/stable/ref/templates/api/)

### 关于页面内容

进入到各个页面的数据/内容，是通过Django的 `{{ double-brace }}`表示法来i就那些访问/输出的。模型中的每个字段，都必须通过前缀`page.`来进行访问。比如页面标题`{{ page.title }}`或另一个`{{ page.author }}`字段。

此外，`request.`也是可用的，该对象包含了Django的请求对象。

<a name="static_files"></a>
## 静态文件

像是CSS、JS与图片这样的静态文件，通常存储在这里：

```bash
name_of_project/
    name_of_app/
        static/
            name_of_app/
                css/
                js/
                images/
        models.py
```

（其中的文件夹名`css`、`js`等并不重要，那只是他们在树中的位置）

`static`文件中中的所有文件，都应通过使用`{% static %}`标签，插入到HTML中。更多有关此方面的内容：[静态文件（标签）](#static-tag)。

### 用户的图片

由用户上传到Wagtail站点的图片（与上面提到的开发者的静态文件相反），被放入到图片库中，并通过[页面编辑器界面](editor_manual/new_pages/inserting_images.md)从图片库添加到页面。

与其他内容管理系统不同，将图片添加到页面并不涉及到所使用图片“版本”的选择。Wagtail没有预先定义的图片“格式”或“尺寸”。而是有模板开发者，通过一种模板中的特殊语法，来定义图片操作，这种图片操作，实在图片被请求时， *于操作中* 进行的（Instead the template developer defines image manipulation to occur *on the fly* when the image is requested, via a special syntax within the template）。

图片库中的图片，必须通过这种语法加以请求，但对于开发者的静态图片，则可通过传统方式，比如`img` 标签，予以添加。只有图片库中的图片，才能够 *于操作中* 进行处理。

更过有关这种图片处理语法的信息，请参考：[在模板中使用图片](topics/images.md#image-tag)。


## 模板的标签与过滤器

除了Django的标准标签与过滤器外，Wagtail还其他了一些自己的标签与过滤器，这些标签与过滤器可[如同其他标签与过滤器一样](https://docs.djangoproject.com/en/stable/howto/custom-template-tags/)，通过 `load` 进行加载。


### 图片（标签）

`image`标签，将一个兼容XHTML的`img`元素，插入到页面中，并设置其 `src`、`width`、`height`与`alt`属性。请参阅[对`img`标签的更多控制](topics/images.md#image-tag-alt)。

`image`标签的语法如下：

    {% image [image] [resize-rule] %}

示例：

    {% load wagtailimages_tags %}
    ...

    {% image page.photo width-400 %}

    <!-- 或一个正方形的缩略图： -->
    {% image page.photo file-80x80 %}

请参阅完整文档 [在模板中使用图片](topics/images.md#image-tag)。

### 富文本（过滤器）

此过滤器将一块HTML内容，在页面中作为安全的HTML进行渲染。重要的是他还将HTML块内部的缩略式引用展开为嵌入式图片，以及将那些在Wagtail编辑器中制作的链接，展开为可供显示的恰当的HTML。

在模板中，只有那些使用了`RichTextField`的字段，才需要应用此过滤器。

    {% load wagtailcore_tags %}

    ...
    {% page.body|richtext %}


### 响应式嵌入内容

**Responsive Embeds**

Wagtail在包含嵌入内容与图片时，是以其完整宽度进行嵌入的，这样做可能超出在模板中定义的嵌入内容容器的边界。要让图片与嵌入内容成为响应式的 -- 那意味着这些内容将缩放到适合他们的容器的大小 -- 请将下面的CSS加入到站点样式表中：

```css
.rich-text img {
    max-width: 100%;
    height: auto/
}

.responsive-object {
    position: relative;
}

.responsive-object iframe;
.responsive-object object;
.responsive-object embed {
    position: absolute;
    top: 0;
    left 0;
    width: 100%;
    height: 100%;
}
```

### 内部的链接（标签）

+ `pageurl`

    从某个页面对象，在该页面与当前页面为同一个站点时，返回一个相对的URL（`/foo/bar/`），在不是同一个站点时，返回一个绝对URL（`http://example.com/foo/bar/`）。

        {% raw %}
        {% load wagtailcore_tags %}
        ...
        <a href="{% pageurl page.blog_page %}">
        {% endraw %}

+ `slugurl`

    从页面的“Promote”分页中定义的`slug`，返回一个与该页面匹配的URL。如该别名下存在多个页面，那么就确定不下来所选的页面了。

    与`pageurl`类似，该标签在可能的情况下会提供一个相对链接，在所给页面位于不同站点时，则会默认为一个绝对链接。这在创建共享页面特性时，比如顶层的导航栏，或全站链接时，是最有用的（like `pageurl`, this will try to provide a relative link if possible, but will default to an absolute link if the Page is on a different Site. This is most useful when creating shared page feature, e.g. top level navigation or site-wide links）。

        {% raw %}
        {% load wagtailcore_tags %}
        ...
        <a href="{% slugurl 'news' %}">News index</a>
        {% endraw %}


### 静态文件（标签）

该标签用于从静态文件目录装入任意文件。该标签的使用，避免了在主机环境变化时重写静态路径，因为在开发环境下与上线后这些静态路径可能有所不同。

    {% raw %}
    {% load static %}
    ...
    <img src="{% static "name_of_app/myimage.jpg" %}" alt="My image" />
    {% endraw %}


## Wagtail的用户栏

该标签为已登入用户提供了一个上下文弹出菜单（a contextual flyout menu）。该菜单给网站编辑提供了编辑当前页面或加入一个子页面的能力，以及在Wagtail页面浏览器中显示该页面，或前往Wagtail管理控制台的选项。对于网站主编，也能够通过用户栏，完成对某个提交预览的页面予以通过或撤回的内容审核工作。

    {% load wagtailuserbar %}
    ...
    {% wagtailuserbar %}


默认用户栏从浏览器窗口边缘插入到页面的右下角。如此默认行为与设计有冲突，就可通过传入一个参数到该模板标签，而对其进行移动。下面的示例给出了将用户栏放置于屏幕各个角落的方法：

    {% wagtailuserbar 'top-left' %}
    {% wagtailuserbar 'top-right' %}
    {% wagtailuserbar 'bottom-left' %}
    {% wagtailuserbar 'bottom-right' %}

用户栏也可以放置于任何最有利于设计的地方。此外，可在CSS文件中以CSS规则的方式，来确定他的放置位置，比如：

```css
...
.wagtail-userbar {
    top: 200px !important;
    left: 10px !important;
}
```

## 在内容上线之前经由预览对输出进行检查

有的时候可能希望是否依据页面经由预览或上线查看，来对模板输出进行检查。比如在站点上有着诸如Google Analytics这样的访问者追踪代码时，那么在预览时进行检查就比较好，因为这样做的话，站点编辑的操作，不会出现在分析报告中。Wagtail提供了一个`request.is_preview`的变量，来对预览和上线进行区分：

    {% if not request.is_preview %}
        <script>
            (function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
                ...
            }})
        </script>
    {% endif %}
