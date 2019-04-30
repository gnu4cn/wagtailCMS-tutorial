# 私有页面

有着某个页面发布权限的用户，可通过点击页面浏览器或编辑界面右上角的“私密（Privacy）”控件，将该页面设置为私有页面。这将设置一个仅允许一部分人查看该页面及其子页面的限制。有下面几种类型的限制：

+ __限制为仅登入用户可访问__：要查看该页面，用户需要登入。所有用户都可访问该页面，而与权限级别无关。

+ __限制为输入口令才可访问__：用户必须输入给定的口令，才能查看到该页面。这种方法适用于在打算将某个页面分享给信任的一组人，但又不适合给他们分别创建一个账户的情况。这时所有查看该页面的用户使用的是同一个口令，同时这种做法独立于站点上既有的全部用户账号。

+ __限制为仅对特定分组的用户可访问__：用户必须登入，且时一个或多个指定用户分组中的成员，才能查看到该页面。


与页面类似，文档也可通过将其放入到带有适当隐私设置的集合中，而令其成为私有的（参见：[图片/文档的权限](#image-document-permissions)）。

在Wagtail中，私有页面与文档特性时开箱即用的 -- 站点实现者无需任何设置就可以建立起来。但默认的“登入”与“需要口令”表单，仅是一些基本的HTML页面，站点实现者可能希望将他们替换为适应站点设计的定制页面。

## 设置一个登录页面

通过将`WAGTAIL_FRONTEND_LOGIN_TEMPLATE`设置到希望使用的某个模板的路径，就可对Wagtail自带的基本登录页面进行定制：

```python
# settings.py

WAGTAIL_FRONTEND_LOGIN_TEMPLATE = 'myapp/login.html'
```

这里Wagtail使用了Django标准的`django.contrib.auth.views.LoginView`视图，因此在该模板上的上下文变量，在 [Django的登录视图文档](https://docs.djangoproject.com/en/stable/topics/auth/default/#django.contrib.auth.views.LoginView)中有详细说明。

在该原装的Django登录视图并不适合 -- 比如要使用某种外部认证系统，或将Wagtail集成到某个已有一个工作的登录视图的现有Django站点时 -- 便可通过`WAGTAIL_FRONTEND_LOGIN_URL`设置，指定登录视图的URL：

```python
# settings.py

WAGTAIL_FRONTEND_LOGIN_URL = '/account/login/'
```

要将Wagtail集成到已有某种登录机制的Django站点，通常进行`WAGTAIL_FRONTEND_LOGIN_URL = LOGIN_URL`设置，就足够了。

## 设置一个全局的“需要口令”页面

通过在Django设置文件中设置`PASSWORD_REQUIRED_TEMPLATE`，可指定某个将用于站点上所有“需要口令”表单的模板路径（除开那些特殊指定了覆写该项设置的页面类型，详情参见后面，by setting `PASSWORD_REQUIRED_TEMPLATE` in your Django settings file, you can specify the path of a template which will be used for all "password required" forms on the site(except for page types that specifically override it - see below)）：

```python
# settings.py

PASSWORD_REQUIRED_TEMPLATE = 'myapp/password_required.html'
```

此模板将接收到与屏蔽页面通过`get_context()`方法将传递给其自己的模板相同上下文变量集 -- 包括用于对该页面对象自身进行引用的`page` -- 以及以下的额外变量（这些变量将覆盖该页面本身的同名上下文变量）：

+ __`form`__ -- 一个口令提示框的Django表单对象；该变量将包含一个名为`password`的字段，作为其可见字段。也可能有着一些隐藏的字段，因此在没有使用诸如`form.as_p`这样的Django渲染帮助器之一时，该页面就必须对`form.hidden_fields`进行遍历。

+ __`action_url`__ -- 该口令表单作为一个`POST`请求，将提交到的URL。


一个适合作为`PASSWORD_REQUIRED_TEMPLATE`进行使用的基本模板，将像下面这样：

```html
<!DOCTYPE HTML>

<html>
    <head>
        <title>需要口令</title>
    </head>

    <body>
        <h1>需要口令</h1>
        <p>需要口令来访问此页面。</p>
        <form action="{{ action_url }}" method="POST">
            {% csrf_token %}
            {{ form.non_field_errors }}

            <div>
                {{ form.password.errors }}
                {{ form.password.label_tag }}
                {{ form.password }}
            </div>

            {% for field in form.hidden_fields %}
                {{ field }}
            {% endfor %}

            <input type="submit" value="继续">
        </form>
    </body>
</html>
```

文档上的口令限制，使用另外的模板，经由`DOCUMENT_PASSWORD_REQUIRED_TEMPLATE`设置项加以指定；该模板也将接收到上面讲到的`form`与`action_url`上下文变量。

## 设定特定页面类型的“需要口令”页面

可在页面模型上定义`password_required_template`属性，以使用仅在那个页面类型下、“需要口令”视图的定制模板。比如，某个站点有着一个用于与描述文字一道，显示嵌入视频的页面类型，该页面类型可能要使用定制的、通常需要展示视频描述文字，但又是在原本放置嵌入视频的地方，展示口令表单的“需要口令”模板：

```python
class VideoPage(Page):
    ...

    password_required_template = 'video/password_required.html'
```
