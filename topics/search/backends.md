# 关于搜索后端


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
## `AUTO_UPDATE`

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

## `ATOMIC_REBUILD`

> **警告** 此选项在Elasticsearch 5.4及更高版本上不会工作，是因为[别名处理中的一个bug](https://github.com/elastic/elasticsearch/issues/24644) 影响到这些版本。

默认（使用Elasticsearch后端时）在`update_index`运行时，Wagtail会删除索引并从头开始重建出来。这就会导致在重建完成之前搜索引擎不会返回结果，同时这也有着发生了错误时无法回滚的风险。

将`ATOMIC_REBUILD`设置项目，设置为`True`，就会令到Wagtail重建为一个独立的索引，而在新的索引完全建立之前，保持原有的索引处于活动状态。在重建完成之后，新旧索引进行原子交换，且旧的索引被删除。

## `BACKEND`

下面是Wagtail原生支持的后端清单。

### 数据库后端（默认支持

`wagtail.search.backends.db`

数据库后端是甚为基础的，而仅是打算供开发与小型站点所用。其无法依相关度对结果进行排序，从而严重妨碍了其在对大量页面集进行搜索时的可用性。

数据库后端不具备对以下特性的支持：

+ 在`Page`基类的子类中字段的搜索（除非该类是直接进行搜索的）
+ [对可调用属性与其他属性的索引](#indexing-callable-fields)
+ 将重音字符转换成ASCII字符（注：accendted characters）

在上述任何一个特性都是重要的情况下，就要使用Elasticsearch了。

### PostgreSQL 的后端

`wagtail.contrib.postgres_search.backend`

在使用PostgreSQL作为数据库，且站点页面数少于一百万时，就可能打算使用此后端了。

请参阅[PostgreSQL的搜索引擎](reference/contrib.html#postgres-search)以了解更多知识。

<a name="wagtailsearch-backends-elasticsearch"></a>
### Elasticsearch 后端

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


### Amazon AWS 的Elasticsearch

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

### 构造自己的后端

**Rolling Your Own**

Wagtail搜索后端实现了在`wagtail/wagtail/wagtailsearch/backends/base.py`中的接口。在最低限度下，后端的`search()`方法必须返回一个对象集合或`model.objects.none()`。而对于一个具有完整特性的搜索后端，请在`elasticsearch.py`中查看Elasticsearch的后端代码。
