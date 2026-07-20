# Chapter07 教学笔记：Bookmarks 项目 —— 关注系统、动态流与 Redis 排行榜

第七章信息量很大——这一章把 Bookmarks 项目从"能收藏图片"升级成了一个真正的**社交网络**：关注系统、动态流、Redis 排行榜全部在这一章落地。而且这一章恰好印证了之前聊 TSD 时说的那个隐患——因为项目一直没上自定义 User 模型，这里为了给 `User` 加关注关系，不得不用一些"外挂"手法。

## 一、本章定位与新增依赖

```diff
+ django-debug-toolbar==4.3.0
+ redis==5.0.4
```

`docker-compose.yml` 新增了一个 `cache` 服务（Redis 7.2.4），`web` 服务加了 `env_file: .env`。三条主线：

1. **关注系统**（`account.Contact` + 动态给 `User` 加 `following`/`followers`）
2. **动态流/Activity Stream**（新 app `actions`，用 GenericForeignKey 做多态关联）
3. **Redis 浏览计数 + 排行榜**（`images` 里新增 `image_ranking` 视图）

顺带引入了 `django-debug-toolbar` 这个开发期利器。

## 二、关注系统：给别人的模型"打补丁"

```python
# account/models.py
class Contact(models.Model):
    user_from = models.ForeignKey(settings.AUTH_USER_MODEL, related_name='rel_from_set', on_delete=models.CASCADE)
    user_to = models.ForeignKey(settings.AUTH_USER_MODEL, related_name='rel_to_set', on_delete=models.CASCADE)
    created = models.DateTimeField(auto_now_add=True)
    class Meta:
        indexes = [models.Index(fields=['-created'])]
        ordering = ['-created']

# 动态给 User 类挂一个字段
user_model = get_user_model()
user_model.add_to_class(
    'following',
    models.ManyToManyField('self', through=Contact, related_name='followers', symmetrical=False),
)
```

这是本章**最值得拿出来细讲**的一段代码。想让"关注"关系挂在 `User` 上，最自然的做法是直接在 User 模型类里加一个 `following` 字段——但这个项目从 Chapter04 到现在**一直用的是 Django 默认的 `auth.User`**，你没法去改 Django 内部那个类的源码。于是作者用了 `add_to_class()`：在 `account/models.py` 模块被加载时，硬生生往 `User` 类上"挂"一个新的多对多字段。

**这正是之前聊《Two Scoops of Django》时说的那个警告应验了**——"项目起步就该用自定义 User 模型"，原因就是像这样后期想往 User 上加字段/关系时，只能靠 `add_to_class` 这种带 monkey-patch 味道的手法绕过去；如果一开始就是 `class User(AbstractUser)`，这里直接在类体里写一行 `following = models.ManyToManyField(...)` 就完事了，不需要这段模块级别的"动态注入"代码，可读性和 IDE 补全都会好很多。

几个字段设计细节：

- `symmetrical=False`——自关联的多对多默认是**对称**的（Django 假设"我的朋友"这种双向关系，A 关联 B 自动等于 B 关联 A），但"关注"是单向的（A 关注 B 不代表 B 关注 A），必须显式关掉对称性。
- `through=Contact`——因为需要在关系上附加额外信息（`created` 时间戳）并且要能按 `user_from`/`user_to` 精确查询/删除某一条关注记录，用自动生成的隐式 M2M 中间表做不到这些，所以必须显式声明 `through`。
- `related_name='followers'`——所以 `user.following.all()` 是"我关注的人"，`user.followers.all()` 是"关注我的人"，一套字段名同时覆盖了两个查询方向。

`account/migrations/0002_contact.py` 是标准的建表迁移，依赖 `account` 的 0001 迁移和可替换的 `AUTH_USER_MODEL`。

## 三、`user_follow` 视图

```python
@require_POST
@login_required
def user_follow(request):
    user_id = request.POST.get('id')
    action = request.POST.get('action')
    if user_id and action:
        try:
            user = User.objects.get(id=user_id)
            if action == 'follow':
                Contact.objects.get_or_create(user_from=request.user, user_to=user)
                create_action(request.user, 'is following', user)
            else:
                Contact.objects.filter(user_from=request.user, user_to=user).delete()
            return JsonResponse({'status': 'ok'})
        except User.DoesNotExist:
            return JsonResponse({'status': 'error'})
    return JsonResponse({'status': 'error'})
```

