# 性能问题

Wagtail以速度为设计目标，体现在编辑器界面与前端上，但在想要更好的性能，或需要处理非常高的流量时，下面就是一些从安装中压榨更多性能的技巧。

## 编辑器界面的性能

Wagtail开发者已经尽力将一个可以工作的Wagtail安装的外部依赖最小化了，目的就是令其尽可能简单的运作起来。尽管如此，仍可对一些默认设置进行配置，以获取更好的性能：

### 缓存方面

这里推荐使用 [Redis](http://redis.io/) 作为一个快速、持久的缓存。经由包管理器（在Debian或Ubuntu上：`sudo apt-get install redis-server`），并将`django-redis`添加到`requirements.txt`，且将其作为一个缓存后端进行开启：

```python
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/0',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'PASSWORD': '+7xYeaxVbiR/Kkm+gv5p2LoSALRpRmzSCqMTRC2KE2D+gHiDf4/7Sdhx+mW/szMtwEgZH96ZIJKUPJj/',
        }
    }
}
```

### 搜索方面

Wagtail有着对 [Elasticsearch](http://www.elasticsearch.org/)很强的支持 -- 同时对编辑器界面与站点用户来说 -- 但在没有 Elasticsearch时也可回滚到数据库搜索。比起Django用于文本搜索的ORM，Elasticsearch更为快速且更为强大，因此推荐安装Elasticsearch，或者使用一个像是 [Searchly](http://www.searchly.com/)这样的主机服务。

更多有关配置Elasticsearch下的Wagtail的内容，请参见[Elasticsearch后端](topics/search/backends.md#wagtailsearch-backends-elasticsearch)。

### 数据库方面

Wagtail在PostgreSQL、SQLite与MySQL上进行过测试。他也应工作在一些第三方数据库后端上（MS SQL已知可以工作但未进行测试）。这里推荐使用PostgreSQL作为生产用途。

## 模板方面


