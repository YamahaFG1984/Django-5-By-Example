# Chapter02 Blog 项目详解教学

> 对应目录：`Chapter02/`（Django 5 by Example 全书第二个可运行项目，在 Chapter01 骨架上迭代）

## 一、这一章的定位

如果说 Chapter01 是"能跑起来的最小模型"，Chapter02 就是把它变成一个"看起来像真博客"的产品：文章有了漂亮的 URL、列表能翻页、读者能通过邮件分享文章、能在文章下面留言。同时代码里第一次出现了 **邮件发送**、**表单（Form / ModelForm）**、**Class-Based View**。

值得注意的是：Chapter01 里那个用来演示 `CompositePrimaryKey` 的 `FavouritePost`/收藏功能，在这一章**彻底消失**了——`models.py`、`admin.py`、`urls.py`、模板里都找不到它的痕迹。这印证了收藏功能只是仓库作者为了展示 Django 5.2 新特性而插入的"教学彩蛋"，并不属于全书主线，从 Chapter02 起主线重新回到书本原著内容。

## 二、数据模型层：两处关键升级

### 1. `slug` 加上了 `unique_for_date='publish'`

```python
slug = models.SlugField(max_length=250, unique_for_date='publish')
```
对应迁移 `0002_alter_post_slug.py`。这个约束的含义是："同一天发布的文章，slug 不能重复"（不是全局唯一，而是按天唯一）。这是为下面的日期型 URL 做铺垫——URL 用 `年/月/日/slug` 定位文章，只要保证同一天内 slug 不重复，这个 URL 就能唯一确定一篇文章。

### 2. `get_absolute_url()` 从按 id 跳转改成按日期+slug 跳转

```python
def get_absolute_url(self):
    return reverse('blog:post_detail', args=[
        self.publish.year, self.publish.month, self.publish.day, self.slug,
    ])
```
对比 Chapter01 的 `args=[self.id]`——这是本章最核心的语义变化：**从"数据库内部 id 驱动的 URL"升级为"人类可读、对 SEO 友好的 URL"**（例如 `/blog/2024/4/4/my-first-post/`）。搜索引擎和用户都更喜欢这种带日期和关键词的地址。

### 3. 新增 `Comment` 模型

```python
class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')
    name = models.CharField(max_length=80)
    email = models.EmailField()
    body = models.TextField()
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)
    active = models.BooleanField(default=True)

    class Meta:
        ordering = ['created']
        indexes = [models.Index(fields=['created'])]
```
- `related_name='comments'`：使得可以从文章反查评论，即 `post.comments.all()`（后面视图里用到）。
- `active` 字段：给管理员一个"软删除/审核"开关——不需要真的删除垃圾评论，把 `active` 设为 `False` 即可让它不再显示，这是内容审核的常见设计。
- `ordering = ['created']`：按时间正序，最早的评论排最前面（和 `Post` 的 `-publish` 倒序刚好相反，符合评论区"按时间线阅读"的直觉）。

对应迁移 `0003_comment_...py`：建表 + `created` 索引。

## 三、表单层（本章新引入的模块 `forms.py`）

```python
class EmailPostForm(forms.Form):
    name = forms.CharField(max_length=25)
    email = forms.EmailField()
    to = forms.EmailField()
    comments = forms.CharField(required=False, widget=forms.Textarea)

class CommentForm(forms.ModelForm):
    class Meta:
        model = Comment
        fields = ['name', 'email', 'body']
```
教学要点：
- **`forms.Form` vs `forms.ModelForm`** 的第一次对比出现：`EmailPostForm` 不对应任何数据库模型，纯粹用来做"发件人信息 + 收件邮箱 + 留言"的输入校验；`CommentForm` 则直接从 `Comment` 模型反推出表单字段（`fields = [...]` 白名单指定允许用户填写哪些字段，`post` 外键、`created`、`active` 都不暴露给用户，由后端逻辑控制）。
- `forms.EmailField()` 自带邮箱格式校验，不用手写正则。

## 四、视图层：本章信息量最大的部分

