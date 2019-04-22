# 特征识别

__Feature Detection__

Wagtail有着自动侦测面孔及图片中一些特征，并根据这些特征而对图片加以裁剪的能力。

特征识别特性，在图片上传时，使用OpenCV来识别图片中的面孔/特征。识别到的特征，作为`Image`模型上`focal_point_{x, y, width, height}`字段中的一个焦点，进行了内部保存（the detected features stored internally as a focal point in the `focal_point_{x, y, width, height}` fields on the `Image` model）。这些字段在图片于模板中被渲染时，为`fill`图片过滤器用来对图片进行裁剪。

## 设置

特征识别需要OpenCV，而OpenCV在安装时有点麻烦，因为其当前尚不支持`pip`的安装。

### 在Debian/Ubuntu上按照OpenCV

Debian与Ubuntu提供了一个名为`python-opencv`的`apt-get`软件包：

```sh
$ sudo apt-get install python-opencv python-numpy
```

这将把`PyOpenCV`安装到站点包中。在使用虚拟环境时，就要确保开启了站点包，否则Wagtail将不能导入`PyOpenCV`。

### 在虚拟环境中开启站点包

__Enabling site packages in the virtual environment__

如未用到虚拟环境，那么可跳过此步骤。

站点包的开启，则会根据是否使用`pyvenv`（仅 Python 3.3以上版本），还是`virtualenv`来管理虚拟环境，而有所不同。

+ `pyvenv`

    前往`pyvenv`目录并打开`pyvenv.cfg`文件，然后将`include-system-site-packages`设置为`true`。


+ `virtualenv`

    前往虚拟环境目录并删除一个名为`lib/python-x.x/no-global-site-packages.txt`的文件。

## 对OpenCV安装进行测试

此时可通过打开一个Python命令行界面（在虚拟环境激活下），并进行下面的输入，对Wagtail是否能看到OpenCV进行测试：

```python
import cv
```

> **译者注** 这里尚需本地安装 `opencv-python` 包，通过 `pip install opencv-pyton`，并输入`import cv2`。

此时如没有看到`ImportError`，那么就算已经安装好了（如看到`libdc1394 error: Failed to initialize libdc1394`，这是无害的告警，可被忽略）。

### 在Wagtail中开启特征识别特性

一旦OpenCV安装好了，就需要将`WAGTAILIMAGES_FEATURE_DETECTION_ENABLED`设置为`True`:

```python
# settings.py

WAGTAILIMAGES_FEATURE_DETECTION_ENABLED = True
```

### 手动运行特征识别

特征识别是在有新的图片上传到Wagtail中时运行的。在站点中已有一些图片且想要在这些图片上运行特征识别时，就必须手动运行了。

```python
from wagtail.images.models import Image

for image in Image.objects.all():
    if not image.has_focal_point():
        image.set_focal_point(image.get_suggested_focal_point())
        image.save()
```
