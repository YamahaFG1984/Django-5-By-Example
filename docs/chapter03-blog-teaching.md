# Chapter03 Blog 项目详解教学

> 对应目录：`Chapter03/`（Blog 项目在全书三章里的最终形态，在 Chapter02 基础上迭代）

## 一、这一章的定位

Chapter03 把 Chapter02 已经具备"URL 规范 + 分页 + 评论 + 邮件分享"的博客，进一步升级为一个**具备内容发现能力的完整博客系统**：读者可以按标签浏览、全文搜索、订阅 RSS；系统会根据标签自动推荐相似文章；正文支持 Markdown 排版；搜索引擎能通过 `sitemap.xml` 更好地收录站点。同时数据库从 SQLite 正式切换到了 **PostgreSQL**，这是因为搜索功能依赖 PostgreSQL 专有的 `pg_trgm`（三元组相似度）扩展。

有个小细节值得一提：`Chapter03/prompts/task.md` 里有一段提示词，内容是"如何把标签筛选页也加进 sitemap.xml"——这更像是仓库作者留下的一个**练习/思考题草稿**（用于配合 AI 工具做扩展练习），并不是书本正文的一部分，当前的 `sitemaps.py` 里其实还没有实现这个扩展，只做了文章的 sitemap。

## 二、数据库切换：SQLite → PostgreSQL

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': config('DB_NAME'),
        'USER': config('DB_USER'),
        'PASSWORD': config('DB_PASSWORD'),
        'HOST': config('DB_HOST'),
    }
}
```
所有连接参数都通过 `python-decouple` 的 `config()` 从环境变量读取（对应 `docker-compose.yml` 里 `db` 服务：`postgres:16.2`）。切换的直接原因是：`TrigramSimilarity` 全文相似度搜索是 PostgreSQL 的 `pg_trgm` 扩展提供的能力，SQLite 没有等价功能。

`INSTALLED_APPS` 新增了四项：
```python
'django.contrib.sites',      # sitemap/feed 框架依赖 SITE_ID 拼接完整域名
'django.contrib.sitemaps',   # 站点地图框架
'django.contrib.postgres',   # 解锁 TrigramSimilarity 等 PostgreSQL 专有 ORM 功能
'taggit',                    # 第三方标签库 django-taggit
```
以及顶层新增 `SITE_ID = 1`（`django.contrib.sites` 要求）。

`requirements.txt` 相应新增：`django-taggit==5.0.1`、`Markdown==3.6`、`psycopg==3.1.18`。

## 三、数据模型层：只加了一行，却带来一整条新迁移链

```python
from taggit.managers import TaggableManager
...
class Post(models.Model):
    ...
    tags = TaggableManager()
```
`Post` 模型本身**只多了这一个字段**，但背后触发了两条新迁移：

- **`0004_post_tags.py`**：给 `Post` 加 `tags` 字段，`through='taggit.TaggedItem'`——`TaggableManager` 底层是标准的"多对多 + 中间表"模式，`taggit.Tag` 存标签本身，`taggit.TaggedItem` 存"哪个对象打了哪个标签"的关联，支持任意模型复用（这也是它能通过 `GenericForeignKey` 给不同模型挂标签的原理，Chapter04+ 的书签项目会更深入用到这套机制）。
- **`0005_trigram_ext.py`**：
  ```python
  from django.contrib.postgres.operations import TrigramExtension
  operations = [TrigramExtension()]
  ```
  在数据库里执行 `CREATE EXTENSION pg_trgm`，为后面的全文相似度搜索开启底层能力。这是**数据库级别的迁移**（不建表、不改字段，而是给 PostgreSQL 实例开一个扩展），教学意义在于告诉你 Django 迁移系统不仅能管理表结构，还能管理数据库扩展这种基础设施级配置。

`Comment` 模型没有任何变化，`Meta`/`Admin` 配置也和 Chapter02 完全一致。

## 四、视图层：本章改动最集中的地方

### 1. `post_list`：从纯分页升级为"标签筛选 + 分页"

```python
def post_list(request, tag_slug=None):
    post_list = Post.published.all()
    tag = None
    if tag_slug:
        tag = get_object_or_404(Tag, slug=tag_slug)
        post_list = post_list.filter(tags__in=[tag])
    paginator = Paginator(post_list, 3)
    ...
    return render(request, 'blog/post/list.html', {'posts': posts, 'tag': tag})
