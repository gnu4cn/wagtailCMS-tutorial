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

额外的`request.`也是可用的，该对象包含了Django的请求对象。

<a name="static_files"></a>
## 静态文件
