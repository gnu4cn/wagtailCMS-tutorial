# 搜索功能

Wagtail提供了全面且可扩展的搜索接口。此外，其还经由“站点编辑精选（Editor's Picks）”方式，提供提升搜索结果的方法。Wagtail还将一些有关通过搜索接口进行的查询的、简单统计数据收集了起来。


+ [建立索引](#indexing)

    - [更新索引](#updating-the-index)
    - [对额外字段进行索引](#indexing-extra-fields)
    - [对定制模型建立索引](#indexing-custom-models)


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
## 索引的建立

要令到模型可视化，就需要将其加入到搜索索引中。所有页面、图片与文档都被索引起来，那么就可以立即开始对他们进行搜索了。

在某个页面或图片基类的子类中建立了额外字段时，可能会将这些新的字段，也加入到搜索索引中，从而令到用户的搜索查询，对这些子类的内容进行匹配。请参阅[对额外字段建立索引](#indexing-extra-fields)。

在希望将定制模型纳入搜索时，请参阅[建立定制模型的索引](#indexing-custom-models)。

### 更新索引

在搜索索引与数据库分离的情况下（比如使用Elesticsearch时），就要让二者保持同步状态。有两种方式可以完成此操作：使用搜索信号处理器，或周期性地调用`update_index`命令。处于最佳速度与可靠性的考虑，最好的做法是在可能的情况下二者同时使用。

### 关于信号处理器

`wagtailsearch`库提供了一些绑定到所有已索引模型的保存/删除信号的信号处理器。这将自动在已于`WAGTAILSEARCH_BACKENDS`中注册的所有后端中，添加及删除相应他们。在`wagtail.search`应用装入时，这些信号处理器自动进行了注册。

**关于`update_index`命令**

Wagtail还提供了用于从头开始重建索引的一个命令

```bash
./manage.py update_index
```

推荐每周运行一次此命令，以及在以下时刻运行他：

+ 在经由脚本（比如导入内容后）创建了页面后
+ 在对模型或搜索的配置进行了修改之后

因为此命令运行时的搜索不会返回结果，因此要避免在峰值时段运行此命令。

<a name="indexing-extra-fields"></a>
### 对额外字段进行索引

> **警告** 数据库后端并不支持对额外字段的索引。在使用数据库后端是，经由`search_fields`所定义的所有其他字段，都会被忽略。


必要要将字段显式地加入到那些由`Page`基类所派生的模型的`search_fields`字段，以便对这些字段进行搜索/过滤。这是通过对`search_fields`进行覆写，来将一个额外的`SearchField`/`FilterField`字段给他，而完成的。

__示例__

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

### 关于`index.SearchField`

指定用于在模型上执行全文搜索的字段，通常是指文本字段。

**选项**

+ `partial_match(boolean)` -- 将该选项设置为真时，允许结果在部分词上匹配。比如此选项默认对标题就设置为真，那么在某个页面的标题为`Hello World!`时，若用户在搜索框中输入的是`Hel`，则该页面将会被找到。

+ `boost(int/float)`  -- 此选项允许将一些字段设置为较其他字段更为重要。将某字段的此选项设置为一个较高的数字，将那些在该字段匹配的页面，在结果共排序靠前。通常页面标题字段的该选项被设置为`2`，所有其他字段的该选项被设置为`1`。

+ `es_extra(dict)` -- 指定该字段允许开发者设置或覆写Elasticsearch映射中该字段上的所有设置。在打算使用某些Wagtail尚不支持的Elasticsearch的特性时，就要用到此选项。


### 关于`index.FilterField`

指定要加入到搜索索引的字段，但这些字段不用于全文搜索。而是可以在搜索结果上运行过滤器。


### 关于`index.RelatedFields`

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

__`index.RelatedFields`上的过滤__

使用`QuerySet`编程接口在`index.RelatedFields`中的所有`index.FilterFields`上进行过滤，都是不可能的。不过这些字段既然有被索引起来，那么就有可能通过手动查询Elasticsearch来用到他们。

Wagtail计划在未来的发行中，实现经由`QuerySet`在`index.RelatedFields`上的过滤。


<a name="indexing-callable-fields"></a>
### 对可调用及其他属性的索引

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
### 对定制模型进行索引

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


### 进行搜索

__搜索的QuerySets__

Wagtail的搜索特性，是建立在Django的 [QuerySet API](https://docs.djangoproject.com/en/stable/ref/models/querysets/)之上的。可搜索所有由模型所提供的Django QuerySet，以及那些已加入到搜索索引的、进行过滤的字段（You should be able to search any Django QuerySet provided the model and the fields being filtered on have been added to the search index）。

__对页面的搜索__

Wagtail提供了搜索页面的一个捷径：`.search()` `QuerySet`方法。可在所有`PageQuerySet`上调用该方法。比如：

```bash
# 对未来的 EventPage进行搜索
>>> from wagtail.core.models import EventPage
>>> EventPage.objects.filter(date__gt=timezone.now()).search("Hello world!")
```

所有其他`PageQuerySet`的方法，都可以与`.search()`一起使用。比如：

```bash
# 搜素所有活动索引下在线显示的EventPage
>>> EventPage.objects.live().descendant_of(events_index).search("Event")
[<EventPage: Event 1>, <EventPage: Event 2>]
```

> **注意** 方法`search()`将吧`QuerySet`转换为某个Wagtail `SearchResults`类（取决于后端情况）的一个实例。这意味着在调用`search()`方法之前进行过滤。

__对图片、文档及定制模型的搜索__

Wagtail的文档及图片模型，与页面模型一样，均在他们的 QuerySets 上提供了一个`search`方法：

```bash
>>> from wagtail.images.models import Image

>>> Image.objects.filter(uploaded_by_user=user).search("Hello")
[<Image: Hello>, <Image: Hello world!>]
```

对于[定制模型](#indexing-custom-models)，可通过直接在搜索后端，使用`search`方法进行搜索：

```bash
>>> from myapp.models import Book
>>> from wagtail.search.backends import get_search_backend

# 对书籍进行搜索
>>> s = get_search_backend
>>> s.search("Great", Book)
[<Book: Great Expectations>, <Book: The Great Gatsby>]
```

也可将一个 QuerySet 传递进 `search` 方法，从而实现将过滤器添加到搜索结果（you can also pass a QuerySet into the `search` method which allows you to add filters to your search results）：

```bash
>>> from myapp.models import Book
>>> from wagtail.search.backends import get_search_backend

# 对书籍进行搜索
>>> s = get_search_backend
>>> s.search("Great", Book.objects.filter(published_date__year__lt=1900))
[<Book: Great Expectations>]
```

### 指定要搜索的字段

默认Wagtail将搜索所有已使用`index.SearchField`进行索引了的字段。

可使用`fileds`关键字参数，将搜索字段限制为确切的字段集合：

```sh
# 仅搜索标题字段
>>> EventPage.objects.search("Event", fields=["title"])
[<EventPage: Event 1>, <EventPage: Event 2>]
```

### 分面搜索

**Faceted search**

Wagtail带有对分面搜索的支持，分面搜索是一种基于分类字段（诸如类别或页面类型）的过滤（Wagtail supports faceted search which is kind of filtering based on a taxonomy field(such as category or page type)）。

方法`.facet(field_name)`返回的是一个`OrderedDict`。字典中的键是相关对象的IDs，这些对象则是已被该字段引用到的；字典中的值，是到各个ID的引用的数目。方法结果字典，是依引用数目降序排序的。

下面是在搜索结果中找出最常用页面类型的代码：

```sh
>>> Page.objects.search("Test").facet("content_type_id")

# 注意：这些键是与某个 Content_Type 对象对应的ID，值则是
# 那个类型下所返回页面的数目
OrderedDict([
    ('2', 4), # 有4个页面有着 content_type_id == 2
    ('1', 2), # 有2个页面有着 content_type_id == 1
])
```

## 改变搜索行为

**Changing search behaviour**

### 搜索运算符

搜索运算符指明了在用户输入了多个搜索词条时，搜索应如何进行。有两个可能的取值：

+ `or` -- 结果必须匹配至少一个词条（这是Elasticsearch默认的行为）
+ `and` -- 结果必须匹配所有词条（这是数据库搜索的默认行为）

二者都有利有弊。`or`运算符将返回更多的结果，但会倾向于包含很多的不相关结果。`and`运算符则仅返回那些包含了所有搜索词条的结果，但要求用户的查询更为精准。

在以相关度进行排序时，推荐使用`or`运算符，在以所有其他线索排序时使用`and`运算符（注意：当前数据库后端并不支持以相关度为依据的排序）。

下面是一个使用`operator`关键字参数的示例：

```sh
# 数据库中包含了一个都有以下条目的“Thing”的模型：
# - Hello world
# - Hello
# - World

# 使用 “or” 运算符的搜索
>>> s = get_search_backend()
>>> s.search("Hello world", Things, operator="or")

# 所有记录都被返回，因为他们都保护了“hello”或“world”
[<Thing: Hello World>, <Thing: Hello>, <Thing: World>]

# 以“and”运算符进行搜索
>>> s = get_search_backend()
>>> s.search("Hello world", Things, operator="and")

# 仅返回“hello world”, 因为那是仅有的包含了“hello”与“world”两个词条的数据库条目
[<Thing: Hello world>]
```

对于页面、图片及文档模型，在QuerySet的`search`方法上，也是支持`operator`关键字参数的：

```sh
>>> Page.objects.search("Hello world", operator="or")

# 所有包含了“hello”或“world”的页面都被返回
[<Page: Hello World>, <Page: Hello>, <Page: World>]
```

### 对排序进行定制

默认在后端支持的情况下，搜索结果是以想高度进行排序的。要保留QuerySet的既有排序，就要将`search()`方法上的`order_by_relevance`关键字参数设置为`False`。

比如：

```sh
# 获取一个依日期排序的活动清单
>>> EventPage.objects.order_by('date').search("Event", order_by_relevance=False)

# 这就是依日期排序的活动了
[<EventPage: Easter>, <EventPage: Halloween>, <EventPage: Christmas>]
```

### 使用分值来对结果进行批注

**Annotating results with score**

对每项匹配结果，Elasticsearch会计算出一个“分值”，所谓分值，就是根据用户的查询，用于表示该项结果相关度的一个数字。结果通常是基于分值进行排序的。

在某些场合，对分值的访问是很有用的（诸如程序实现的对不同模型的两个查询结合起来）。通过调用`SearchQuerySet`上的`.annotate_score(field)`方法，可将分支加入到每次查询。

比如：

```sh
>>> events = EventPage.objects.search('Event').annotate_score('_score')
>>> for event in events:
...    print(event.title, event._score)
...
("Easter", 2.5),
("Haloween", 1.7),
("Christmas", 1.5),
```

请注意分值本身是随意的，且仅在对同样查询下的结果比较才有用处（note that the score itself is arbitrary and it is only useful for comparison of results for the same query）。

## 页面搜索视图的示例

下面是一个可用于将“搜索”页面加入到站点的Django视图的示例：

```python
# views.py

from django.shortcuts import render

from wagtail.core.models import Page
from wagtail.search.models import Query

def search(request):

    # 搜索
    search_query = request.GET.get('query', None)
    if search_query:
        search_results = Page.objects.live().search(search_query)

        # 对该查询进行记录，从而令到Wagtail可以给出改进的结果建议
        Query.get(search_query).add_hit()

    else:
        search_results = Page.objects.none()

    # 对模板进行渲染
    return render(request, 'search_results.html', {
        'search_query': search_query,
        'search_results': search_results,
    })
```

以下是一个与该试图一同工作的模板：

{% raw %}
    {% extends "base.html" %}
    {% load wagtailcore_tags %}

    {% block title %}搜索{% endblock %}

    {% block content %}
        <form action="{% url 'search' %}" method="get">
            <input type="text" name="query" value="{{ search_query }}" />
            <input type="submit" value="搜索" />
        </form>

        {% if search_results %}
            <ul>
                {% for result in search_results %}
                    <li>
                        <h4><a href="{% pageurl result %}">{{ result }}</a></h4>
                        {% if result.search_description %}
                            {{ result.search_description|safe }}
                        {% endif %}
                    </li>
                {% endfor %}
            </ul>
        {% elif search_query %}
            未找到结果
        {% else %}
            请将搜索词条输入到搜索框中
        {% endif %}
    {% endblock %}
{% endraw %}

### 提升后的搜索结果

**Promoted search results**

“提升了的搜索结果”特性，允许网站编辑显式地将相关内容，链接到搜索条目，如此结果页面就能包含干预后的内容，作为搜索引擎结果的补充。

此功能是有社区贡献模块`search_promotions`所提供了。


## 关于搜索后端


Wagtail有着对多种后端的支持，从而提供了使用数据库的搜索，或使用诸如Elasticsearch这样的外部服务进行搜索的选择。默认启用的是数据库后端。

可使用`WAGTAILSEARCH_BACKENDS`设置，来配置要使用的后端：

```python
WAGTAILSEARCH_BACKENDS = {
    'default': {
        'BACKEND': 'wagtail.search.backends.db',
    }
}
```

<a name="AUTO_UPDATE"></a>
### `AUTO_UPDATE`

默认Wagtail将自动保持所有索引处于更新状态。此默认行为在编辑内容时，将对性能造成影响，尤其是在索引驻留在外部服务上时。

`AUTO_UPDATE`设置，允许在单个索引基础上，关闭默认行为：

```python
WAGTAILSEARCH_BACKENDS = {
    'default': {
        'BACKEND': ...,
        'AUTO_UPDATE': False,
    }
}
```

如关闭了自动更新，那么就必须定期运行`update_index`命令，以保持索引与数据库同步。

### `ATOMIC_REBUILD`

> **警告** 此选项在Elasticsearch 5.4及更高版本上不会工作，是因为[别名处理中的一个bug](https://github.com/elastic/elasticsearch/issues/24644) 影响到这些版本。

默认（使用Elasticsearch后端时）在`update_index`运行时，Wagtail会删除索引并从头开始重建出来。这就会导致在重建完成之前搜索引擎不会返回结果，同时这也有着发生了错误时无法回滚的风险。

将`ATOMIC_REBUILD`设置项目，设置为`True`，就会令到Wagtail重建为一个独立的索引，而在新的索引完全建立之前，保持原有的索引处于活动状态。在重建完成之后，新旧索引进行原子交换，且旧的索引被删除。

### `BACKEND`

下面是Wagtail原生支持的后端清单。

__数据库后端（默认支持）__

`wagtail.search.backends.db`

数据库后端是甚为基础的，而仅是打算供开发与小型站点所用。其无法依相关度对结果进行排序，从而严重妨碍了其在对大量页面集进行搜索时的可用性。

数据库后端不具备对以下特性的支持：

+ 在`Page`基类的子类中字段的搜索（除非该类是直接进行搜索的）
+ [对可调用属性与其他属性的索引](#indexing-callable-fields)
+ 将重音字符转换成ASCII字符（注：accendted characters）

在上述任何一个特性都是重要的情况下，就要使用Elasticsearch了。

__PostgreSQL 的后端__

`wagtail.contrib.postgres_search.backend`

在使用PostgreSQL作为数据库，且站点页面数少于一百万时，就可能打算使用此后端了。

请参阅[PostgreSQL的搜索引擎](reference/contrib.html#postgres-search)以了解更多知识。


__Elasticsearch 后端__

*Wagtail 2.1 的改动： 加入了对 Elasticsearch 6.x 的支持*

Wagtail支持 Elasticsearch 的版本2、5与6。请使用对应版本的后端：

`wagtail.search.backends.elasticsearch2` （Elasticsearch 2.x）

`wagtail.search.backends.elasticsearch5` （Elasticsearch 5.x）

`wagtail.search.backends.elasticsearch6` （Elasticsearch 6.x）

使用此后端的前提，是先要有[Elasticsearch](https://www.elastic.co/downloads/elasticsearch)服务本身，以及通过`pip`安装上[elasticsearch-py](http://elasticsearch-py.readthedocs.org/)这个包。该包的大版本号要与所安装的Elasticsearch的版本匹配：

```sh
$ pip install "elasticsearch>=2.0.0,<3.0.0" # 对于Elasticsearch 2.x

$ pip install "elasticsearch>=5.0.0,<6.0.0" # 对于Elasticsearch 5.x

$ pip install "elasticsearch>=6.0.0,<6.3.1" # 对于Elasticsearch 6.x
```

> **注意** 版本 `6.3.1` 的 Elasticsearch客户端库与Wagtail不兼容。请使用 `6.3.0`或更早版本。


后端实在设置中配置的：

```python
WAGTAILSEARCH_BACKENDS = {
    'default': {
        'BACKEND': 'wagtail.search.backends.elasticsearch2',
        'URLS': ['http://localhost:9200'],
        'INDEX': 'wagtail',
        'TIMEOUT': 5,
        'OPTIONS': {},
        'INDEX_SETTINGS': {},
    }
}
```

与`BACKEND`不同，其他键是可选的，且默认为上面的那些值。在`OPTIONS`中定义的所有键，都被直接作为区分大小写的关键字参数（比如`'max_retries': 1`），传递给 Elasticsearch的构造器。

`INDEX_SETTINGS`则是一个用于对默认创建索引方式设置进行覆写的字典。该创建索引方式设置项，定义在模块`wagtail/wagtail/wagtailsearch/backends/elasticsearch.py`模块里`ElasticsearchSearchBacken`类的内容。将加入所有的新键，对于既有键，如其不是一个字典，那么都将以新的值进行替换。下面是一个如何配置分片数，以及将意大利语的语言分析器作为默认分析器的示例：

```python
WAGTAILSEARCH_BACKENDS = {
    'default': {
        ...,
        'INDEX_SETTINGS': {
            'settings': {
                'index': {
                    'number_of_shards': 1,
                },
                'analysis': {
                    'analyzer': {
                        'default': {
                            'type': 'italian'
                        }
                    }
                }
            }
        }
    }
}
```

若不在开发或生产环境选择运行一个Elasticsearch服务器，那么有着多个可用的第三方主机服务，包括[Bonsai](https://bonsai.io/signup)，该站点提供了一个适合与测试与开发的免费帐号。要使用Bonsai：

+ 在Bonsai注册一个帐号
+ 使用Bonsai的仪表盘创建一个集群
+ 使用Bonsai仪表盘中的该集群URL，配置`WAGTAILSEARCH_BACKENDS`中的`URLS`条目
+ 运行`./manage.py update_index`命令


__Amazon AWS 的Elasticsearch__

Wagtail的Elasticsearch后端，是与[Amazon 的Elasticsearch服务](https://aws.amazon.com/elasticsearch-service/)兼容的，但需要额外配置，以处理基于IMA的认证。这可通过[requests-aws4auth](https://pypi.python.org/pypi/requests-aws4auth) `pip` 包，与以下的配置来完成：

```python
from elasticsearch import RequestsHttpConnection
from requests_aws4auth import AWS4AUTH

WAGTAILSEARCH_BACKENDS = {
    'default': {
        'BACKEND': 'wagtail.search.backends.elasticsearch2',
        'INDEX': 'wagtail',
        'TIMEOUT': 5,
        'HOSTS': [{
            'host': 'YOURCLUSTER.REGION.es.amazonaws.com',
            'port': 443,
            'use_ssl': True,
            'verify_certs': True,
            'http_auth': AWS4AUTH('ACCESS_KEY', 'SECRET_KEY', 'REGION', 'es'),
        }],
        'OPTIONS': {
            'connection_class': RequestsHttpConnection,
        }
    }
}
```

__构造自己的后端__

**Rolling Your Own**

Wagtail搜索后端实现了在`wagtail/wagtail/wagtailsearch/backends/base.py`中的接口。在最低限度下，后端的`search()`方法必须返回一个对象集合或`model.objects.none()`。而对于一个具有完整特性的搜索后端，请在`elasticsearch.py`中查看Elasticsearch的后端代码。
