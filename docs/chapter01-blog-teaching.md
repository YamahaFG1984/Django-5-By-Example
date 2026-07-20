# Chapter01 Blog 项目详解教学

> 对应目录：`Chapter01/`（Django 5 by Example 全书第一个可运行项目）

## 一、这一章的定位

Chapter01 是全书第一个可运行的最小 Django 项目：一个只有"文章列表 + 文章详情"的博客雏形。目的是让读者从零跑通 `startproject` → `startapp` → 模型 → 迁移 → 管理后台 → 视图 → URL → 模板 的完整链路。

**注意一个和纸质书不完全一致的地方**：这份代码仓库里的 Chapter01 比书中原始文字描述的内容更丰富——多了一个"收藏文章"（Favourite）功能，并且用到了 **`models.CompositePrimaryKey`**，这是 Django **5.2** 才引入的新特性（复合主键）。这说明仓库作者针对 Django 5.2 重新改写/扩充了示例，用来提前演示新特性，而不是书本原文的逐字复现。

## 二、目录结构

```
Chapter01/
├── Dockerfile / docker-compose.yml / do.sh   # 开发环境
├── requirements.txt                           # Django~=5.2, asgiref, sqlparse
└── mysite/
    ├── manage.py
    ├── mysite/                # 项目配置包
    │   ├── settings.py / urls.py / wsgi.py / asgi.py
    └── blog/                  # 唯一的 app
        ├── models.py / admin.py / views.py / urls.py / apps.py
        ├── migrations/0001_initial.py, 0002_favouritepost.py
        ├── static/css/blog.css
        └── templates/blog/{base.html, post/{list,detail,favourites}.html}
```

和 Chapter03（最终形态）相比，这里**没有**：Comment 模型、taggit 标签、PostgreSQL、表单、分页、搜索、RSS、sitemap——这些都是后续章节逐步加上去的。这一章就是最纯粹的骨架。

## 三、项目配置层

**`settings.py`**：教科书式的默认配置。
- 用的是 **SQLite**（`BASE_DIR / 'db.sqlite3'`），不像 Chapter03 已切到 PostgreSQL——这正是"循序渐进"的痕迹，数据库升级是后面章节的内容。
- `INSTALLED_APPS` 只多了一行 `'blog.apps.BlogConfig'`，没有 `django.contrib.sites`、`postgres`、`taggit` 等，都是后面才加的。
- `DEBUG=True`、`SECRET_KEY` 是明文占位符——典型的开发环境配置，生产不能这么用。

**`mysite/urls.py`**：只有两条路由：
```python
path('admin/', admin.site.urls),
path('blog/', include('blog.urls', namespace='blog')),
```
根路径 `/` 直接 404，所有博客内容都挂在 `/blog/` 下。

## 四、数据模型层（本章核心）

```python
class PublishedManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(status=Post.Status.PUBLISHED)

class Post(models.Model):
    class Status(models.TextChoices):
        DRAFT = 'DF', 'Draft'
        PUBLISHED = 'PB', 'Published'
    title = models.CharField(max_length=250)
    slug = models.SlugField(max_length=250)          # 注意：这里还没加 unique_for_date
    author = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='blog_posts')
    body = models.TextField()
    publish = models.DateTimeField(default=timezone.now)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)
    status = models.CharField(max_length=2, choices=Status, default=Status.DRAFT)

    objects = models.Manager()       # 默认管理器
    published = PublishedManager()   # 自定义管理器，只返回已发布文章

    class Meta:
        ordering = ['-publish']
        indexes = [models.Index(fields=['-publish'])]

    def get_absolute_url(self):
        return reverse('blog:post_detail', args=[self.id])
```

教学要点：
1. **双管理器模式**：`objects`（全部数据，给 admin 用）+ `published`（业务代码只看已发布内容）。这是 Django 里"数据全集 vs 业务视图"的经典范式，Chapter03 原封不动沿用了这个设计。
2. **`TextChoices`**：Django 3.0+ 的枚举写法，比老式的 tuple-of-tuples 更清晰，`Post.Status.PUBLISHED` 既是值也带 label。
3. **`slug` 此时还没加 `unique_for_date='publish'`**——这是故意的：Chapter01 的 URL 用 `id` 做主键（`<int:id>/`），到后面章节才改成"按日期+slug"的 SEO 友好 URL，slug 唯一性约束是那时候才引入的。
4. **`get_absolute_url()`**：Django 约定俗成的方法，供模板/admin 跳转到对象详情页用。

