# 创建多语言站点（通过复制页面树的方式）

此教程将给出一种Wagtail中通过复制页面树的方式，创建多语言站点的方法。

比如：

```sh
/
    en/
        about/
        contact/
    fr/
        about/
        contact/
```

<a name="the-root-page"></a>
## 根页面

根页面（`/`）需要去侦测浏览器语言并将不同语言的浏览器请求，转发到正确语言的主页（`/en/`、`/fr/`）。根页面应位处站点根部（那里也是主页通常的所在）。

这里就必须设置 Django 的`LANGUAGES` 设置项，从而不会将那些非英语或法语的用户，重定向到不存在的页面。

```python
# settings.py

LANGUAGES = (
    ('en', _("English")),
    ('fr', _("French")),
)
```

```python
# models.py

from django.utils import translation
from django.http import HttpResponseRedirect

from wagtail.core.models import Page

class LanguageRedirectionPage(Page):
    
    def serve(self, request):
        # 此方法仅会返回一个在 Django 的 LANGUAGE 设置项中的某个语言
        language = translation.get_language_from_request(request)

        return HttpResponseRedirect(self.url + language + '/')
```

<a name="linking-pages-together"></a>
## 将全部页面链接起来

将同一页面的不同语言版本链接在一起，从而允许访问者可以轻松地在语言之间进行切换，将是有用的。但又不打算过多地增加站点编辑的负担，理想情况下，站点编辑只需将这些相同页面的一个版本，与其他版本链接起来，那么其他版本之间的链接将隐式地创建出来。

因为此做法需要添加到所有翻译的页面类型，所以最好将此做法放到混入中（a mixin）。

下面是一个如何实现此做法的示例（英语作为主语言，法语/西班牙语作为替代语言）：

```python
from wagtail.core.models import Page
from wagtail.admin.edit_handlers import MultiFieldPanel, PageChooserPanel

class TranslatablePageMixin(models.Model):
    # 每种替代语言对应一个链接
    # 这些属性与方法，应只应用在主语言的页面上（也就是英语）

    french_link = models.ForeignKey(Page, null=True, on_delete=models.SET_NULL, blank=True, related_name='+')
    spanish_link = models.ForeignKey(Page, null=True, on_delete=models.SET_NULL, blank=True, related_name='+')

    panels = [
        PageChooserPanel('french_link'),
        PageChooserPanel('spanish_link'),
    ]

    def get_language(self):
        """
        此方法返回的时该页面的语言代码。
        """

        # 通过对本页面的祖先进行查找，以找到他的语言代码
        # 语言主页位于深度 3 处（The language homepage is located at depth 3）

        language_homepage = self.get_ancestors(inclusive=True).get(depth=3)

        # 语言主页的别名，应总是被设置为语言代码
        return language_homepage.slug

    # 用于找出本页面主语言的方法
    # 这是通过对上面的链接进行反向追溯完成的（this works by reversing the above links）

    def english_page(self):
        """
        该方法找出本页面的英语版本
        """
        language = self.get_language()

        if language == 'en':
            return self
        elif language == 'fr':
            return type(self).objects.filter(french_link=self).first().specific
        elif language == 'es':
            return type(self).objects.filter(spanish_link=self).first().specific


    # 这里需要某种找出本页面的每种替代语言的一个版本的方法。
    # 这些方法工作原理一样。他们首先找出页面的主要语言版本（也就是英语）。
    # 然后从主要语言版本开始，就只需跟随链接，就能获得相应语言的正确版本了。

    def french_page(self):
        """
        此方法找出该页面的法语版本
        """

        english_page = self.english_page()

        if english_page and english_page.french_link:
            return english_page.french_link.specific

    def spanish_page(self):
        """
        此方法找出该页面的西班牙版本
        """

        english_page = self.english_page()

        if english_page and english_page.spanish_link:
            return english_page.spanish_link.specific

    class Meta:
        abstract = True

class AboutPage(Page, TranlatablePageMixin):
    ...

    content_panels = [
        ...

        MutliFieldPanel(TranslatablePageMixin.panels, '语言链接')
    ]

class ContactPage(Page, TranslatablePageMixin):
    ...

    content_panels = [
        ...

        MultiFieldPanel(TranslatablePageMixin.panels, '语言链接')
    ]
```

随后在模板中就可以向下面这样，利用上这些方法了：

{% raw %}

    {% if page.english_page and page.get_language != 'en' %}
        <a href="{{ page.english_page.url }}">{% trans "View in English" %}</a>        
    {% endif %}

    {% if page.french_page and page.get_language != 'fr' %}
        <a href="{{ page.english_page.url }}">{% trans "View in English" %}</a>        
    {% endif %}

    {% if page.spanish_page and page.get_language != 'es' %}
        <a href="{{ page.english_page.url }}">{% trans "View in English" %}</a>        
    {% endif %}
{% endraw %}
