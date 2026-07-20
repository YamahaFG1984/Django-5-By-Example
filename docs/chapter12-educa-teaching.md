# Chapter12 教学笔记：第四个项目 educa —— 灵活内容模型与课程结构

Chapter12 开启本书**第四个、也是最后一个项目 educa**（在线教育平台）。这一章还没有涉及太多视图/前端交互，核心是把**数据模型设计**打磨到位：课程如何拆成模块，模块里的内容如何支持"文本/文件/图片/视频"四种完全不同的类型，以及"排序"这件小事背后藏着一个可复用的自定义字段类型。

## 一、项目结构

新项目 `educa`，单一 app `courses`。`requirements.txt` 精简回最基础的四行（`asgiref`、`Django`、`sqlparse`、`Pillow`），没有携带前几个项目的任何依赖——这是全新项目的起点。`settings.py` 也是标准的 `startproject` 生成物，`INSTALLED_APPS` 里 `'courses.apps.CoursesConfig'` 被放在了列表**最前面**（不是常见的排在 Django 内置 app 之后），这是个无关紧要的小细节，但也提示这本书里每个项目的脚手架代码并非完全一致的模板套用。

`urls.py` 直接用 Django 内置的 `LoginView`/`LogoutView`（不像 Bookmarks 项目那样通过 `include('django.contrib.auth.urls')`，而是显式手写两条 `path`）——这是同一个需求的另一种写法，只注册了登录/登出两条，没有注册密码重置等其它认证相关 URL，说明这一章暂时只需要"登录才能进后台管理内容"这个最基本的能力。

## 二、核心数据模型：课程 → 模块 → 内容

```python
class Subject(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200, unique=True)
    class Meta:
        ordering = ['title']

class Course(models.Model):
    owner = models.ForeignKey(User, related_name='courses_created', on_delete=models.CASCADE)
    subject = models.ForeignKey(Subject, related_name='courses', on_delete=models.CASCADE)
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200, unique=True)
    overview = models.TextField()
    created = models.DateTimeField(auto_now_add=True)
    class Meta:
        ordering = ['-created']

class Module(models.Model):
    course = models.ForeignKey(Course, related_name='modules', on_delete=models.CASCADE)
    title = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    order = OrderField(blank=True, for_fields=['course'])
    class Meta:
        ordering = ['order']
```

三层结构很直白：`Subject`（学科分类，比如"数学"、"编程"）→ `Course`（某学科下的一门课，有 `owner` 记录是谁创建的）→ `Module`（一门课拆成若干模块/章节，比如"第一章：变量与类型"）。这里首次出现的 `owner = models.ForeignKey(User, ...)`（直接用 Django 内置 `auth.User`，没有自定义 User 模型，也没有 Profile）——继续沿用了 Bookmarks 项目"不上自定义 User 模型"的做法，是否会在后续章节重新踩到 `add_to_class`/`ABSOLUTE_URL_OVERRIDES` 那类补丁，值得留意后续章节。

## 三、`Content` 模型：用 GenericForeignKey 装下四种完全不同的内容类型

这是本章**设计上最核心**的部分——一个模块里的"内容条目"可能是一段文字、一个文件、一张图片或一个视频链接，这四种东西的字段结构完全不同（文字有 `content` 文本字段，文件/图片有 `FileField`，视频只有一个 `URLField`），但又需要在同一个模块下统一排序、统一展示。

```python
class ItemBase(models.Model):
    owner = models.ForeignKey(User, related_name='%(class)s_related', on_delete=models.CASCADE)
    title = models.CharField(max_length=250)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)
    class Meta:
        abstract = True
    def __str__(self):
        return self.title

class Text(ItemBase):
    content = models.TextField()

class File(ItemBase):
    file = models.FileField(upload_to='files')

class Image(ItemBase):
    file = models.FileField(upload_to='images')

class Video(ItemBase):
    url = models.URLField()


class Content(models.Model):
    module = models.ForeignKey(Module, related_name='contents', on_delete=models.CASCADE)
    content_type = models.ForeignKey(
        ContentType,
        on_delete=models.CASCADE,
        limit_choices_to={'model__in': ('text', 'video', 'image', 'file')},
    )
    object_id = models.PositiveIntegerField()
    item = GenericForeignKey('content_type', 'object_id')
    order = OrderField(blank=True, for_fields=['module'])
    class Meta:
        ordering = ['order']
```

几个关键设计点：