```
- **`tag_slug=None` 默认参数**：同一个视图函数既服务于"全部文章列表"（`/blog/`，不传 `tag_slug`）又服务于"按标签筛选"（`/blog/tag/django/`，传 `tag_slug`）——一个函数复用两条路由，靠 URLconf 层面决定要不要传这个参数。
- 注意这里视图函数**改回了纯 FBV**（`Paginator` 手写版），`urls.py` 里对应地把 `PostListView.as_view()` 注释掉、换回 `views.post_list`：
  ```python
  path('', views.post_list, name='post_list'),
  # path('', views.PostListView.as_view(), name='post_list'),
  ```
  原因很直接：`ListView` 是通用类视图，不方便优雅地接收并处理"可选的 tag_slug 参数 + 按标签过滤查询集"这种定制逻辑，所以书本这里从 CBV 又切回了手写 FBV——这是"当需求变复杂时，选择更灵活的函数视图而非硬套通用类视图"的实际权衡案例，`PostListView` 类本身还留在代码里没删，仅是不再被路由使用。

### 2. `post_detail`：新增"相似文章推荐"算法

```python
post_tags_ids = post.tags.values_list('id', flat=True)
similar_posts = Post.published.filter(tags__in=post_tags_ids).exclude(id=post.id)
similar_posts = similar_posts.annotate(
    same_tags=Count('tags')
).order_by('-same_tags', '-publish')[:4]
```
这是本章最有含金量的 ORM 用法，逐行拆解：
1. `post.tags.values_list('id', flat=True)`：拿到当前文章所有标签的 id 列表（`flat=True` 让结果是 `[1, 3, 5]` 而不是 `[(1,), (3,), (5,)]`）。
2. `Post.published.filter(tags__in=post_tags_ids)`：找出所有"至少共享一个标签"的已发布文章。
3. `.exclude(id=post.id)`：排除自己。
4. `.annotate(same_tags=Count('tags'))`：**关键一步**——对每篇候选文章，统计它与当前文章"重叠的标签数"有多少（因为前面 filter 已经限定 tags 在 post_tags_ids 范围内，所以这里 `Count('tags')` 统计的正是共同标签数，而不是该文章的全部标签数）。
5. `.order_by('-same_tags', '-publish')[:4]`：按"共同标签数"降序排列，共同标签数相同时按发布时间降序，只取前 4 篇。

这套"用标签重叠度做相似度排序"的算法不需要任何机器学习，纯靠 SQL 聚合就实现了推荐功能，是这本书里最经典的 ORM 教学案例之一。

### 3. `post_search`：PostgreSQL 全文相似度搜索（本章第二个亮点）

```python
def post_search(request):
    form = SearchForm()
    query = None
    results = []
    if 'query' in request.GET:
        form = SearchForm(request.GET)
        if form.is_valid():
            query = form.cleaned_data['query']
            results = (
                Post.published.annotate(
                    similarity=TrigramSimilarity('title', query),
                )
                .filter(similarity__gt=0.1)
                .order_by('-similarity')
            )
    return render(request, 'blog/post/search.html', {'form': form, 'query': query, 'results': results})
