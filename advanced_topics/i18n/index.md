# 国际化问题

本文档描述了Wagtail的国际化特性与如何创建多语言站点。

Wagtail使用了Django的 [国际化框架](https://docs.djangoproject.com/en/stable/topics/i18n/)，因此大多数步骤都与其他Django项目相同。

__目录__

+ 国际化问题

    - Wagtail管理界面的翻译
    - 在单个用户基础上改变Wagtail的管理界面语言
    - 改变Wagtail安装的主要语言
    + 创建具备多种语言的站点
    
        - 开启多语言支持
        - 根据不同URLs而提供不同的语言
        - 对模板进行翻译
        - 对内容进行翻译
        - 其他方法


## Wagtail管理界面的翻译

Wagtail的管理后端已被翻译为了多种不同语言。在Wagtail的 [Transifex 页面](https://www.transifex.com/torchbox/wagtail/)，可以找到一个当前可用翻译的清单（注意：若使用的时某个较早版本的Wagtail，那么该页面并不能精确反映此安装下有哪些可用的语言）。

如在那个页面上没有自己的语言，那么贡献新的语言，或修正其中的错误，也是容易的。在 [Transifex](https://www.transifex.com/torchbox/wagtail/)上注册病提交修改即可。翻译的更新通常会在提交后的一个月内，合并到Wagtail的正式发布中。

## 在用户层面修改Wagtail管理界面的语言

已登入用户可在`/admin/account`处设置其偏好的语言。默认下Wagtail提供了有着高于90%的翻译覆盖的语言清单。可通过`WAGTAILADMIN_PERMITTED_LANGUAGES`设置，来覆写此行为。

在没有语言或只有一种语言得以允许时，该表单将被隐藏。

在用户没有选择语言时，将使用`settings.py`中的`LANGUAGE_CODE`设置。

## 修改 Wagtail 安装的主要语言

Wagtail的默认语言为`en-us`（美国英语）。可通过调整一两个Django的设置来修改其默认语言：

+ 确保 [`USE_I18N`](https://docs.djangoproject.com/en/stable/ref/settings/#use-i18n) 被设置为 `True`。
+ 将 [`LANGUAGE_CODE`](https://docs.djangoproject.com/en/stable/ref/settings/#std:setting-LANGUAGE_CODE)设置为网站的主要语言。

在有着所设置语言的翻译时，那么现在的Wagtail管理后端就应是所选的语言了。

## 创建带有多种语言的站点

可资利用 Django 的 [翻译特性](https://docs.djangoproject.com/en/stable/topics/i18n/translation/)，来创建带有多种语言支持的站点。

本小节的文档，将给出如何结合Wagtail来使用 Django 的翻译特性，还将介绍几种使用Wagtail页面来存储/获取已翻译内容的方法。

### 开启多语言支持

首先要确保Django 的`USE_I18N`设置已被置为 `True`。

要开启多语言支持，就要将 `django.middleware.locale.LocaleMiddleware` 添加到`MIDDLEWARE`设置项：

```python
MIDDLEWARE = (
    ...

    'django.middleware.locale.LocaleMiddleware',
)
```

此中间件类对用户的浏览器语言进行查看，并[根据用户浏览器语言来对站点的语言加以设置](https://docs.djangoproject.com/en/stable/topics/i18n/translation/#how-django-discovers-language-preference)。

### 根据不同URLs而提供不同的语言

有时仅开启Django的多语言支持还不够。默认Django提供到有着相同URL的同一页面提供不同语言。但这样做却有着一些弊端：

+ 用户无法在不修改他们的浏览器设置下改变语言
+ 在有着多种缓存设置时，这种做法可能无法工作（在内容随浏览器设置而变化时）

在开启了 Django 的 `i18n_patterns` 特性时，就会在URLs前冠以语言代码（比如`/en/about-us`）。此时用户在初次访问站点时，依据其浏览器语言，而转往其偏好的版本了。

此特性是通过项目的根URL配置进行开启的。只需将那些想要开启此特性的视图，放入到一个`i18n_patterns`清单中，并将那个清单追加到其他URL模式即可：

```python
# mysite/urls.py

from django.conf.urls import include, re_path
from django.conf.urls.i18n import i18n_patterns
from django.conf import settings
from django.contrib import admin

from wagtail.admin import urls as wagtailadmin_urls
from wagtail.documents import urls as wagtaildocs_urls
from wagtail.core import urls as wagtail_urls
from search import views as search_views

urlpatterns = [
    re_path(r'^django-admin', include(admin.site.urls)),

    re_path(r'^admin', include(wagtailadmin_urls)),
    re_path(r'^documents', include(wagtaildocs_urls)),
]

urlpatterns += i18n_patterns(
    # 下面这些URLs将带有前面追加的 /<language_code>/

    re_path(r'^search/$', search_views.search, name='search'),

    re_path(r'', include(wagtail_urls)),
)
```

可通过改变URL前面部分，来实现语言的切换。因为每种语言都有其自己的URL，因此在设置有缓存的情况下，也可良好工作。

### 模板的翻译

模板中的静态文本，需要以某种方式标记起来，从而令到Django的 `makemessages` 命令能够为翻译者找到并导出那些字符串，同时在模板被提供时，允许这些字符串切换到翻译后的版本。

因为Wagtail使用了Django的模板，故这些标记的插入，以及模板中字符串的导出与翻译流程，与其他所有Django项目都是一样的。

请参阅：https://docs.djangoproject.com/en/stable/topics/i18n/translation/#internationalization-in-template-code

### 对内容的翻译

Wagtail中对内容的翻译，最常用的方法，就是对各个可翻译文本字段进行复制，为每种语言提供一个单独的字段。

本小节将就如何手动实现这种方法进行讲解，但有一个可使用的第三方模块，[wagtail 模型翻译](https://github.com/infoportugal/wagtail-modeltranslation)，在该模块满足需求时，要快捷一些。

__复制模型中的字段__

对那些认为可以翻译的各个字段，为所支持的每种语言都复制一份，并以语言代码作为后缀：

```python
class BlogPage(Page):

    title_fr = models.CharField(max_length=255)

    body_en = StreamField(...)
    body_fr = StreamField(...)

    # 独立于不同语言的字段，无需进行复制
    thumbnail_image = models.ForeignKey('wagtailimages.Image' on_delete=model.SET_NULL, null=True, ...)
```

> **注意** 这里只定义了法语版本的`title`，因为Wagtail 已经提供了英语版本。

__对管理界面中的字段进行组织__

既可以将有着翻译的全部字段依次放在“内容”分页上，也可将其他语言的翻译放在不同分页上。

请参阅 [对分页界面进行定制](../customisation/page_editing_interface.md#customising-the-tabbed-interface) 了解有关如何将更多分页加入到管理界面的信息。

__从模板访问这些字段__

为了让这些翻译显式在站点的前端，就需要根据客户端所选的语言，在模板中使用正确的字段。

如果每次在模板中显示一个字段时，都必须加入语言的检查，那么将令到模板十分混乱。有个小小的窍门，令到既能实现此目的，又能保持模板与模型代码的整洁。

可使用类似下面的一小段代码，来给页面模型添加一些访问器字段。这些访问器字段将指向包含了用户选定语言的字段。

将下面的代码复制到项目中，并确保其在所有包含了有着翻译字段的`Page`页面模型类的`models.py`文件中有被导入。为了支持不同语言，将需要对此代码进行一些修改。

```python
from django.utils import transaltion

class TranslatedField:

    def __init__(self, en_field, fr_field):
        self.en_field = en_field
        self.fr_fiedl = fr_fiedl

    def __get__(self, instance, owner):
        if translation.get_language() == 'fr':
            return getattr(instance, self.fr_field)
        else:
            return getattr(instance, self.en_fiedl)
```

随后为各个翻译字段，创建一个有着良好命名（因为那就是模板中将要引用的名称）的`TranslatedField`的实例。

比如以下就是将要应用到上面的`BlogPage`模型的方法：

```python
class BlogPage(Page):
    ...

    translated_title = TranslatedField(
        'title',
        'title_fr',
    )

    body = TranslatedField(
        'body_en',
        'body_fr',
    )
```

最后在模板中，对访问器而非底层的数据库字段进行引用：

```html
{{ page.translated_title }}

{{ page.body }}
```

### 其他方式

+ [创建多语言站点（通过复制页面树）](duplicate_tree.md)
    
    - [根页面](duplicate_tree.md#the-root-page)
    - [将页面链接起来](duplicate_tree.md#linking-pages-together)
