# 以Python方式生成图片的转写

由Wagtail的`{% image %}`模板标签所生成的原始图片的渲染版本，被成为“转写（reditions）”，他们作为新的图片文件，在初次调用时，就存储在站点的`[media]/images`目录中了。

还可通过原生的`get_rendition()`方法，从Python动态地生成图片转写，比如：

```python
newimage = myimage.get_rendition('fill-300x150|jpegquality-60')
```

在`myimage`的文件名为`foo.jpg`时，那么将生成一个名为`foo.fill-300x150.jpegquality-60.jpg`的该图片文件的新的转写，并保存到该站点的`[media]/images`目录。该方法的参数选项与模板标签`{% image %}`的过滤器规范一致，且应使用`|`进行分隔。

所生成的`Rendition`对象，将有着一些特定于该图片版本的属性，比如`url`、`width`及`height`等，因此类似于这些就可在某个API生成器中加以使用，比如：

```python
url = myimage.get_rendition('fill-300x150|jpegquality-60').url
```

属于所生成转写的原始图片的那些参数，比如`title`，可通过该转写的`image`属性而访问到：

```sh
>>> newimage.image.title
'Blue Sky'
>>> newimage.image.is_landscape()
True
```

另请参阅：[在模板中使用图片](../../topics/images.md#image-tag)