- **`ItemBase` 是抽象基类**（`Meta.abstract = True`）——`Text`/`File`/`Image`/`Video` 四个具体模型各自继承它，每个都会生成**独立的数据库表**（`courses_text`、`courses_file`、`courses_image`、`courses_video`），而不是共用一张表。这和之前 Bookmarks 项目里 `Profile` 那种"实体继承"（`ForeignKey`/`OneToOneField` 关联）不同，抽象基类不会产生额外的表和 JOIN 开销，纯粹是"把公共字段定义写一遍，四个子类各自拷贝一份"的代码复用手段。
- **`owner` 字段用了 `related_name='%(class)s_related'`**——这是抽象基类里定义 `related_name` 时的标准写法，`%(class)s` 会在每个具体子类生成表时被替换成小写的类名，所以最终 `Text.owner` 的反向访问器是 `user.text_related`，`File.owner` 是 `user.file_related`，以此类推。如果四个子类都直接写死 `related_name='items_related'`，Django 会因为四张表都试图在 `User` 上注册同一个反向访问器名字而报错——`%(class)s` 占位符正是为了解决"一个抽象基类被多个子类继承、且都需要不同 `related_name`"这个问题而存在的。
- **`Content.content_type` + `object_id` + `item = GenericForeignKey(...)`**——这是本书第二次用 contenttypes 框架实现多态关联（第一次是 Bookmarks 项目 Chapter07 的 `Action.target`，用来指向"任意模型的一条动态记录"）。这里的用法更进一步：**通过 `limit_choices_to` 把可选的目标模型限制在 `Text`/`Video`/`Image`/`File` 这四种**，防止 admin 后台不小心把 `content_type` 关联到项目里其他风马牛不相及的模型上（比如 `Course` 或 `Subject` 自己）。
- 这套"抽象基类定义公共字段 + `ContentType` 框架做多态关联"的组合拳，是解决"同一个位置可能放不同类型内容"这类需求的经典 Django 模式，比"给 `Content` 表加四个可空外键字段（`text_id`、`file_id`、`image_id`、`video_id`），每次只有一个非空"的笨拙方案优雅得多——后者每加一种新内容类型就要改 `Content` 表结构，前者只需要新增一个 `ItemBase` 子类即可，`Content` 表本身完全不用动。

## 四、`OrderField`：一个自定义模型字段

```python
# courses/fields.py
class OrderField(models.PositiveIntegerField):
    def __init__(self, for_fields=None, *args, **kwargs):
        self.for_fields = for_fields
        super().__init__(*args, **kwargs)

    def pre_save(self, model_instance, add):
        if getattr(model_instance, self.attname) is None:
            try:
                qs = self.model.objects.all()
                if self.for_fields:
                    query = {
                        field: getattr(model_instance, field)
                        for field in self.for_fields
                    }
                    qs = qs.filter(**query)
                last_item = qs.latest(self.attname)
                value = getattr(last_item, self.attname) + 1
            except ObjectDoesNotExist:
                value = 0
            setattr(model_instance, self.attname, value)
            return value
        else:
            return super().pre_save(model_instance, add)
```

`Module.order = OrderField(blank=True, for_fields=['course'])`、`Content.order = OrderField(blank=True, for_fields=['module'])`——这是一个**自定义 Django 模型字段类**（继承 `models.PositiveIntegerField`），专门用来实现"新建一条记录时自动算出它排在同类记录的第几位"这件事，是本章技术含量最高的一段代码：

- **为什么重写 `pre_save` 而不是在 `save()` 方法或表单里手动算？** `pre_save()` 是 Django 字段类的一个钩子方法，在**每次保存该字段所在的模型实例之前**自动被调用，返回值就是这次要写入数据库的字段值。把"自动编号"的逻辑做成字段类本身的行为，意味着**任何用到这个字段的模型都能免费获得这个能力**——`Module` 和 `Content` 两个毫不相关的模型都直接声明 `order = OrderField(...)` 就自动有了"新建时自动排到最后"的行为，不需要在每个模型的 `save()` 里各自重复实现一遍。这是"自定义字段类型"这个 Django 扩展点的典型应用场景：把一种可复用的字段级行为封装成字段类，而不是散落在多个模型的业务逻辑里。
- **`for_fields` 参数——"在哪个范围内排序"**。`Module` 传的是 `for_fields=['course']`，意味着"模块的顺序编号只在同一门课程内部计算"（A 课程的模块从 0 开始编号，B 课程的模块也从 0 开始编号，互不影响）；`Content` 传的是 `for_fields=['module']`，同理是"内容条目的顺序只在同一个模块内部计算"。`pre_save` 里 `qs.filter(**query)` 这一步就是用 `for_fields` 里列出的字段名，动态构造出"和当前正在保存的这条记录同属一组"的过滤条件（比如 `Module.objects.filter(course=<当前课程>)`），再用 `.latest(self.attname)` 找出这一组里 `order` 值最大的那条，加一即为新记录的顺序号。
- **`getattr(model_instance, self.attname) is None` 这个判断很关键**——只有在**用户没有手动指定 `order` 值**的情况下才会触发自动编号逻辑；如果调用方显式传了 `order=5`，就会走 `else` 分支直接用 `super().pre_save()` 的默认行为（原样存这个值）。这保证了"自动编号"只是一个**默认行为**而不是强制覆盖，未来如果要支持手动拖拽调整顺序（写死某个具体的 `order` 值），这个字段类完全不需要改动。
- `except ObjectDoesNotExist: value = 0`——如果这是这门课程的第一个模块（或这个模块的第一条内容），`.latest()` 会因为查询结果为空而抛 `ObjectDoesNotExist`，这时候顺理成章地把顺序号设成 `0`，作为这一组里的第一条记录。