```
- `SearchForm`（`forms.py` 新增）：只有一个 `query = forms.CharField()` 字段。
- 用的是 **`GET` 而不是 `POST`**：`if 'query' in request.GET`——搜索结果页应该是可收藏/可分享的 URL（如 `/blog/search/?query=django`），这是 GET 语义更贴切表单的经典场景，和邮件分享、评论提交那种"有副作用的动作"必须用 POST 形成对比。
- **`TrigramSimilarity('title', query)`**：这是 `django.contrib.postgres.search` 提供的功能，底层调用 PostgreSQL `pg_trgm` 扩展（也就是上面 `0005_trigram_ext` 迁移开启的能力），把标题和查询词都拆成三字符片段（trigram），计算重叠比例得到一个 0~1 的相似度分数。
- `.filter(similarity__gt=0.1)`：过滤掉相似度太低（基本不相关）的结果，`0.1` 是一个经验阈值。
- 这种模糊匹配比 `title__icontains=query` 更强大：即使用户拼错单词或者只匹配到部分片段，只要三元组重叠度够高也能被搜到，这是传统 `LIKE`/`icontains` 做不到的。

## 五、URL 路由：本章新增两条

```python
app_name = 'blog'
urlpatterns = [
    path('', views.post_list, name='post_list'),
    path('tag/<slug:tag_slug>/', views.post_list, name='post_list_by_tag'),  # 新增：标签筛选
    path('<int:year>/<int:month>/<int:day>/<slug:post>/', views.post_detail, name='post_detail'),
    path('<int:post_id>/share/', views.post_share, name='post_share'),
    path('<int:post_id>/comment/', views.post_comment, name='post_comment'),
    path('feed/', LatestPostsFeed(), name='post_feed'),   # 新增：RSS 订阅
    path('search/', views.post_search, name='post_search'),  # 新增：搜索
]
```
- `path('tag/<slug:tag_slug>/', views.post_list, ...)` 和根路径 `path('', views.post_list, ...)` **指向同一个视图函数**，只是有没有传 `tag_slug` 参数的区别——印证了上面提到的"一个函数服务两条路由"。
- `LatestPostsFeed()` 直接实例化后作为视图挂到路由上——Django Syndication 框架的 `Feed` 类本身实现了 `__call__`，可以直接当作可调用视图使用。

`mysite/urls.py` 也新增了 sitemap 路由：
```python
from blog.sitemaps import PostSitemap
sitemaps = {'posts': PostSitemap}
urlpatterns = [
    ...
    path('sitemap.xml', sitemap, {'sitemaps': sitemaps}, name='django.contrib.sitemaps.views.sitemap'),
]
```

## 六、RSS 订阅（`feeds.py`，全新模块）

```python
class LatestPostsFeed(Feed):
    title = 'My blog'
    link = reverse_lazy('blog:post_list')
    description = 'New posts of my blog.'

    def items(self):
        return Post.published.all()[:5]

    def item_title(self, item):
        return item.title

    def item_description(self, item):
        return truncatewords_html(markdown.markdown(item.body), 30)

    def item_pubdate(self, item):
        return item.publish
```
教学要点：
- **为什么用 `reverse_lazy` 而不是 `reverse`**：`Feed` 类的 `link` 是一个**类属性**，在模块被 import 的那一刻就会求值。如果用 `reverse('blog:post_list')`，此时 URLconf 可能还没有完全加载完成，会抛出 `NoReverseMatch`。`reverse_lazy` 返回一个惰性求值的代理对象，真正需要用到 URL 字符串时（比如渲染 RSS XML 时）才去解析，从而避开这个"循环依赖/加载顺序"问题。这是 Django 里"类属性 vs 方法调用时机"的一个经典坑点。
- `item_description` 里先用 `markdown.markdown(item.body)` 把 Markdown 正文转成 HTML，再用 `truncatewords_html` 截断到 30 个词——注意用的是 `_html` 版本而不是普通 `truncatewords`，因为普通版本会破坏 HTML 标签结构（比如在 `<strong>` 标签中间截断），`truncatewords_html` 能感知标签边界安全截断。

## 七、站点地图（`sitemaps.py`，全新模块）

```python
class PostSitemap(Sitemap):
    changefreq = 'weekly'
    priority = 0.9

    def items(self):
        return Post.published.all()

    def lastmod(self, obj):
        return obj.updated
