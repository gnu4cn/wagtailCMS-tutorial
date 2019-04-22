# 对图片模型进行定制

`Image`模型可被定制，从而实现将额外字段添加到图片。

为了实现这个目的，需将以下两个模型添加到项目中：

+ 从`wagtail.images.models.AbstractImage`继承的图片模型本身。这就是要将额外字段进行添加的地方。
+ 从`wagtail.images.models.AbstractRendition`继承的转写模型。这用于存储新模型的转写。


以下是一个示例：

```python
# models.py
from django.db import models

from wagtail.images.models import Image, AbstractImage, AdstractRendition

class CustomImage(AbstractImage):
    # 在这里将所有额外字段添加到图片

    # 比如，要添加一个说明字段：
    # caption = models.CharField(max_length=255, blank=True)

    admin_form_fields = Image.admin_form_fields + (
        # 随后在这里添加字段名称，以令到其出现在表单中：
        'caption',
    )

    class CustomRendition(AbstractRendition):
        image = models.ForeignKey(CustomImage, on_delete=models.CASACDE, related_name='renditions')

        class Meta:
            unique_together = (
                ('image', 'filter_spec', 'focal_point_key'),
            )
```

> **注意** 在某个定制图片模型上定义的字段，必须要么设置为非要求的（`blank=True`），要么指定一个默认值 -- 这是因为图片的上传与定制数据的输入，是两个独立发生的事情，同时Wagtail需要能够在上传时立即创建出一条图片记录。

之后要将`WAGTAILIMAGES_IMAGE_MODEL`设置为指向该定制图片模型：

```python
WAGTAILIMAGES_IMAGE_MODEL = 'images.CustomImage'
```

__从内建图片模型进行迁移__

__Migrating from the builtin image model__

在将某个既有站点更改为使用某种定制图片模型时，图片不会自动拷贝到新的模型。需要使用一个[数据迁移](https://docs.djangoproject.com/en/stable/topics/migrations/#data-migrations)，来将原有图片拷贝到新的模型。

所有参考内建图片模型的模板，仍将之前那样继续工作，但要对其进行更新，才能观看到新的图片。


## 对定制图片模型的引用

<a name="wagtail.images.get_image_model"></a>
### `wagtail.images.get_image_model()`

从`WAGTAILIMAGES_IMAGE_MODEL`设置取得图片模型。此方法对于那些制作Wagtail插件、需要图片模型的开发者有用。在没有定义定制模型时，默认为标准的`Image`模型。

<a name="wagtail.images.get_image_model_string"></a>

获取图片模型的字符串形式的、带点的`app.Model`名称。对于制作Wagtail插件、需要对图片模型进行引用，比如在外键中，但又不需要模型本身的开发者有用。
