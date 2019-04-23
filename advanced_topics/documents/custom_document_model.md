# 对文档模型进行定制

备用的`Document`模型，可用于添加定制的行为与额外字段。

要使用`Document`模型，需要完成以下步骤：

+ 创建出一个继承自`wagtail.documents.models.AbstractDocument`的新文档模型。那里就是要加入额外字段的地方。
+ 将`WAGTAILDOCS_DOCUMENT_MODEL`指向到该模型。

下面是一个示例：

```python
# models.py

from wagtail.documents.models import Document, AbstractDocument

class CustomDocument(AbstractDocument):
    # 定制字段示例：
    source = models.CharField(
        max_length=255,
        # 下面两个参数必须进行设置，以允许Wagtail在上传时创建出一个文档实例
        blank=True,
        null=True
    )

    admin_form_fields = Document.admin_form_fields + (
        # 这里要加入所有定制字段的名称，以令到他们出现在表单中：
        'source',
    )
```

> **注意** 在定制文档模型上定义的那些字段，必须要么设置为非必须（`blank=True`），要么指定一个默认值。这是因为文档的上传与定制数据的输入是两个不同的动作。Wagtail需要在上传时能够立即创建出一条文档记录。

在应用设置模块中：

```python
# 要确保将下面的 app_lebel，替换为之前把定制模型放入的那个应用
WAGTAILDOCS_DOCUMENT_MODEL = 'app_lebel.CustomDocument'
```

__从内建文档模型进行迁移__

在将某个既有站点修改为使用某个定制文档模型时，将不会自动拷贝原有文档到新的模型。这需要手动使用一个[数据迁移](https://docs.djangoproject.com/en/stable/topics/migrations/#data-migrations)来完成。

对内建文档模型进行引用的所有模板仍将如之前那样工作

## 对该定制文档模型的引用

<a name="wagtail.documents.models.get_document_model"></a>
+ `wagtail.documents.models.get_document_model()`
    
    该方法从`WAGTAILDOCS_DOCUMENT_MODEL`设置获取到文档模型。在没有定义定制模型时，默认为标准的`Document`模型。