```
- `changefreq`/`priority` 是给搜索引擎爬虫的提示信息（多久重新抓取一次、这批 URL 的相对重要程度），不是强制约束。
- `lastmod(self, obj)` 返回每个对象的最后修改时间——`Sitemap` 框架会自动对每个 `items()` 返回的对象调用 `obj.get_absolute_url()` 拼 URL，配合 `lastmod` 生成完整的 `<url><loc>...</loc><lastmod>...</lastmod></url>` 条目。这也是为什么 `Post` 模型必须实现 `get_absolute_url()`——sitemap 框架就是靠这个约定方法自动拿到每个对象的地址，不需要你手写。

## 八、自定义模板标签（`templatetags/blog_tags.py`，全新模块）

```python
register = template.Library()

@register.simple_tag
def total_posts():
    return Post.published.count()

@register.inclusion_tag('blog/post/latest_posts.html')
def show_latest_posts(count=5):
    latest_posts = Post.published.order_by('-publish')[:count]
    return {'latest_posts': latest_posts}

@register.simple_tag
def get_most_commented_posts(count=5):
    return Post.published.annotate(
        total_comments=Count('comments')
    ).order_by('-total_comments')[:count]

@register.filter(name='markdown')
def markdown_format(text):
    return mark_safe(markdown.markdown(text))
