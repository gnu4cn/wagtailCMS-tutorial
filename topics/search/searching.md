# 进行搜索

## 搜索的QuerySets

Wagtail的搜索特性，是建立在Django的 [QuerySet API](https://docs.djangoproject.com/en/stable/ref/models/querysets/)之上的。可搜索所有由模型所提供的Django QuerySet，以及那些已加入到搜索索引的、进行过滤的字段（You should be able to search any Django QuerySet provided the model and the fields being filtered on have been added to the search index）。

## 对页面的搜索

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

## 对图片、文档及定制模型的搜索

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

## 指定要搜索的字段

默认Wagtail将搜索所有已使用`index.SearchField`进行索引了的字段。

可使用`fileds`关键字参数，将搜索字段限制为确切的字段集合：

```sh
# 仅搜索标题字段
>>> EventPage.objects.search("Event", fields=["title"])
[<EventPage: Event 1>, <EventPage: Event 2>]
```

## 分面搜索

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

## 提升后的搜索结果

**Promoted search results**

“提升了的搜索结果”特性，允许网站编辑显式地将相关内容，链接到搜索条目，如此结果页面就能包含干预后的内容，作为搜索引擎结果的补充。

此功能是有社区贡献模块`search_promotions`所提供了。

