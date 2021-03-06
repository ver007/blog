title: 《High Performance Django》阅读笔记（一）
slug: high-performance-django-note-1
tags: High Performance Django, Django
date: 2015-09-26

[![](/static/images/reading/s28041145.jpg)](http://book.douban.com/subject/26359018/)


一句话点评：老司机的经验之谈，物有所值。


## 第一章：The Big Picture

作者开篇就提到大家总说 Django 性能不行，但是实际上
有很多高性能的站点是使用 Django 开发的。

> Django’s scaling success stories are almost too numerous to list at this point. It backs Disqus,
> Instagram, and Pinterest. Want some more proof? Instagram was able to sustain over 30
> million users on Django with only 3 engineers (2 of which had no back-end development
> experience). Disqus serves over 8 billion page views per month. Those are some huge
> numbers. These teams have proven Django most certainly does scale. Our experience here
> at Lincoln Loop backs it up. We’ve built big Django sites capable of spending the day on
> the Reddit homepage without breaking a sweat.

在作者的公司，他们开发高性能 Django 站点的准则就是 **simplicity** :

* Using as few moving parts as possible to make it all work. “Moving parts” may be
servers, services or third-party software.
* Choosing proven and dependable moving parts instead of the new hotness.
* Using a proven and dependable architecture instead of blazing your own trail.
* Deflecting traffic away from complex parts and toward fast, scalable, and simple parts .

> Simple systems are easier to scale, easier to understand, and easier to develop. 

构建高性能 Web 应用通常需要关注一下几点：

* 数据库。关系型数据库通常是整个技术栈中最慢最复杂的部分，一个办法是改用 NoSQL 数据库，不过
  大多数情况下都可以通过缓存解决。
* 模板。我们可以用一个更快的模板引擎替换 Django 自带的模板引擎，不过即便是这样模板仍旧是
  整个技术栈中第二慢的部分。我们仍然可以通过缓存解决这个问题。
* Python。Python 在通常情况下已经足够快了。我们可以使用 Web 加速器（比如：Varnish）缓存服务器响应，
  在请求进入到 Python 那一层之前就返回相应。

这章作者一直在强调缓存，"CACHE ALL THE THINGS"。无论你怎么优化你的技术栈，没有比缓存更快的优化方案。
说到缓存可能大家可能会顾虑缓存过期的问题，作者说了现在先别关心这个问题，之后会给出解决方案。

作者提到一般的网站都保护这几层：负载均衡器，Web 加速器，APP 服务器，缓存，数据库。

顺便提了一个 HTTPS 的负载均衡配置方法：客户端与负载均衡器之间使用 HTTPS，负载均衡器与后端服务之间使用 HTTP。这样既保证了安全又可以减少 HTTPS 对性能的影响。


## 第二章 The Build

### 小技巧

* 本地开发环境应该尽可能的与线上环境一致：相同的数据库，相同的操作系统以及相同的软件版本。。。
* 组织 settings 文件，创建一个基础的配置文件 settings/base.py ，然后再为开发，测试，部署分别创建一个配置文件，一些重要的配置信息可以通过环境变量获取
    * settings/base.py
    * settings/dev.py
    * settings/deploy.py

  这里作者有提到一个小技巧，那就是环境变量的值都是字符串，那么如果将值转换为布尔值，元组甚至字典呢？答案就是可以使用 ast 模块:

        >>> ast.literal_eval('True')
         True

        >>> ast.literal_eval('1, 2, 3')
         (1, 2, 3)

        >>> ast.literal_eval('{"foo": "bar"}')
         {'foo': 'bar'}

### 小心第三方 APP

在决定使用某个第三方 APP 之前，先回答下面几个问题：

* 它真的符合你的需求吗？还是只是有点相近？
* 它是个健康的项目吗？
    * 维护者有一个好的追踪记录吗？
    * 文档写的好吗？
    * 测试覆盖率够吗？
    * 社区怎样（贡献者，pull requests 等等）？
    * 还在处于活跃开发吗？
    * 有很多旧的 issues 和 pull requests 吗？
* 性能咋样？
    * 它会产生多少数据库查询？
    * 易于缓存吗？
* 跟你项目的其他部分有冲突吗？
* 它的授权协议跟你的项目兼容吗？

不再维护以及不稳定的第三方应用很快就会成为你的项目的负债。
可以尝试阅读源代码，然后从中找出你的项目需要的代码然后应用到项目中。

### 找出性能问题

可以使用下面这些工具

* [Django Debug Toolbar](http://django-debug-toolbar.readthedocs.org/)
* [django-debug-panel](https://github.com/recamshak/django-debug-panel)
* [django-devserver](https://github.com/dcramer/django-devserver)

观察页面性能：

* 执行了多少条 SQL 语句？
* 有多少时间花费在数据库上？
* 执行了什么特殊的查询操作，每次查询花费多长时间？
* 这些查询是有什么代码生成的？
* 渲染页面都用到了哪些模板？
* 冷/热缓存是如果影响性能的？（提示：可以使用 settings 来切换缓存）

### 哪里需要优化

#### 数据库优化

* 减少查询次数
    * 使用 `select_related`, `prefetch_related`, (提示：`prefetch_related` 要放在查询的最后，不然会没有效果。)
* 减少查询时间
    * 不要忘记加索引（索引也是有代价的，每次对数据库进行写操作都需要更新索引）
    * 某些情况下 join 查询性能很差，在这种情况下两条查询语句比一条 join 耗时更少。
* 限制结果数
    * 留意 `.all()` 只取需要的结果数, `queryset[:20]`
* count 查询很慢。可以的话，不要使用 count。比如使用 `.exists()` 代替 count 进行判断记录是否存在。
* generic 外键。generic 外键是个很 cool 的功能，但是它会生成一些特别复杂的查询，所以可能的话，不要使用它。如果你一定要用的话，记得这是个需要缓存的地方。
* 优化 model 方法。如果某个 model 方法在一个请求内会多次被调用，可以使用 `cache_property` 缓存方法解决（缓存只在该请求内有效）

        from django.utils.functional import cached_property
        
        class TheModel(models.Model):
        
            @cached_property
            def expensive(self):
               # ...
               return result

* 结果太大了，包含了不需要的字段。使用 `defer`, `only`, `values`, `values_list` 限制结果大小:

        posts = Post.objects.all().defer('body')
        posts = Post.objects.all().only('title')
        posts = Post.objects.all().values('id')
        posts = Post.objects.all().values_list('id')
        posts = Post.objects.all().values_list('id', flat=True)

* 缓存查询结果。这里提到两个库: [Johnny Cache](https://johnny-cache.readthedocs.org/en/latest/ ) 和 [Cache Machine](https://cache-machine.readthedocs.org/en/latest/) 这两个库的原理都是在 ORM 和数据库中间加了一个缓存层，将 ORM 生成的 SQL 作为 key 来缓存查询结果。
* 只读 replicas。对那些读远远大于写的站点，可以考虑从 只读 replicas 中读取数据，实现读写分离。减少主库的负担优化性能。
* raw 查询。如果感觉 ORM 有点慢话，可以考虑使用 `raw` 方法执行原生的 SQL 语句。
* 反范式。这种方法有个问题就是每次更新的时候都需要同时更新其他表中相关的冗余字段。
* 使用其他数据存储方式。比如： Postgres, redis, mongodb，使用 Elasticsearch 进行全文检索等。
  需要注意的是，在生产环境下新增一个服务并无法没有代价的。作为开发者我们可以不太在意这个，但是系统需要
  支持，配置，监控，备份等。新增服务的时候要考虑到这些代价以及你的系统管理员的意见。
* sharding。99.9% 的网站都不需要用到数据库的 sharding 功能，所以只有在你确信你遇到了那 0.1% 的时候
  再使用 sharding 功能。

### 模板优化

应该缓存模板中一切可以缓存的东西。

* 俄罗斯套娃式缓存。在一个模板里缓存嵌套缓存，就像俄罗斯套娃一样，一层套一层。比如:

        {% cache MIDDLE_TTL "post_list" request.GET.page %}
            {% include "inc/post/header.html" %}
            <div class="post-list">
            {% for post in post_list %}
                {% cache LONG_TTL "post_teaser_" post.id post.last_modified %}
                    {% include "inc/post/teaser.html" %}
                {% endcache %}
            {% endfor %}
            </div>
        {% endcache %}
* 自定义一个支持通过 url 参数刷新模板缓存的 cache 标签，这样就可以随时刷新缓存了

### 随后处理耗时的任务

可以把耗时的，不需要同步知道结果的任务放到类似 celery 的任务队列中异步执行，
从而减少请求——响应的处理时间。下面这些任务可以考虑放到 celery 中：

* 第三方 API 的调用
* 发邮件
* 非常复杂的计算（视频处理，大量的数字处理等）

对于使用 celery 作者提到了一下小提示：

* 不要将 model 实例作为任务的参数，可以改用传主键的方式。因为在这期间 model 的数据可能已经发生改变了，
  还有就是那个 model 实例可能不支持序列化。
* 保持任务小，不要再一个任务中执行太多的工作。把一个任务分割成多个任务，一方面可以使用多核或多 worker 的
  方式加速任务执行，另一方面，单个任务可以很快的执行完方便安全快速的重启 worker，因为一个 worker 重启时
  会等待正在执行的问题完成，保持任务小巧的话，可以加快部署时间。
* 可以考虑使用 celery 的 beat 功能去自习定时任务。

### 前段优化

* 压缩 css 和 javascript（min, gzip, 版本化静态文件）。（个人建议：版本化文件应该类似这样 foo-xxyy.js 而不是这样 foo.js?v=xxyy ，主要是方便使用 CDN，防止出现缓存未过期的情况。）
* 压缩图片。
* 使用 CDN 服务静态文件。

### 文件上传

可以考虑使用分布式文件系统或者云存储来存储上传的文件。使用云存储的时候要考虑备用方案，万一服务不可用怎么办。


### 测试

好的测试用例是健康代码的强有力的基石。测试应该覆盖到你代码中最复杂，最重要，最容易出问题的地方。

#### 自动化测试以及持续集成

一个持续集成系统可以让开发者在开发进度的早期就发现问题，通过持续集成系统来执行
自动化测试以及检查你的代码的健康度。作者提到了他们的检查点：

* 单元测试
* 代码覆盖率
* PEP8/linting
* 使用 Selenium 进行功能测试
* 所以 Jmeter 进行性能测试