和 Chapter06 的 `image_like` 是一模一样的套路（AJAX + `get_or_create`/`filter().delete()` 切换状态），`user/detail.html` 里的关注按钮 JS 也和点赞按钮 JS 几乎是复制粘贴（乐观更新按钮文字和计数）——这里能看出这本书里**同一类交互模式在不同业务场景下反复复用**的写法习惯。

`user_list`/`user_detail` 两个视图都很简单：

```python
@login_required
def user_list(request):
    users = User.objects.filter(is_active=True)
    return render(request, 'account/user/list.html', {'section': 'people', 'users': users})

@login_required
def user_detail(request, username):
    user = get_object_or_404(User, username=username, is_active=True)
    return render(request, 'account/user/detail.html', {'section': 'people', 'user': user})
```

`user/detail.html` 里还顺手用 `{% include "images/image/list_images.html" with images=user.images_created.all %}` 复用了 Chapter06 图片列表的局部模板，展示这个用户收藏过的图片。

## 四、`ABSOLUTE_URL_OVERRIDES`——另一处"没有自定义 User 模型"的补丁

```python
# settings.py
ABSOLUTE_URL_OVERRIDES = {
    'auth.user': lambda u: reverse_lazy('user_detail', args=[u.username])
}
```

模板里到处用 `{{ user.get_absolute_url }}`（`actions/action/detail.html`、`user/list.html` 都在用），但 `auth.User` 类本身没有 `get_absolute_url()` 方法。`ABSOLUTE_URL_OVERRIDES` 是 Django 提供的一个全局配置字典，专门解决"我没法改这个模型的类定义，但又想让它支持 `get_absolute_url()`"的问题——用一个 lambda 全局注册，Django 在调用 `user.get_absolute_url()` 时会先查这个字典。这又是一处**如果一开始用自定义 User 模型，直接在类里写方法就够了**的例子，两处补丁放在一起看，TSD 那条建议的说服力就很直观了。

## 五、`actions` 应用：动态流

```python
# actions/models.py
class Action(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, related_name='actions', on_delete=models.CASCADE)
    verb = models.CharField(max_length=255)
    created = models.DateTimeField(auto_now_add=True)
    target_ct = models.ForeignKey(ContentType, blank=True, null=True, related_name='target_obj', on_delete=models.CASCADE)
    target_id = models.PositiveIntegerField(null=True, blank=True)
    target = GenericForeignKey('target_ct', 'target_id')
    class Meta:
        indexes = [models.Index(fields=['-created']), models.Index(fields=['target_ct', 'target_id'])]
        ordering = ['-created']
```

`target_ct` + `target_id` + `GenericForeignKey`——这是本书**第一次真正用上 contenttypes 框架做多态关联**。之前 Blog 项目的 `Comment.post` 是普通外键，只能指向 `Post`；这里的 `Action.target` 能指向**任意模型的任意一条记录**（一个 `Image`、一个 `User`、以后可能是别的什么），因为动态流本身就需要"用户对某个不确定类型的对象做了某个动作"这种通用表达。

### `create_action`——带防抖的写入工具

```python
# actions/utils.py
def create_action(user, verb, target=None):
    now = timezone.now()
    last_minute = now - datetime.timedelta(seconds=60)
    similar_actions = Action.objects.filter(user_id=user.id, verb=verb, created__gte=last_minute)
    if target:
        target_ct = ContentType.objects.get_for_model(target)
        similar_actions = similar_actions.filter(target_ct=target_ct, target_id=target.id)
    if not similar_actions:
        action = Action(user=user, verb=verb, target=target)
        action.save()
        return True
    return False
```

在真正插入之前，先查"过去 60 秒内，这个用户对同一个目标做过同样动作的记录是否已经存在"——如果存在就**不重复插入**。这是为了防止用户手抖连续点赞/取消点赞好几次，动态流里刷出一串一模一样的"张三 likes 某图片"。这是个很实用的"业务层去重"模式，比事后清理垃圾数据要轻量。

四个业务动作会调用它：注册（`'has created an account'`）、收藏图片（`'bookmarked image'`，target=图片）、点赞（`'likes'`，target=图片）、关注（`'is following'`，target=被关注的人）。

`actions/views.py` 是空文件——本章的动态流没有独立的视图/URL，只通过 `dashboard` 视图查询 + 局部模板 `include` 展示。`actions/admin.py` 注册了 `ActionAdmin(list_display=['user','verb','target','created'], list_filter=['created'], search_fields=['verb'])`，方便在后台直接查动态记录。

### `dashboard` 视图的动态流查询