**`FavouritePost` 模型（本章的亮点/新知识点）**：
```python
class FavouritePost(models.Model):
    pk = models.CompositePrimaryKey("user", "post")
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    post = models.ForeignKey('blog.Post', on_delete=models.CASCADE)
    created = models.DateTimeField(auto_now_add=True)
```
- `CompositePrimaryKey("user", "post")` 是 **Django 5.2 新增的 API**，用 `(user_id, post_id)` 联合作为主键，而不是单独再造一个自增 `id` 字段。效果等价于旧版 Django 里常见的写法：
  ```python
  # Django 5.2 之前的写法
  class Meta:
      unique_together = ('user', 'post')
  ```
  但 `CompositePrimaryKey` 是数据库层真正的复合主键（而不是唯一约束+隐藏自增 id），语义更准确，也是这本书选用 Django 5.2 的原因之一——顺带教这个新特性。
- 这张表本质上是"用户收藏文章"的多对多中间表，天然防止同一用户重复收藏同一篇文章（复合主键唯一）。

## 五、迁移演进

- `0001_initial.py`：建 `Post` 表 + `-publish` 索引，依赖 `auth.user`。
- `0002_favouritepost.py`（`Django 5.2b1` 生成，日期 2025-03-10）：建 `FavouritePost`，用到 `models.CompositePrimaryKey(...)`，并对 `settings.AUTH_USER_MODEL` 做了 `swappable_dependency`（标准写法，允许项目自定义 User 模型而不破坏迁移依赖关系）。

## 六、Admin 后台

```python
@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ['title', 'slug', 'author', 'publish', 'status']
    list_filter = ['status', 'created', 'publish', 'author']
    search_fields = ['title', 'body']
    prepopulated_fields = {'slug': ('title',)}
    raw_id_fields = ['author']
    date_hierarchy = 'publish'
    ordering = ['status', 'publish']
    show_facets = admin.ShowFacets.ALWAYS
```
和 Chapter03 的 admin 配置**完全一致**，说明这块内容在第一章就定型了，后面章节没再改。值得讲的两点：
- `prepopulated_fields = {'slug': ('title',)}`：在 admin 表单里输入标题时自动生成 slug（JS 联动），减少编辑负担。
- `show_facets = admin.ShowFacets.ALWAYS`：**Django 5.0+ 新特性**，在 list_filter 侧边栏里显示每个筛选项匹配的记录数量（facet counts），默认是根据结果量自动判断是否显示，这里强制常显。

注意 `FavouritePost` **没有注册到 admin**——它是纯业务表，不需要人工管理。

## 七、视图层（4 个函数视图）

```python
def post_list(request):
    posts = Post.published.all()
    return render(request, 'blog/post/list.html', {'posts': posts})
```
最简单的列表视图，此时还**没有分页**（`Paginator` 是后面章节加的）。

```python
def post_detail(request, id):
    favourite_post = None
    post = get_object_or_404(Post, id=id, status=Post.Status.PUBLISHED)
    try:
        if request.user.is_authenticated:
            favourite_post = FavouritePost.objects.get(user=request.user, post=post)
    except FavouritePost.DoesNotExist:
        pass
    return render(request, 'blog/post/detail.html',
                  {'is_favourite': favourite_post is not None, 'post': post})
```
- `get_object_or_404(Post, id=id, status=Post.Status.PUBLISHED)`：直接在查询里带上状态过滤，草稿文章即使知道 id 也访问不到，返回 404——这是"以查询条件做权限控制"的常见手法。
- 用 `try/except FavouritePost.DoesNotExist` 而不是 `filter().first()`，是显式教你 Django ORM 的 `DoesNotExist` 异常机制；只有登录用户才查询是否收藏过。

```python
def add_favourite(request, id):
    post = get_object_or_404(Post, id=id)
    FavouritePost.objects.get_or_create(user=request.user, post=post)
    return HttpResponseRedirect(post.get_absolute_url())
```
- `get_or_create`：利用复合主键的唯一性，重复点"收藏"也不会报错或产生重复记录，天然幂等。
- 注意这个视图**没有加 `@login_required`**——如果匿名用户点击收藏，`request.user` 是 `AnonymousUser`，`get_or_create` 会因为外键约束失败而报错。这是一个值得留意的小漏洞/待改进点（模板里其实是在 `post_detail` 页面无条件渲染"收藏"链接，并没有先判断是否登录）。