### 1. 分页版 `post_list`（函数视图，写好后被注释掉了）

```python
def post_list(request):
    post_list = Post.published.all()
    paginator = Paginator(post_list, 3)
    page_number = request.GET.get('page', 1)
    try:
        posts = paginator.page(page_number)
    except PageNotAnInteger:
        posts = paginator.page(1)
    except EmptyPage:
        posts = paginator.page(paginator.num_pages)
    return render(request, 'blog/post/list.html', {'posts': posts})
```
标准的三层异常兜底分页模式：
- `PageNotAnInteger`：URL 里 `?page=abc` 这种非数字输入，兜底回第 1 页。
- `EmptyPage`：`?page=999` 超出范围，兜底到最后一页。
- 每页 3 篇文章（`Paginator(post_list, 3)`）。

### 2. `PostListView`（Class-Based View，实际被启用的版本）

```python
class PostListView(ListView):
    queryset = Post.published.all()
    context_object_name = 'posts'
    paginate_by = 3
    template_name = 'blog/post/list.html'
```
在 `urls.py` 里可以看到：
```python
# path('', views.post_list, name='post_list'),   # 被注释掉
path('', views.PostListView.as_view(), name='post_list'),  # 实际生效
```
**这是书中很经典的教学对比手法**：先手写一遍分页逻辑让你理解原理，再展示 Django 自带的 `ListView` 如何用几行声明式代码替代整个 try/except 分页逻辑（`ListView` 内部自动处理了 `Paginator` 和两种异常）。上线用的是 CBV 版本，FBV 版本留着当参考注释。

有个细节要注意：`ListView` 传给模板的分页对象变量名是 **`page_obj`**（Django 内置约定），而手写的 FBV 版本传的是 **`posts`**。所以 `list.html` 里必须写 `{% include "pagination.html" with page=page_obj %}`，如果之后改回 FBV 版本，这行要跟着改成 `page=posts`——这是 CBV/FBV 切换时容易踩的坑。

### 3. `post_detail`：从按 id 查询改为按日期+slug 联合查询

```python
def post_detail(request, year, month, day, post):
    post = get_object_or_404(
        Post, status=Post.Status.PUBLISHED, slug=post,
        publish__year=year, publish__month=month, publish__day=day,
    )
    comments = post.comments.filter(active=True)
    form = CommentForm()
    return render(request, 'blog/post/detail.html',
                  {'post': post, 'comments': comments, 'form': form})
```
- 用 `publish__year=year` 这种字段查找（field lookup）语法按年月日过滤，配合 slug 唯一定位一篇文章。
- `post.comments.filter(active=True)`：只把审核通过的评论传给模板，被标记为不活跃的评论不会展示（但不会被删除，管理员随时可以在 admin 里重新激活）。
- 顺带把一个空的 `CommentForm()` 传给模板，用于渲染"发表评论"表单（GET 请求只是展示表单，真正提交由另一个视图 `post_comment` 处理——这是"展示"和"处理提交"分离成两个视图的模式）。

### 4. `post_share`：邮件分享功能（本章的第二个亮点）

