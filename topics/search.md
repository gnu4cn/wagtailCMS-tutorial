# 搜索功能

Wagtail提供了全面且可扩展的搜索接口。此外，其还经由“站点编辑精选（Editor's Picks）”方式，提供提升搜索结果的方法。Wagtail还将一些有关通过搜索接口进行的查询的、简单统计数据收集了起来。


+ [建立索引](#indexing)

    - [更新索引](updating-the-index)
    - [对额外字段进行索引](indexing-extra-fields)
    - [对定制模型建立索引](indexing-custom-models)


+ [进行搜索](#searching)

    - [搜索的QuerySet](#searching-querysets)
    - [页面搜索视图示例](#an-example-page-search-view)
    - [提升后的搜索结果](#promoted-search-results)


+ [搜索后端](#backends)

    - [`AUTO_UPDATE`](#auto-update)
    - [`ATOMIC_REBUILD`](#atomic-rebuild)
    - [`BACKEND`](#backend)


## 索引的建立

必须首先将对象加入到搜索索引，那么这些对象才是可搜索的。而这涉及到那些想要进行索引的模型与字段的配置（索引正是对页面、图片与文档进行的），随后才是将这些对象插入到索引中。

请参阅[更新索引](#updating-the-index)部分，了解如何领导搜索索引中的对象，与数据库中的对象保持同步。

在`Page`或`Image`基类的子类中已创建了一些额外字段时，或许打算将这些新的字段加入到搜索索引，从而令到用户的搜索查询，可以与这些页面或图片的额外内容进行匹配。请参阅[对额外字段进行索引](#indexing-extra-fields)。

在有着不是从`Page`或`Image`基类派生的定制模型时，却又要令到其可搜索，那么请参阅[建立定制模型的索引](#indexing-custom-models)。

## 进行搜索

Wagtail提供了用于在模型上完成搜索的一个API。同时还可以在Django QuerySets上进行搜索查询。

请参阅[进行搜索](#searching)。


## 关于后端

Wagtail提供了为搜索索引的存储与完成搜索查询提供了三种后端：Elasticsearch、数据库，以及 PostgreSQL（需要Django >= 1.10）。也可运行自己的搜索后端。

请参阅 [关于搜索后端](#backends)。

<a name="indexing"></a>
# 索引的建立

要令到模型可视化，就需要将其加入到搜索索引中。所有页面、图片与文档都被索引起来，那么就可以立即开始对他们进行搜索了。

在某个页面或图片基类的子类中建立了额外字段时，可能会将这些新的字段，也加入到搜索索引中，从而令到用户的搜索查询，对这些子类的内容进行匹配。请参阅[对额外字段建立索引](#indexing-extra-fields)。

在希望将定制模型纳入搜索时，请参阅[建立定制模型的索引](#indexing-custom-models)。

## 更新索引

在搜索索引与数据库分离的情况下（比如使用Elesticsearch时），就要让二者保持同步状态。有两种方式可以完成此操作：使用搜索信号处理器，或周期性地调用`update_index`命令。处于最佳速度与可靠性的考虑，最好的做法是在可能的情况下二者同时使用。

### 关于信号处理器

`wagtailsearch`库提供了一些绑定到所有已索引模型的保存/删除信号的信号处理器。这将自动在已于`WAGTAILSEARCH_BACKENDS`中注册的所有后端中，添加及删除相应他们。在`wagtail.search`应用装入时，这些信号处理器自动进行了注册。

### 关于`update_index`命令

Wagtail还提供了用于从头开始重建索引的一个命令

```bash
./manage.py update_index
```

推荐每周运行一次此命令，以及在以下时刻运行他：

+ 在经由脚本（比如导入内容后）创建了页面后
+ 在对模型或搜索的配置进行了修改之后

因为此命令运行时的搜索不会返回结果，因此要避免在峰值时段运行此命令。

<a name="indexing-extra-fields"></a>
## 对额外字段进行索引

> **警告** 数据库后端并不支持对额外字段的索引。在使用数据库后端是，经由`search_fields`所定义的所有其他字段，都会被忽略。


必要要将字段显式地加入到那些由`Page`基类所派生的模型的`search_fields`字段，以便对这些字段进行搜索/过滤。这是通过对`search_fields`进行覆写，来将一个额外的`SearchField`/`FilterField`字段给他，而完成的。

### 示例

下面的代码创建了有着两个字段的`EventPage`模型：`description`与`date`。`description`作为了`SearchField`进行索引，而`date`则作为`FilterField`进行索引。

```python
from watail.search import index
from django.utils import timezone

class EventPage(Page):
    description = models.TextField()
    date = models.DateField()

    search_fields = Page.search_fields + [ # 这里从 Page 基类继承 search_fields
        index.SearchField('description'),
        index.FilterField('date'),
    ]

# 获取在标题或活动描述中包含 “Christmas” 字符串的未来活动
>>> EventPage.objects.filter(date__gt=timezone.now()).search("Christmas")
```

## 关于`index.SearchField`

指定用于在模型上执行全文搜索的字段，通常是指文本字段。

### 选项

+ `partial_match(boolean)` -- 将该选项设置为真时，允许结果在部分词上匹配。比如此选项默认对标题就设置为真，那么在某个页面的标题为`Hello World!`时，若用户在搜索框中输入的是`Hel`，则该页面将会被找到。

+ `boost(int/float)`  -- 此选项允许将一些字段设置为较其他字段更为重要。将某字段的此选项设置为一个较高的数字，将那些在该字段匹配的页面，在结果共排序靠前。通常页面标题字段的该选项被设置为`2`，所有其他字段的该选项被设置为`1`。

+ `es_extra(dict)` -- 指定该字段允许开发者设置或覆写ElasticSearch映射中该字段上的所有设置。在打算使用某些Wagtail尚不支持的ElasticSearch的特性时，就要用到此选项。


## 关于`index.FilterField`

指定要加入到搜索索引的字段，但这些字段不用于全文搜索。而是可以在搜索结果上运行过滤器。


## 关于`index.RelatedFields`

此特性允许对相关对象的字段进行索引。其工作于相关字段的所有类型，包含他们的反向访问器（This allows you to index fields from related objects. It works on all types of related fields, including their reverse accessors）。

比如这里有一个到书籍作者的`ForeignKey`，就可以将作者模型的`name`与`date_of_birth`字段，嵌入到书籍模型内：

```python
from wagtail.search import index

class Book(models.Model, index.Indexed):
    ...

    search_fields = [
        index.SearchField('title'),
        index.FilterField('published_date'),

        index.RelatedFields('author', [
            index.SearchField('name'),
            index.FilterField('date_of_birth'),
        ]),
    ]
```

### `index.RelatedFields`上的过滤

使用`QuerySet`编程接口在`index.RelatedFields`中的所有`index.FilterFields`上进行过滤，都是不可能的。不过这些字段既然有被索引起来，那么就有可能通过手动查询ElasticSearch来用到他们。

Wagtail计划在未来的发行中，实现经由`QuerySet`在`index.RelatedFields`上的过滤。


## 对可调用及其他属性的索引

**Indexing callables and other attributes**

> **注意** [数据库后端（默认）](#backends-database) 不支持此特性。

搜索/过滤器字段无需是Django模型字段。他们还可以是模型类的方法或属性。

此特性的一种用处，是对那些Django自动创建的、带有选项的字段的`get_*_display`方法的索引。

```python
from wagtail.search import index

class EventPage(Page):

    IS_PRIVATE_CHOICES = (
        (False, "公开的"),
        (False, "私有的"),
    )

    is_private = models.BooleanField(choices=IS_PRIVATE_CHOICES)

    search_fields = Page.search_fields + [
        # 对人类可读的字符串进行索引，以进行搜索
        index.SearchField('get_is_private_display'),

        # 对逻辑值进行索引，以进行过滤
        index.FilterField('is_private'),
    ]
```

可调用属性还提供到一种对相关模型字段的索引方法（Callables also provide a way to index fields from related models）。在[内联面板与模型集群](reference/pages.html#panels-inline-panels)的示例中，就是通过相关链接的标题，来对各个`BookPage`进行索引的。

```python
class BookPage(Page):
    
    # ...

    def get_related_link_titles(self):
        
        # 获取到标题清单，并将他们级联起来
        return '\n'.join(self.related_links.all().values_list('name', flat=True))

    search_fields = Page.search_fields + [
        # ...
        index.SearchField('get_realted_link_titles'),
    ]
```

<a name="indexing-custom-models"></a>
## 对定制模型进行索引

所有Django模型，都可以被索引与搜索。

要实现这一点，就要从`index.Indexed`进行继承，并将一些`search_fields`加入到该模型。

```python
from wagtail.search import index

class Book(index.Indexed, models.Model):
    title = models.CharField(max_length=255)
    genre = models.CharField(max_length=255, choices=GENRE_CHOICES)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    published_date = models.DateTimeField()

    search_fields = [
        index.SearchField('title', partial_match=True, boost=10),
        index.SearchField('get_genre_display'),

        index.FilterField('genre'),
        index.FilterField('author'),
        index.FilterField('published_date'),
    ]

# 因为此模型在其QuerySet中并没有一个搜索方法，因此就必须直接在后端调用搜索
>>> from wagtail.search.backends import get_search_backend
>>> s = get_search_backend()

# 运行一次对 Roald Dohl 所写的书的搜索
>>> roald_dahl = Author.objects.get(name="Roald Dahl")
>>> s.search("chocolate factory", Book.objects.filter(author=roald_dahl))
[<Book: Charlie and the chocolate factory>]
```