```python
@login_required
def dashboard(request):
    actions = Action.objects.exclude(user=request.user)
    following_ids = request.user.following.values_list('id', flat=True)
    if following_ids:
        actions = actions.filter(user_id__in=following_ids)
    actions = actions.select_related('user', 'user__profile').prefetch_related('target')[:10]
    return render(request, 'account/dashboard.html', {'section': 'dashboard', 'actions': actions})
```

- `exclude(user=request.user)`——自己的动作不用在自己的首页里再看一遍。
- **冷启动兜底**：如果 `following_ids` 是空的（新用户还没关注任何人），就不加过滤条件，直接展示**全站**的动态——这样新用户的首页不会是一片空白，这是个很实际的产品细节。
- `.select_related('user', 'user__profile')`：跨两层外键（`Action.user` 和 `User.profile`）一次 JOIN 查完，避免列表里每条动态都单独查一次作者信息和头像。
- `.prefetch_related('target')`：因为 `target` 是 `GenericForeignKey`，指向的表不固定，普通 SQL JOIN 做不到，Django 会按 `ContentType` 分组批量查询，比对列表里每条动态单独查一次 target 已经好很多，但依然比不上普通外键的一次 JOIN 高效——这是 GenericForeignKey 这类"万能关联"必须付出的查询代价。

`actions/action/detail.html` 是个可复用局部模板，同时渲染"谁做的（头像）"和"对象是什么（如果 target 有 `image` 字段就显示缩略图）"，用 `{% if target.image %}` 做的是鸭子类型判断——完全不关心 target 具体是哪个模型，只要它凑巧有 `image` 属性就显示。

`register` 视图在创建 Profile 之后新增一行：

```python
create_action(new_user, 'has created an account')
```

## 六、Redis：浏览计数与排行榜

```python
r = redis.Redis(host=settings.REDIS_HOST, port=settings.REDIS_PORT, db=settings.REDIS_DB)

def image_detail(request, id, slug):
    image = get_object_or_404(Image, id=id, slug=slug)
    total_views = r.incr(f'image:{image.id}:views')
    r.zincrby('image_ranking', 1, image.id)
    ...

def image_ranking(request):
    image_ranking = r.zrange('image_ranking', 0, -1, desc=True)[:10]
    image_ranking_ids = [int(id) for id in image_ranking]
    most_viewed = list(Image.objects.filter(id__in=image_ranking_ids))
    most_viewed.sort(key=lambda x: image_ranking_ids.index(x.id))
    return render(request, 'images/image/ranking.html', {'section': 'images', 'most_viewed': most_viewed})
```

- `r.incr(...)`——Redis 字符串类型的原子自增，每次访问详情页 +1，返回值直接就是当前总浏览数，不用额外再查一次。
- `r.zincrby('image_ranking', 1, image.id)`——Redis **有序集合**（Sorted Set），成员是图片 ID，分数是浏览次数，每次访问对应图片分数 +1。
- `r.zrange(..., desc=True)[:10]`——直接从 Redis 取分数最高的前 10 个 ID，这一步排序是 Redis 内部结构原生支持的，不需要应用层再排。
- 但拿到 ID 列表后还要回数据库查详情，而 `filter(id__in=image_ranking_ids)` **不保证**返回顺序和传入的 ID 顺序一致——所以额外用 `.sort(key=lambda x: image_ranking_ids.index(x.id))` 在 Python 层按 Redis 给出的名次重新排一遍。这是"跨存储拼接数据"时一个容易被忽略的细节：Redis 负责排序，数据库负责补充详情字段，两者顺序要手动对齐。

`images/urls.py` 新增 `path('ranking/', views.image_ranking, name='ranking')`，`images/image/ranking.html` 是个简单的有序列表模板；`images/image/detail.html` 也新增了浏览次数的展示。

**为什么浏览计数用 Redis，点赞数却继续用数据库字段？** 看 `images/signals.py`：

```python
@receiver(m2m_changed, sender=Image.users_like.through)
def users_like_changed(sender, instance, **kwargs):
    instance.total_likes = instance.users_like.count()
    instance.save()
```

`total_likes` 是数据库字段（`images/models.py` 新增 `total_likes = models.PositiveIntegerField(default=0)` + `models.Index(fields=['-total_likes'])`，支持"按点赞数排序"的查询）——点赞频率远低于浏览频率，用数据库字段 + 信号维护足够；而**浏览量是纯只读高频写入**（几乎每次请求都要 +1），如果也存成数据库字段，频繁 `UPDATE` 会带来不小的行锁开销，Redis 的原子自增结构就是为这种场景设计的。这是"选对存储介质"的一个很好的对比案例。

`m2m_changed` 信号在 `AppConfig.ready()` 里导入注册：

