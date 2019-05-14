# 对管理模板进行定制

在Wagtail项目中，可能希望在管理界面中，使用自己的品牌元素，来替换默认的Wagtail徽标等元素。可经由Django的继承机制，实现此目标。

需要在某个应用目录下，创建一个`templates/wagtailadmin/`的文件夹 -- 这既可以是一个既有的文件夹，也可以是为此目的而新建的文件夹，比如 `dashboard`。该应用必须是已在 `INSTALLED_APPS` 中，于`wagtail.admin`之前已注册的应用：

```python
INSTALLED_APPS = (
    # ...
    'dashboard',

    'wagtail.core',
    'wagtail.admin',

    # ...
)
```


## 品牌的定制

可用于对管理界面中的品牌加以定制的模板块包含以下这些：

+ `branding_logo`

    要替换默认的徽标，就要创建一个对默认的`branding_logo`块进行覆写的模板文件 `dashboard/templates/wagtailadmin/base.html`：

{% raw %}

    {% extends "wagtailadmin/base.html" %}
    {% load static %}

    {% block branding_logo %}
        <img src="{% static 'images/custom-logo.svg' %}" alt="定制项目" width="80">
    {% endblock %}

{% endraw %}

该徽标也出现在管理的 `404` 错误页面上；要在那里对其进行替换，就要创建一个对`branding_logo`块加以覆写的`dashboard/templates/wagtailadmin/404.html`的模板文件。

+ `branding_favicon`

    要替换在查看管理页面时所显示的站点图标（the favicon），就要创建一个对`branding_favicon`块进行覆写的 `dashboard/templates/wagtailadmin/admin_base.html`的模板文件：

{% raw %}
    
    {% extends "wagtailadmin/admin_base.html" %}
    {% load static %}

    {% block branding_favicon %}
        <link rel="shortcut icon" href="{% static 'images/favicon.ico' %}" />
    {% endblock %}
{% endraw %}


+ `branding_login`

要替换登录消息，就要创建一个用于对`branding_login`块进行覆写的 `dashboard/templates/wagtailadmin/login.html`的模板文件：

{% raw %}

    {% extends "wagtailadmin/login.html" %}
    {% block branding_login %}登入科服斯网站{% endblock %}
{% endraw %}


+ `branding_welcome`

    要替换站点仪表盘上的欢迎消息，就要创建一个对 `branding_welcome`块进行覆写的模板文件 `dashboard/templates/wagtailadmin/home.html`：

{% raw %}

    {% extends "wagtailadmin/home.html" %}

    {% block branding_welcome %}
        欢迎来到科服斯管理后台！
    {% endblock %}
{% endraw %}

## 在站点品牌中指定一个站点或页面

管理界面有着一些对渲染器上下文可用的变量，他们可用于对管理页面中的品牌进行定制。在对多租户的Wagtail安装进行定制时，这些变量将是有用的：

+ `root_page`

返回的是对于当前登入用户来说最高的可浏览页面对象。在用户没有浏览权限时，这将默认为 `None`。

+ `root_site`

返回上述根页面的站点记录上的名称。

+ `site_name`

在计算出 `root_site` 的值不是 `None` 时，返回其值。在`root_site`的值为`None`时，将返回 `settings.WAGTAIL_SITE_NAME`的值。

要使用这些变量，就如同之前对仪表盘中的那些模板块的覆写一样，要创建一个`dashboard/templates/wagtailadmin/home.html`的模板文件，并像使用其他Django的模板变量一样，对这些变量进行使用：

{% raw %}

    {% extends "wagtailadmin/home.html" %}
    {% block branding_welcome %}
        欢迎来到 {{ root_site }} 的管理首页
    {% endblock %}
{% endraw %}

<a name="extending-the-login-form"></a>
## 对登录表单进行扩展

要将额外控件加入到登录表单，就要创建一个模板文件 `dashboard/templates/wagtailadmin/login.html`。

+ `above_login` 与 `below_login`

要在登录表单的上面或下面添加内容，就要对这些块进行覆写：

{% raw %}

    {% extends "wagtailadmin/login.html" %}
    {% block branding_login %}登入 {{ site_name }}{% endblock %}

    {% block above_login %}请使用用户名和口令登入{% endblock %}

    {% block below_login %}若非本站会员，请不要尝试登入。{% endblock %}
{% endraw %}


+ `fields`

要将额外字段添加到登录表单，就要对 `fields` 块进行覆写。将需要在块中的某处加入 `{{ block.super }}`，以将用户名与口令字段包含进来：

{% raw %}

    {% extends "wagtailadmin/login.html" %}

    {% block fields %}
        {{ block.super }}
        <li class="full">
            <div class="field iconfield">
                两步认证令牌
                <div class="input icon-key">
                    <input type="text" name="two-factor-auth" />
                </div>
            </div>
        </li>
    {% endblock %}
{% endraw %}


+ `submit_buttons`

要将额外按钮添加到登录表单，就要对 `submit_buttons` 块进行覆写。将需要在块中某处加入 `{{ block.super }}`，以将登入按钮包含进来:

{% raw %}

    {% extends "wagtailadmin/login.html" %}
    
    {% block submit_buttons %}
        {{ block.super }}
        <a href="{% url 'signup' %}">
            <button type="button" class="button" tabindex="4">{% trans 'Sign up' %}</button>
        </a>
    {% endblock %}
{% endraw %}

+ `login_form`

要对登录表单进行完全的定制，就要对 `login_form` 块进行覆写。此块封装了`form`元素的全部内容：

{% raw %}

    {% extends "wagtailadmin/login.html" %}

    {% block %}
        <p>一些额外的表单内容</p>
        {{ block.super }}
    {% endblock %}
{% endraw %}

## 对客户端组件进行扩展

一些Wagtail的管理界面，是作为带有 [React](https://reactjs.org/) 的客户端JavaScript代码编写而成的。为了对这些组件进行定制或扩展，就需要用到 React，以及其他一些相关的库。为令到此工作更为容易，Wagtail将其自己与React有关的依赖，作为管理界面中的全局依赖而加以了暴露。下面时这些可用的包：

```javascript
// 'focus-trap-react'
window.FocusTrapReact;
// 'react'
window.React;
// `react-dom`
window.ReactDOM;
// 'react-transition-group/CSSTransitionGroup'
window.CSSTransitionGroup;
```

Wagtail还暴露了其自己的一些 React 组件。可重用这些组件：

```javascript
window.wagtail.components.Icon;
window.wagtail.components.Portal;
```

包含了富文本编辑器的页面还可以访问到下面这些组件：

```javascript
// `draft.js`
window.DraftJS;
// 'draftail'
window.Draftail;

// Wagtail 与 Draftail有关的 AIPs 及组件。
window.draftail;
window.draftail.ModalWorkflowSource;
window.draftail.Tooltip;
window.draftail.TooltipEntity;
```