```
四种不同的模板标签/过滤器写法，一次性教全：
- **`simple_tag`**（`total_posts`）：最简单的形式，返回值直接输出到模板，比如 `{% total_posts %}` 输出数字。
- **`inclusion_tag`**（`show_latest_posts`）：不仅计算数据，还指定一个子模板（`latest_posts.html`）来渲染这份数据，返回的字典就是那个子模板的上下文——本质是"局部视图 + 局部模板"的复用单元，`{% show_latest_posts 3 %}` 直接嵌入渲染好的 `<ul>...</ul>`。
- **`simple_tag` + `as` 语法**：`get_most_commented_posts` 用 `{% get_most_commented_posts as most_commented_posts %}` 把结果存进模板变量，而不是直接输出——因为调用方（`base.html`）想自己控制怎么遍历渲染这批文章，而不是用固定的子模板。这里的 `annotate(total_comments=Count('comments'))` 又是一次"用聚合查询实现热门排序"的手法，和相似文章推荐算法同源。
- **自定义过滤器 `markdown`**（`@register.filter(name='markdown')`）：`markdown.markdown(text)` 把 Markdown 源码转成 HTML 字符串，但 Django 模板默认会对字符串做自动转义（防 XSS），所以必须用 `mark_safe()` 显式告诉 Django"这段 HTML 是可信的，不要转义"。这是模板层里少数需要主动关闭自动转义的合法场景——**前提是正文内容来自可信的博客作者（通过 admin 后台写入），而不是未经处理的用户输入**，否则会有 XSS 风险。这也是为什么只在渲染 `post.body` 这种作者可信内容时才用它，评论的 `comment.body` 依然用普通的 `linebreaks`（不转 HTML，更安全）。

## 九、模板层：把新能力接入页面

- **`base.html`**（侧边栏彻底变成动态内容）：
  ```html
  {% load blog_tags %}
  ...
  I've written {% total_posts %} posts so far.
  <a href="{% url "blog:post_feed" %}">Subscribe to my RSS feed</a>
  <h3>Latest posts</h3>
  {% show_latest_posts 3 %}
  <h3>Most commented posts</h3>
  {% get_most_commented_posts as most_commented_posts %}
  <ul>
    {% for post in most_commented_posts %}
      <li><a href="{{ post.get_absolute_url }}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
  ```
  对比 Chapter01/02 里写死的"This is my blog."静态文案，这里侧边栏现在每次请求都要额外跑 3 条查询（总数、最新 3 篇、评论最多的 5 篇）——是一个"内容丰富但有性能代价"的设计取舍，真实生产环境这类侧边栏数据通常会加缓存。
- **`list.html`**：
  - 顶部加了 `{% if tag %}<h2>Posts tagged with "{{ tag.name }}"</h2>{% endif %}`。
  - 每篇文章下方列出标签，并且每个标签本身就是链接：`{% url "blog:post_list_by_tag" tag.slug %}`——点标签能直接跳到该标签的筛选列表，形成"标签云"式导航。
  - 正文预览从 `{{ post.body|truncatewords:30|linebreaks }}` 换成了 `{{ post.body|markdown|truncatewords_html:30 }}`——先转 Markdown 为 HTML，再安全截断（`truncatewords_html` 而不是普通 `truncatewords`，原因同 feeds.py 里提到的）。
- **`detail.html`**：正文改用 `{{ post.body|markdown }}`（不截断，全文渲染）；新增"Similar posts"区块遍历 `similar_posts`，`{% empty %}` 处理没有相似文章的情况。
- **`search.html`**（全新模板）：GET 表单，`{% if query %}` 区分"刚打开搜索页"和"已经提交过查询"两种状态；结果里同样用 `markdown|truncatewords_html:12` 渲染摘要；带一个"Search again"链接方便清空重搜。
- **`latest_posts.html`**（全新的 inclusion_tag 专用子模板）：极简的 `<ul>` 列表，只负责渲染 `show_latest_posts` 传进来的 `latest_posts`。

## 十、端到端业务流程走查

1. **按标签浏览**：点击某篇文章下的标签链接 → `GET /blog/tag/django/` → `post_list(tag_slug='django')` → `get_object_or_404(Tag, slug='django')` 查到标签对象 → `post_list.filter(tags__in=[tag])` 过滤 → 分页渲染，标题上方额外显示"Posts tagged with 'django'"。
2. **查看详情触发相似推荐**：`GET /blog/2024/4/4/xxx/` → 常规详情逻辑之外，额外计算 `similar_posts`（按共同标签数+时间排序取 4 篇）→ 模板渲染"Similar posts"区块，读者可以继续点进相关内容，提升站内停留时长。
3. **全文搜索**：`GET /blog/search/?query=djago`（哪怕拼错一个字母）→ `TrigramSimilarity` 依然可能因为片段重叠度够高而匹配到"Django"相关标题 → 按相似度降序展示结果。
4. **订阅 RSS**：`GET /blog/feed/` → `LatestPostsFeed` 返回最新 5 篇文章的 XML Feed，RSS 阅读器可以定期拉取这个地址获取更新，`item_description` 里的正文经过 Markdown 渲染+截断。
5. **搜索引擎抓取**：爬虫请求 `GET /sitemap.xml` → Django 站点地图框架自动遍历 `PostSitemap.items()`（所有已发布文章）→ 用每篇文章的 `get_absolute_url()` + `updated` 时间生成标准 sitemap XML，帮助搜索引擎更高效地发现和重新抓取内容。

## 十一、与前面章节的完整对照

| 维度 | Chapter01 | Chapter02 | Chapter03（本章） |
|---|---|---|---|
| 数据库 | SQLite | SQLite | **PostgreSQL**（为 pg_trgm 搜索而切换） |
| 列表视图实现 | FBV，无分页 | CBV（`ListView`）+ 分页 | **改回 FBV**（因为要支持可选的标签过滤参数） |
| 标签系统 | 无 | 无 | **新增**（`django-taggit`，标签云导航 + 相似推荐） |
| 正文渲染 | `linebreaks` | `linebreaks` | **Markdown**（`markdown` 过滤器 + `mark_safe`） |
| 搜索 | 无 | 无 | **新增**（`TrigramSimilarity` 模糊搜索） |
| RSS 订阅 | 无 | 无 | **新增**（`django.contrib.syndication`） |
| 站点地图 | 无 | 无 | **新增**（`django.contrib.sitemaps`） |
| 自定义模板标签 | 无 | 无 | **新增**（4 种写法：simple_tag / inclusion_tag / as 语法 / filter） |
| 侧边栏内容 | 静态文案 | 静态文案 | **动态**（总数/最新文章/热门评论，3 条额外查询） |

**一句话总结**：Chapter03 把博客从"能发布、能互动"升级为"能被发现"——标签让站内导航更立体，搜索和相似推荐让内容互相引流，RSS 和 sitemap 则打通了站外的订阅与搜索引擎收录渠道。这也是全书 Blog 项目（Chapter01→03）的最终形态，后续 Chapter04 开始会转向一个全新项目（Social/Bookmarks 书签站）。
