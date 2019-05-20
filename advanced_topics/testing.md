# 对 Wagtail站点进行测试

Wagtail自带了一些可以简化编写站点测试的工具。

## `WagtailPageTests`

<a name=""wagtail.tests.utils.WagtailPageTests></a>
+ `class wagtail.tests.utils.WagtailPageTests`

`WagtailPageTests` 对 `django.test.TestCase` 进行了扩展，加入了一些新的 `assert` 方法。可对此类进行扩展，以利用其各个方法：

```python
from wagtail.tests.utils import WagtailPageTests
from myapp.models import MyPage

class MyPageTests(WagtailPageTests):
    def test_can_create_a_page(self):
        ...
```

+ `assertCanCreatAt(parent_model, child_model, msg=None)`

断言某个特定子页面类型可在某个父页面类型下进行创建。`parent_model` 与 `child_model` 应是要测试的页面类型。

```python
def test_can_create_under_home_page(sefl):
    # 这里可在某个 HomePage 下创建一个 ContentPage
    self.assertCanCreateAt(HomePage, ContentPage)
```

+ `assertCanNotCreateAt(parent_model, child_model, msg=None)`

断言不能在某个父页面类型下创建某个特定的子页面类型。`parent_model`与`child_model`应是要测试的页面类。

```python
def test_cant_create_under_event_page(self):
    # 这里无法在 EventPage 下创建 ContentPage 的页面
    self.assertCanNotCreateAt(EventPage, ContentPage)
```
+ `assertCanCreate(parent, child_model, data, msg=None)`

断言某个给定页面类型的子页面，可在该父页面下，使用提供的 POST 数据进行创建。

`parent` 应是某个页面实例，同时`child_model`应是一个页面子类。`data`应是一个将在Wagtail管理界面的页面创建方法处将要 POST 提交的字典。

```python
from wagtail.tests.utils.form_data import netsted_form_data, streamfield

def test_can_create_content_page(self):

    # 获取主页
    root_page = HomePage.objects.get(pk=2)

    # 断言可在这里以该 POST 数据，制作一个 ContentPage
    self.assertCanCreate(root_page, ContentPage, nested_form_data({
        'title': '关于我们',
        'body': streamfield([
            ('text', 'Lorem ipsum dolor sit amet'),
        ]),
    }))
```

