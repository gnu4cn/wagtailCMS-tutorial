# 搜索功能

Wagtail提供了全面且可扩展的搜索接口。此外，其还经由“站点编辑精选（Editor's Picks）”方式，提供提升搜索结果的方法。Wagtail还将一些有关通过搜索接口进行的查询的、简单统计数据收集了起来。


+ [建立索引](indexing.md)

    - [更新索引](indexing.md#updating-the-index)
    - [对额外字段进行索引](indexing.md#indexing-extra-fields)
    - [对定制模型建立索引](indexing.md#indexing-custom-models)


+ [进行搜索](searching.md)

    - [搜索的QuerySet](searching.md#searching-querysets)
    - [页面搜索视图示例](searching.md#an-example-page-search-view)
    - [提升后的搜索结果](searching.md#promoted-search-results)


+ [搜索后端](#backends)

    - [`AUTO_UPDATE`](backends.md#auto-update)
    - [`ATOMIC_REBUILD`](backends.md#atomic-rebuild)
    - [`BACKEND`](backends.md#backend)


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

请参阅 [关于搜索后端](backends.html)。
