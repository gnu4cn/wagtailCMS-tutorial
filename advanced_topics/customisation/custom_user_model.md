# 定制用户模型

## 用户表单定制示例

本示例对如何将一个文本字段与外键，添加到某个定制的用户模型，及将Wagtail的用户表单配置为允许对这些字段进行更新进行了演示。

创建出一个定制用户模型。在此情形下，这里对`AbstractUser`类进行了扩展，并添加了两个字段。这里的外键引用了另一个模型（并未给出）。

```python
class User(AbstractUser):
    country = models.CharField(verbose_name='国家', max_length=255)
    status = models.ForeignKey(MembershipStatus, on_delete=models.SET_NULL, null=True, default=1)
```

将包含了这个用户模型的应用，添加到 `INSTALLED_APPS` 并将 `AUTH_USER_MODEL` 设置为对该模型的应用。在本示例中，应用名为 `users` 同时模型为 `User`

```python
AUTH_USER_MODEL = 'users.USER'
```

在应用中创建出定制的用户 `create` 与 `edit` 表单：

```python
from django import forms
from django.utils.translation import ugettext_lazy as _

from wagtail.users.forms import UserEditForm, UserCreationForm

class CustomUserEditForm(UserEditForm):
    contry = forms.CharField(required=True, label=_("Country"))
    status = forms.ModelChoiceField(queryset=MembershipStatus.objects, required=True, label=_("Status"))

class CustomUserCreationForm(UserCreationForm):
    country = forms.CharField(required=True, label=_("Country"))
    status = forms.ModelChoiceField(queryset=MembershipStatus.objects, required=True, label=_("Status"))
```

对Wagtail的用户 `create` 与 `edit` 模板加以扩展。这些扩展的模板应放在模板目录`wagtailusers/users`下。

模板 `create.html`：

{% raw %}
    
    {% extends "wagtailusers/users/create.html" %}

    {% block extra_fields %}
        {% include "wagtailadmin/shared/field_as_li.html" with field=form.country %}
        {% include "wagtailadmin/shared/field_as_li.html" with field=form.status %}
    {% endblock %}
{% endraw %}


模板 `edit.html`：

{% raw %}
    
    {% extends "wagtailusers/users/edit.html" %}

    {% block extra_fields %}
        {% include "wagtailadmin/shared/field_as_li.html" with field=form.country %}
        {% include "wagtailadmin/shared/field_as_li.html" with field=form.status %}
    {% endblock %}
{% endraw %}


这里的 `extra_fields` 块允许将一些字段插入到默认模板中的 `last_name` 字段下面。有一些其他块的覆写选项，允许将一些字段追加到既有字段的末尾或开头，或是允许对所有字段进行重新定义。

将下面这些 Wagtail 设置项添加到项目，以对这些用户表单附加项进行引用：

```python
WAGTAIL_USER_EDIT_FORM = 'users.forms.CustomUserEditForm'
WAGTAIL_USER_CREATION_FORM = 'users.forms.CustomUserCreationForm'
WAGTAIL_USER_CUSTOM_FIELDS = ['country', 'status']
```