```python
@login_required
def favourites(request):
    favourite_posts = Post.objects.filter(
        id__in=FavouritePost.objects.filter(user=request.user).values_list('post_id', flat=True)
    )
    return render(request, 'blog/post/favourites.html', {'favourite_posts': favourite_posts})
```
- 这里才正确加了 `@login_required`。
- 查询手法：先在 `FavouritePost` 里筛出当前用户收藏的 `post_id` 列表（`values_list('post_id', flat=True)`），再用 `id__in=...` 反查 `Post`。这是子查询模式的教学写法；更地道的写法其实可以用 `Post.objects.filter(favouritepost__user=request.user)` 走反向关联一步到位，但这里选择了更直白、更适合初学者理解的两步查询。

## 八、URL 路由

```python
app_name = 'blog'
urlpatterns = [
    path('', views.post_list, name='post_list'),
    path('<int:id>/', views.post_detail, name='post_detail'),
    path('favrourite/add/<int:id>/', views.add_favourite, name='add_favourite'),   # 注意：favrourite 拼写错误
    path('favourites/', views.favourites, name='favourites'),
]
```
`favrourite` 是个拼写错误（应为 favourite），不影响功能，只是命名不够严谨——这类小瑕疵在教学代码里很常见，实际项目中应该修正。

## 九、模板层

- `base.html`：定义 `{% block title %}` / `{% block content %}`，右侧写死了一个 `#sidebar`（"This is my blog."），到 Chapter03 这里才会被替换成动态的"最新文章/热门标签"侧栏。
- `list.html`：遍历 `posts`，标题可点击进入详情，正文用 `{{ post.body|truncatewords:30|linebreaks }}` 截断预览。**没有分页**，也**没有 markdown 过滤器**（这两个都是 Chapter02/03 引入的）。
- `detail.html`：根据 `is_favourite` 显示 ❤️ 或"Add to favourites"链接，正文全文 `linebreaks`。
- `favourites.html`：结构和 `list.html` 几乎一样，只是遍历 `favourite_posts`。

## 十、端到端业务流程走查

1. **匿名用户浏览列表**：`GET /blog/` → `post_list` → `Post.published.all()`（只含已发布文章，按 `-publish` 排序）→ 渲染 `list.html`。
2. **点进详情**：`GET /blog/3/` → `post_detail` → 404 检查（草稿/不存在都 404）→ 因为未登录，`favourite_post=None` → 模板显示"Add to favourites"链接。
3. **登录用户点击收藏**：`GET /blog/favrourite/add/3/` → `add_favourite` → `get_or_create(user, post)` 写入 `FavouritePost`（复合主键 `(user_id, post_id)`）→ 重定向回详情页 → 这次 `favourite_post` 查得到 → 显示 ❤️。
4. **查看收藏夹**：`GET /blog/favourites/`（未登录会被 `@login_required` 重定向到登录页，但项目里其实**没有配置** `LOGIN_URL`/登录视图，这也是留给后续章节的坑）→ 子查询拿到收藏的 `post_id` 列表 → 渲染 `favourites.html`。

## 十一、开发环境

`Dockerfile` 基于 `python:3.12.3-slim`，只装 `requirements.txt`（此时还不需要 `psycopg`，因为用的是 SQLite）。`docker-compose.yml` 三个服务：`web`（基础镜像）、`web_migrate`（跑一次性迁移）、`web_run`（起 `runserver`，映射 8000 端口，`depends_on: web_migrate` 保证先迁移再起服务）。`do.sh` 提供 `./do.sh start/stop/migrate/shell` 等便捷命令。

## 十二、这一章埋下的伏笔（对照后续章节）

| 本章现状 | 后续章节的演进 |
|---|---|
| SQLite | Chapter03 切到 PostgreSQL |
| `slug` 无唯一约束，URL 用 `id` | Chapter02/03 加 `unique_for_date` + 日期型 URL |
| 列表无分页 | Chapter02 引入 `Paginator` |
| 正文用 `linebreaks` | Chapter03 引入 Markdown 渲染 + 自定义模板标签 |
| 无标签系统 | Chapter03 引入 `django-taggit` |
| 无评论 | Chapter03 引入 `Comment` 模型 |
| Favourite 功能（本章特有） | 后续章节里这个功能**没有再出现**——它更像是本仓库作者单独插入用来演示 `CompositePrimaryKey` 新特性的教学补丁，Chapter02/03 的 `models.py` 里已经找不到 `FavouritePost` 了 |