## 五、Admin 后台

```python
class ModuleInline(admin.StackedInline):
    model = Module

@admin.register(Course)
class CourseAdmin(admin.ModelAdmin):
    list_display = ['title', 'subject', 'created']
    list_filter = ['created', 'subject']
    search_fields = ['title', 'overview']
    prepopulated_fields = {'slug': ('title',)}
    inlines = [ModuleInline]
```

`ModuleInline` 用的是 `StackedInline`（每条模块记录用一整块表单区域展示，字段上下堆叠排列），而不是像 Chapter09 `OrderItemInline` 那样用 `TabularInline`（每条记录一行紧凑展示）——选择哪种主要取决于内联记录里的字段数量和长度：`Module` 有 `description` 这种可能较长的文本字段，用 `StackedInline` 每条记录能有更宽裕的展示空间；`OrderItem` 只有寥寥几个短字段，适合用 `TabularInline` 紧凑地列成表格。这一 app 目前只注册了 `Subject` 和 `Course`（带 `Module` 内联），`Content` 和四个具体内容类型模型都还没有注册到 admin——说明本章的重点仍停留在打好数据模型的地基，内容的编辑界面留给后续章节实现。

## 六、`Subject` 的初始数据：Fixture

```json
[
  {"model": "courses.subject", "pk": 1, "fields": {"title": "Mathematics", "slug": "mathematics"}},
  {"model": "courses.subject", "pk": 2, "fields": {"title": "Music", "slug": "music"}},
  {"model": "courses.subject", "pk": 3, "fields": {"title": "Physics", "slug": "physics"}},
  {"model": "courses.subject", "pk": 4, "fields": {"title": "Programming", "slug": "programming"}}
]
```

`courses/fixtures/subjects.json` 是 Django **fixture** 机制的标准用法——预置几条基础学科分类数据，配合 `python manage.py loaddata subjects` 命令一次性导入，避免每次搭建新环境（开发机、CI、演示环境）都要手动在 admin 里逐条录入这几个学科。这是本书目前为止第一次使用 fixture 机制做初始数据播种（之前几个项目都是纯手动在 admin 里建数据）。

## 七、模板与前端

`base.html` 的结构和之前几个项目的基础模板高度相似（`{% load static %}`、顶部导航、`{% block content %}`、POST 表单实现登出），但多了一个之前项目里没有见过的通用钩子：

```django
<script>
  document.addEventListener('DOMContentLoaded', (event) => {
    {% block domready %}
    {% endblock %}
  })
</script>
```

这其实是 Bookmarks 项目 Chapter06 里就出现过的 `{% block domready %}` 约定（当时用于图片点赞的 AJAX 逻辑），这里从项目一开始就把这个钩子模式**预先搭进了基础模板**，意味着后续章节大概率会大量使用 AJAX/JS 交互（这本书官方目录后续确实会讲到课程模块的拖拽排序——这正是 `OrderField` 这个字段今天先做好铺垫的原因：模型层已经支持"顺序"这个概念，后面只需要一个视图接收前端拖拽后的新顺序、批量更新 `order` 值即可）。

`registration/login.html`/`logged_out.html` 两个模板结构上和之前项目里见过的版本几乎一致，是这本书里反复复用的标准认证模板样式。

## 八、Chapter11(myshop) → Chapter12(educa) 项目切换小结

Chapter12 不是接着 myshop 项目往下写，而是另起一个全新项目，几个值得对比的设计选择：

| 方面 | myshop（Ch08-11） | educa（Ch12） |
|---|---|---|
| 内容多态性需求 | 无（商品字段结构固定） | `Content` 用 `GenericForeignKey` 关联四种完全不同结构的内容类型 |
| 排序需求 | 无显式排序字段（除了 `-created`） | 自定义 `OrderField` 字段类型，两处复用（`Module`、`Content`） |
| 代码复用手段 | 无抽象基类 | `ItemBase` 抽象基类 + `%(class)s` related_name 占位符 |
| 初始数据 | 全靠手动 admin 录入 | 引入 Django fixture 机制（`subjects.json`） |
| User 模型策略 | 沿用默认 `auth.User` | 同样沿用默认 `auth.User`（`Course.owner`/`ItemBase.owner`） |
| 前端交互铺垫 | 无 | 基础模板预置 `{% block domready %}`，为后续 AJAX/拖拽功能做准备 |