```python
class ImagesConfig(AppConfig):
    def ready(self):
        import images.signals
```

这是 Django 官方推荐的信号注册方式——放在 `ready()` 里而不是 `models.py` 顶层直接 import，避免应用还没完全加载完就触发信号相关的循环导入问题。

**这里也正好是"该用 signal"的一个好例子**，可以和之前 TDD 里 `PostRevision` 改成显式调用的判断对照着看：`total_likes` 这个副作用只应该跟着 `users_like` 这个 M2M 关系本身的任何变化发生（不管是通过 `image_like` 视图改的，还是以后管理员在 admin 后台直接改的，甚至未来某个批量导入脚本改的），signal 能保证不管从哪个入口改了这层关系，`total_likes` 都自动同步；而 `PostRevision` 那种"只应该跟着一次特定的编辑操作发生"的副作用，才更适合显式调用。**判断标准是：副作用绑定的是"数据本身的变化"还是"某一次特定的业务动作"。**

`image_create` 和 `image_like` 视图也都各自补上了 `create_action(request.user, 'bookmarked image', new_image)` / `create_action(request.user, 'likes', image)` 调用，把这两个动作也接入了动态流。`images/migrations/0002_image_total_likes_and_more.py` 是标准的字段+索引新增迁移。

## 七、`django-debug-toolbar`

```python
INSTALLED_APPS = [..., 'debug_toolbar']
MIDDLEWARE = ['debug_toolbar.middleware.DebugToolbarMiddleware', ...]  # 必须放最前面
INTERNAL_IPS = ['127.0.0.1']
```

```python
# urls.py
path('__debug__/', include('debug_toolbar.urls')),
```

`DebugToolbarMiddleware` 放中间件列表最前面，是为了能包裹住整个请求-响应周期，统计每个阶段的耗时和 SQL 查询次数；`INTERNAL_IPS` 限制只有从这些 IP 发起的请求才会显示工具栏，防止意外在生产环境暴露内部信息。**但这只是一层"访问限制"，不是"生产安全"**——真正的生产环境应该整个 `debug_toolbar` 都不出现在 `INSTALLED_APPS`/`MIDDLEWARE` 里，这又呼应了之前在 TDD 文档里提过的"settings 该按环境拆分"（`base/dev/production`）——这类只该在开发环境存在的东西，本就不该靠一个 IP 白名单去防，而应该从配置源头就不加载。

## 八、`prompts/task.md`——又一份作者的练习草稿

和 Chapter03 那份一样，这不是书的正文实现，是作者留下的 AI 提示词草稿，内容是"我想用 Django 信号在 User 创建时自动建 Profile"。这份草稿其实是对当前 `register` 视图里手动 `Profile.objects.create(user=new_user)` 做法的一个"备选方案"——如果真的按草稿实现，能彻底堵住之前在 Chapter04/05 里反复提到的"忘记建 Profile"这个隐患，不管用户是走注册视图还是走社交登录 pipeline，signal 都能兜底。但目前代码库里这两条创建路径依然是分别手动调用，没有采用这个草稿里的方案。

## 九、端到端流程

**关注某人**：点击关注按钮 → AJAX POST 到 `user_follow` → `Contact.objects.get_or_create(...)` 建关系 + `create_action(..., 'is following', user)` 写动态 → 按钮乐观更新为"已关注"。

**打开首页**：`dashboard` 视图查询关注的人（或冷启动兜底显示全站）的最近 10 条动态 → `select_related`/`prefetch_related` 优化查询 → 逐条 include `actions/action/detail.html` 渲染。

**查看一张图**：Redis 浏览计数 +1、写入排行榜 zset；点赞 → M2M 变化触发 `total_likes` 信号同步 + 写动态记录。

**查看排行榜**：`image_ranking` 视图从 Redis zset 取 Top10 ID → 数据库补详情 → 按 Redis 给的名次在 Python 里重新排序。

## 十、Chapter06 → Chapter07 变化小结

| 方面 | Chapter06 | Chapter07 |
|---|---|---|
| 用户关系 | 无 | 关注系统（`Contact` + 动态注入的 `following`/`followers`） |
| 首页内容 | 无动态 | 动态流（`actions` app，GenericForeignKey 多态关联） |
| 图片统计 | 无 | Redis 浏览计数 + 排行榜；`total_likes` 数据库字段（signal 维护） |
| 开发工具 | 无 | `django-debug-toolbar` |
| 技术债 | SSRF 风险 | 因未采用自定义 User 模型而产生的两处"补丁式"设计（`add_to_class`、`ABSOLUTE_URL_OVERRIDES`） |
