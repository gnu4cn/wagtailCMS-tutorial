# 动画GIF的支持

Wagtail默认的图片库 Pillow, 并不支持动画的GIFs。

要获得动画GIF的支持，就必须[安装Wand](http://docs.wand-py.org/en/0.4.2/guide/install.html)，Wand是一个到ImageMagick的绑定，所以要确保有安装ImageMagick程序。

在安装好了之后，Wagtail将自动使用Wand来对GIF文件进行调整，而仍然使用Pillow来调整其他图片。