```python
def post_share(request, post_id):
    post = get_object_or_404(Post, id=post_id, status=Post.Status.PUBLISHED)
    sent = False
    if request.method == 'POST':
        form = EmailPostForm(request.POST)
        if form.is_valid():
            cd = form.cleaned_data
            post_url = request.build_absolute_uri(post.get_absolute_url())
            subject = f"{cd['name']} ({cd['email']}) recommends you read {post.title}"
            message = f"Read {post.title} at {post_url}\n\n{cd['name']}'s comments: {cd['comments']}"
            send_mail(subject=subject, message=message, from_email=None, recipient_list=[cd['to']])
            sent = True
    else:
        form = EmailPostForm()
    return render(request, 'blog/post/share.html', {'post': post, 'form': form, 'sent': sent})
```
教学要点：
- **同一个视图函数处理 GET 和 POST 两种请求**：GET 时展示空表单，POST 时校验+发信。这是 Django 表单处理的标准范式（"提交到自身"模式）。
- `request.build_absolute_uri(post.get_absolute_url())`：`get_absolute_url()` 只返回路径（如 `/blog/2024/4/4/xxx/`），`build_absolute_uri` 把它拼成带协议和域名的完整 URL（如 `http://127.0.0.1:8000/blog/2024/4/4/xxx/`），因为邮件里的链接必须是绝对地址。
- `send_mail(from_email=None, ...)`：`from_email` 传 `None` 时 Django 会自动使用 `settings.DEFAULT_FROM_EMAIL`。
- `sent` 标志位控制模板渲染"成功提示"还是"表单"——对应 `share.html` 里的 `{% if sent %}`。
- 对应地，`settings.py` 里第一次出现了邮件配置（Gmail SMTP + `python-decouple` 从环境变量读取账号密码），`docker-compose.yml` 也相应加了 `env_file: - .env`。这说明 Chapter03 里的邮件配置其实从 Chapter02 就开始了，Chapter03 只是继续沿用。

### 5. `post_comment`：评论提交（`commit=False` 模式首次出现）

```python
@require_POST
def post_comment(request, post_id):
    post = get_object_or_404(Post, id=post_id, status=Post.Status.PUBLISHED)
    comment = None
    form = CommentForm(data=request.POST)
    if form.is_valid():
        comment = form.save(commit=False)
        comment.post = post
        comment.save()
    return render(request, 'blog/post/comment.html', {'post': post, 'form': form, 'comment': comment})
```
- `@require_POST`：装饰器强制该视图只接受 POST 请求，GET 请求会被拒绝返回 405——因为提交评论这个动作不应该能被 GET 触发（防止意外的重复提交/CSRF 类问题，也符合 REST 语义）。
- **`form.save(commit=False)`**：这是 `ModelForm` 的关键技巧。`CommentForm` 表单本身不包含 `post` 外键字段（`Meta.fields = ['name', 'email', 'body']`，没有 `post`），所以不能直接 `form.save()`——数据库要求 `post_id` 非空。`commit=False` 让 Django 只在内存里构造出 `Comment` 实例但不写库，这样就有机会手动把 URL 里传来的 `post` 对象赋值给 `comment.post`，再调用 `comment.save()` 真正落库。这是"表单字段不完整、需要后端补全外键"场景的标准解法，Chapter03 完全沿用了这个模式。

## 五、URL 路由

```python
app_name = 'blog'
urlpatterns = [
    path('', views.PostListView.as_view(), name='post_list'),
    path('<int:year>/<int:month>/<int:day>/<slug:post>/', views.post_detail, name='post_detail'),
    path('<int:post_id>/share/', views.post_share, name='post_share'),
    path('<int:post_id>/comment/', views.post_comment, name='post_comment'),
]
```
对比 Chapter01 的 `path('<int:id>/', ...)`，详情页路由变成了四段式路径参数 `<int:year>/<int:month>/<int:day>/<slug:post>/`，`<slug:post>` 用 Django 自带的 slug 转换器约束 URL 段的字符集（字母数字和连字符）。

## 六、Admin 后台

`PostAdmin` 配置和 Chapter01 完全一致；新增了：
```python
@admin.register(Comment)
class CommentAdmin(admin.ModelAdmin):
    list_display = ['name', 'email', 'post', 'created', 'active']
    list_filter = ['active', 'created', 'updated']
    search_fields = ['name', 'email', 'body']
```
`list_filter` 里带 `active`，方便管理员快速筛出待审核/已屏蔽的评论。

## 七、模板层