请参阅 [表单数据助手](#form-data-test-helpers) 了解有关构造 POST 数据的一些有用函数。

+ `assertAllowedParentPageTypes(child_model, parent_models, msg=None)`

对只能在`parent_models` 创建 `child_model` 这一事实进行测试。

在父模型已经设置了 `Page.subpage_types`时， 所允许的父模型清单，可与`Page.parent_page` 中设置的不同。

```python
def test_content_page_parent_pages(self):
    # ContentPage 只能在 HomePage 或另一个 ContentPage 下创建
    self.assertAllowedParentPageTypes(ContentPage, {HomePage, ContentPage})

    # EventPage 只能在某个 EventIndex 下被创建
    self.assertAllowedParentPageTypes(EventPage, {EventIndex})
```

+ `assertAllowedSubpageTypes(parent_model, child_models, msg=None)`

对 `parent_model`下只能创建的页面类型为 `child_models` 这一事实进行测试。

在子模型已被设置为`Page.parent_page_types`时，所允许的子模型清单可与 `Page.subpage_types` 中设置的有所不同。

```python
def test_content_page_subpages(self):

    # ContentPage 只能有其他的 ContentPage 子页面
    self.assertAllowedSubPageTypes(ContentPage, {ContentPage})

    # HomePage 只能有 ContentPage 与 EventIndex 的子页面
    self.assertAllowedParentPageTypes(HomePage, {ContentPage, EventIndex})
```

<a name="form-data-test-helpers"></a>
## 表单数据助手

`assertCanCreate`方法，要求有与页面编辑表单中所提交数据同样格式的页面数据传递给他。对于复杂的页面类型，就难以手动构造该数据结构；模块`wagtail.tests.utils.form_data`就提供了一套助手函数，用于帮助完成 POST 数据的构造。

+ `wagtail.tests.utils.form_data.nested_form_data(data)`

将某个嵌套字典的数据结构，转换为带有连字符分隔键的平面表单数据字典（a flat form data dict with hyphen-separated keys）。

```python
nested_form_data({
    'foo': 'bar',
    'parent': {
        'child': 'field',
    },
})

# 将返回：{'foo': 'bar', 'parent-child': 'field'}
```


+ `wagtail.tests.utils.form_data.rich_text(value, editor='default', features=None)`

将一个类似于 HTML 的富文本字符串，转换为当前活动的富文本编辑器要要求的数据格式。

__参数__：
    
+ `editor` -- 一个在`WAGTAILADMIN_RICH_TEXT_EDITORS` 中所定义的替代性编辑器名称
+ `features` -- 在富文本内容中所允许的功能清单（参见 [对富文本字段中的功能进行限制](customisation/page_editing_interface.md#rich-text-features)）
    
```python
self.assetCanCreate(root_page, ContentPage, nested_form_data({
    'title': '关于我们',
    'body': rich_text('<p>Lorem ipsum dolor sit amet</p>'),
}))
```

+ `wagtail.tests.utils.form_data.streafield(items)`

取得一个元组的清单（`block_tyep`、`value`），并将其转换为 `StreamField`的表单数据。在某个 `nested_form_data()` 调用中使用该方法，将字段名称作为键。

```python
nested_form_data({'content': streamfield([
    ('text', 'Hello, world'),
])})

# 将返回：
# {
#     'content-count': '1',
#     'content-0-type': 'text',
#     'content-0-value': 'Hello, world',
#     'content-0-order': '0',
#     'content-0-deleted': '',
# }
```


+ `wagtail.tests.utils.form_data.inline_formset(items, initial=0, min=0, max=1000)`

取得一个 `InlineFormset` 的表单数据清单，并将其转换为有效的POST数据。在某个 `nested_form_data()` 调用中使用此方法，以表单集的关系名称作为键（with the formset relation name as the key）。

```python
nested_form_data({'lines': inline_formset([
    {'text': 'Hello'},
    {'text': 'World'},
])})

# 将返回：
# {
#     'lines-TOTAL_FORMS': '2',
#     'lines-INITIAL_FORMS': '0',
#     'lines-MIN_NUM_FORMS': '0',
#     'lines-MAX_NUM_FORMS': '1000',
#     'lines-0-text': 'Hello',
#     'lines-0-ORDER': '0',
#     'lines-0-DELETE': '',
#     'lines-1-text': 'World',
#     'lines-1-ORDER': '1',
#     'lines-1-DELETE': '',
# }
```


## Fixtures

### 使用 `dumpdata`

通过创建出一些开发环境中的内容，并使用Django 的 [dumpdata](https://docs.djangoproject.com/en/2.0/ref/django-admin/#django-admin-dumpdata) 命令，是为测试目的而创建 一些 [fixtures](https://docs.djangoproject.com/en/stable/howto/initial-data/)的最佳方法。

请注意默认的 `dumpdata` 将以主键来表示 `content_type`；在添加/移除模型时，这可能会导致一致性问题，因为内容类型是独立于 fixtures 而单独生成的。为防止这个问题，就要使用 `--natural-foreign` 命令开关，此命令开关将使用 `["app", "model"]` 来表示内容类型。

### 手动修改

可手动修改复制的 fixtures，或设置完全手动地编写这些 fixtures。以下是一些需要留意的地方。

__定制页面模型__

在创建 fixtures 中的定制页面模型时，将需要同时添加一个 `wagtailcore.page` 的条目，以及一个定制页面模型的条目。

假设有着一个定义了`Homepage(Page)` 类的 `website` 模块。那么就可以在某个 fixture 中创建以下的一个主页：

```python
[
    {
        "model": "wagtailcore.page",
        "pk": 3,
        "fileds": {
            "title": "客户的主页",
            "content_type": ["website", "homepage"],
            "depth": 2
        }
    },
    {
        "model": "website.homepage",
        "pk": 3,
        "fields": {}
    }
]
```

__Treebeard 字段__

为了令到像是 `get_parent` 这类页面树的操作正确工作，因此填入 `path` / `numchild` / `depth` 字段是必要的。在某些不同寻常的情况下，若未给出值，`url_path`是另一个可导致错误的字段。

[Treebeard 文档](http://django-treebeard.readthedocs.io/en/latest/mp_tree.html) 可帮助理解其工作原理。