- **`list.html`**：链接从 `{% url 'blog:post_detail' post.id %}` 改成了 `{{ post.get_absolute_url }}`（因为详情页现在需要四个参数，直接调用模型方法比在模板里手写四个 `{% url %}` 参数更简洁）；底部加了 `{% include "pagination.html" with page=page_obj %}`。
- **`pagination.html`**（新增的可复用组件）：
  ```html
  {% if page.has_previous %}
    <a href="{% querystring page=page.previous_page_number %}">Previous</a>
  {% endif %}
  ...
  {% if page.has_next %}
    <a href="{% querystring page=page.next_page_number %}">Next</a>
  {% endif %}
  ```
  这里用到了 **`{% querystring %}`**，这是 **Django 5.1 新增的模板标签**，作用是在保留当前 URL 其它查询参数的前提下，只修改/添加指定的参数（这里是 `page`）。比起旧版本手写 `?page={{ n }}` 更健壮——如果 URL 上还带着搜索关键词等其它参数，`querystring` 标签会自动保留它们不丢失。
- **`detail.html`**：新增"Share this post"链接、评论数统计（`{% with comments.count as total_comments %}` + `pluralize` 过滤器自动处理单复数）、遍历渲染评论列表（`{% empty %}` 处理无评论情况）、`{% include "blog/post/includes/comment_form.html" %}` 嵌入评论表单。
- **`comment_form.html`**（新增的 include 片段）：
  ```html
  <form action="{% url "blog:post_comment" post.id %}" method="post">
    {{ form.name.as_field_group }}
    ...
  ```
  `as_field_group` 是 **Django 5.0+ 新增的表单渲染方法**，把单个字段连同它的 label、帮助文本、错误信息一起渲染成一组 HTML，比逐个手写 `{{ form.name.label_tag }}{{ form.name }}{{ form.name.errors }}` 更简洁，也比整体的 `{{ form.as_p }}` 更灵活（可以自由控制每个字段的排版，比如这里 `name`、`email` 用 `class="left"` 并排显示）。
- **`share.html`**：GET/POST 双态模板，`{% if sent %}` 控制显示"发送成功"还是表单本身，`{{ form.as_p }}` 走的是整体快速渲染。
- **`comment.html`**：同样是"成功 vs 重新显示表单"的双态模式，`{% if comment %}` 判断评论是否创建成功。

## 八、端到端业务流程走查

1. **浏览列表（带分页）**：`GET /blog/?page=2` → `PostListView` 自动处理分页 → `list.html` 渲染第 2 页的 3 篇文章 → 底部翻页链接用 `{% querystring %}` 生成。
2. **点进详情**：`GET /blog/2024/4/4/my-first-post/` → 按年月日+slug 精确匹配 → 展示正文、已审核评论列表、空评论表单。
3. **提交评论**：表单 `action` 指向 `POST /blog/5/comment/` → `@require_POST` 拦掉 GET → `form.save(commit=False)` 补齐 `post` 外键后落库 → 渲染 `comment.html` 显示"评论已添加"。
4. **邮件分享**：`GET /blog/5/share/` 显示空表单 → 用户填写好友邮箱和留言，`POST` 提交 → 校验通过后拼出绝对 URL，调用 `send_mail` 真实发送 Gmail 邮件 → 页面切换成"发送成功"提示。

## 九、与前后章节的对照

| 维度 | Chapter01 | Chapter02（本章） | Chapter03 |
|---|---|---|---|
| 详情页 URL | `<int:id>/` | `<year>/<month>/<day>/<slug>/` | 相同，沿用 |
| 列表分页 | 无 | `Paginator` / `ListView` 双版本 | 沿用 CBV，每页 3 篇 |
| 评论 | 无 | 新增 `Comment` 模型+提交流程 | 沿用 |
| 邮件分享 | 无 | 新增，Gmail SMTP | 沿用 |
| 数据库 | SQLite | 仍是 SQLite | 切换到 PostgreSQL |
| 标签/搜索/RSS/sitemap | 无 | 无 | 新增（taggit、TrigramSimilarity、syndication、sitemaps） |
| 收藏功能 CompositePrimaryKey | 有（本章特有教学彩蛋） | **消失** | 无 |

简单说：**Chapter02 把"文章能被看到"升级成"文章能被搜索引擎友好地索引、能翻页浏览、能被分享传播、能收到读者反馈"**，是从静态展示走向"社区互动"的关键一步；Chapter03 则在此基础上叠加标签分类、全文搜索、订阅（RSS）和站点地图，进一步强化内容发现能力。
