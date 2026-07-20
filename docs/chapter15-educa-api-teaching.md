# Chapter15 教学笔记：educa 项目 —— 用 Django REST Framework 暴露 API

Chapter15 给 educa 项目加上一层 REST API（基于 Django REST Framework，简称 DRF），让课程数据能被移动端 App 或第三方脚本消费，而不只是通过网页浏览。这一章也顺带把 Chapter13/14 攒下来的模型设计成果（`OrderField`、`GenericForeignKey`、`render()` 多态方法、Django 权限系统）几乎原样复用了一遍——**好的模型设计一旦到位，加一层 API 的成本会小很多**。

## 一、新依赖与全局配置

```diff
+ djangorestframework==3.15.1
+ requests==2.31.0
```

`requests` 不是给 Django 服务端用的，是给本章末尾的**独立命令行脚本**（`api_examples/enroll_all.py`，用来演示如何用 Python 调用刚做好的 API）准备的客户端库。

```python
INSTALLED_APPS += ['rest_framework']

REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.DjangoModelPermissionsOrAnonReadOnly'
    ]
}
```

`DjangoModelPermissionsOrAnonReadOnly` 是这一章的一个关键决策——它**直接复用 Django 内置的模型级权限系统**（也就是 Chapter13 `PermissionRequiredMixin` 用过的那套 `courses.view_course`/`courses.add_course` 权限），效果是：**未登录的匿名用户只能做只读的 GET 请求**（浏览课程列表、看课程详情），任何写操作（POST/PUT/DELETE）都必须是已登录**且**在 Django 权限系统里被授予了对应权限的用户才能执行。这意味着这一章完全没有重新设计一套 API 专属的权限模型，而是让 Web 端和 API 端共享同一套底层授权数据——一个用户在 admin 后台被赋予的"能不能编辑课程"的权限，天然地也决定了他能不能通过 API 编辑课程。

`educa/urls.py` 新增 `path('api/', include('courses.api.urls', namespace='api'))`，API 相关代码被整齐地放进 `courses/api/` 子包（`serializers.py`/`views.py`/`urls.py`/`permissions.py`/`pagination.py`），和面向网页的 `views.py`/`urls.py` 物理隔离，是比较规范的"网页视图和 API 视图分离"的项目组织方式。

## 二、序列化器（Serializers）：让模型能被翻译成 JSON

```python
# courses/api/serializers.py
class ModuleSerializer(serializers.ModelSerializer):
    class Meta:
        model = Module
        fields = ['order', 'title', 'description']


class CourseSerializer(serializers.ModelSerializer):
    modules = ModuleSerializer(many=True, read_only=True)
    class Meta:
        model = Course
        fields = ['id', 'subject', 'title', 'slug', 'overview', 'created', 'owner', 'modules']
```

`ModelSerializer` 是 DRF 里 `ModelForm` 概念在"数据序列化"场景下的对应物——根据 `Meta.model` + `Meta.fields` 自动推导出该输出/接收哪些字段、每个字段用什么类型校验。`CourseSerializer` 里嵌套了一个 `ModuleSerializer(many=True, read_only=True)`——这样请求某门课程的详情时，返回的 JSON 会**内联包含它所有模块的完整信息**，而不需要客户端再对每个模块单独发一次请求，是 REST API 里"要不要嵌套关联数据"这个常见设计决策的一次具体取舍（这里选择了嵌套，`read_only=True` 意味着这个嵌套字段只用于输出展示，不接受客户端通过它反向创建/修改模块）。

### `SubjectSerializer`：混合"模型字段"和"计算字段"

```python
class SubjectSerializer(serializers.ModelSerializer):
    total_courses = serializers.IntegerField()
    popular_courses = serializers.SerializerMethodField()

    def get_popular_courses(self, obj):
        courses = obj.courses.annotate(total_students=Count('students')).order_by('-total_students')[:3]
        return [f'{c.title} ({c.total_students} students)' for c in courses]

    class Meta:
        model = Subject
        fields = ['id', 'title', 'slug', 'total_courses', 'popular_courses']
```

- `total_courses = serializers.IntegerField()`——`Subject` 模型本身并没有这个字段，它依赖调用方（视图层，见下）用 `.annotate(total_courses=Count('courses'))` 现场算出来并挂在查询结果对象上，序列化器这里只是声明"这个字段存在、类型是整数"，具体值从哪来是**视图的责任**，序列化器不关心。这和 Chapter14 `CourseListView` 里用 `annotate` 给模板变量注入统计值是同一种"数据库层面聚合、避免应用层循环计数"的优化思路，只是这次伺候的是 API 而不是模板。
- `popular_courses = serializers.SerializerMethodField()`——这是一个完全**由代码现场计算**、模型上根本不存在对应字段的输出字段，`get_popular_courses(self, obj)` 方法名严格遵循 `get_<字段名>` 的约定被 DRF 自动调用。这里现场查询"这个学科下报名人数最多的三门课"，返回的不是结构化数据而是**拼接好的字符串列表**（`"Python Basics (120 students)"` 这种格式）——这是一个值得注意的设计选择：API 返回给客户端的是一段已经格式化好、供人类直接阅读的文本，而不是让客户端自己拿到 `{title, student_count}` 这样的结构化数据再自行组装展示文案。这种做法牺牲了一定的客户端灵活性（如果不同客户端想用不同的展示格式，没法只改前端），换来了服务端逻辑的简单直接——是教学示例里常见的权衡，实际生产 API 设计中更常见的做法是返回结构化字段，把格式化留给消费方。

### `ItemRelatedField`：让 API 直接复用 `render()` 生成的 HTML

```python
class ItemRelatedField(serializers.RelatedField):
    def to_representation(self, value):
        return value.render()

class ContentSerializer(serializers.ModelSerializer):
    item = ItemRelatedField(read_only=True)
    class Meta:
        model = Content
        fields = ['order', 'item']
```

这是本章**最值得玩味的一处设计**——`ItemRelatedField` 是一个自定义的 DRF 字段类型，`to_representation` 决定了这个字段在输出 JSON 时该怎么表示，这里直接调用了 Chapter14 给 `ItemBase` 加的 `render()` 方法（那个方法本来是给 Django 模板系统用的，返回一段**渲染好的 HTML 字符串**）。也就是说，**这个 REST API 在 `item` 字段里直接返回一段 HTML 片段**，而不是返回 `{type: 'text', content: '...'}` 这种更"REST 风格"的结构化多态数据。

这样做的好处是**最大化复用**了 Chapter14 已经写好的四套内容类型模板（`text.html`/`image.html`/`video.html`/`file.html`），移动端 App 只要能渲染 HTML（比如用一个 WebView 组件）就能直接展示课程内容，完全不需要自己重新实现"这四种内容类型分别该怎么显示"的逻辑。代价是这打破了"API 应该返回纯数据、样式和展现交给客户端决定"的常规 REST 设计原则——如果未来要做一个原生 App（不依赖 WebView 渲染 HTML），这个字段目前的形态就不够用了，需要另外设计。这是一个"教学示例里刻意复用已有代码，但不完全是生产级 API 设计最佳实践"的典型例子，值得辩证看待。

## 三、ViewSet：把增删改查自动组织成标准 REST 端点

```python
# courses/api/views.py
class SubjectViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Subject.objects.annotate(total_courses=Count('courses'))
    serializer_class = SubjectSerializer
    pagination_class = StandardPagination


class CourseViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Course.objects.prefetch_related('modules')
    serializer_class = CourseSerializer
    pagination_class = StandardPagination
```

`ReadOnlyModelViewSet`——DRF 提供的一种通用视图集，自动生成"列表"和"详情"两个只读端点（对应 `GET /api/courses/` 和 `GET /api/courses/<id>/`），不提供创建/更新/删除（如果要读写全套，可以换成 `ModelViewSet`）。这里选只读，是因为课程数据的创建/编辑本来就该走 Chapter13 已经做好的网页管理端（`OwnerMixin` 那一套），API 目前的定位是"给客户端读数据 + 报名/查看内容"，不是"给客户端管理课程"。

`Subject.objects.annotate(total_courses=Count('courses'))` 这行直接出现在 `queryset` 类属性里——每次列出学科时都自动带上课程数量统计，配合上面提到的 `SubjectSerializer.total_courses` 字段读取这个 annotate 出来的值。`Course.objects.prefetch_related('modules')`——因为 `CourseSerializer` 会嵌套渲染每门课的所有模块，这里提前用 `prefetch_related` 把模块数据一次性批量取出来，避免"列出 N 门课程，每门课程都单独发一次查模块的 SQL"这种 N+1 问题——这是 API 场景下比网页模板更容易被放大的性能陷阱（API 经常被批量拉取列表数据，N+1 问题在这种场景下影响更明显）。

### `@action`：给 ViewSet 加自定义端点

```python
class CourseViewSet(viewsets.ReadOnlyModelViewSet):
    ...
    @action(
        detail=True, methods=['post'],
        authentication_classes=[BasicAuthentication],
        permission_classes=[IsAuthenticated],
    )
    def enroll(self, request, *args, **kwargs):
        course = self.get_object()
        course.students.add(request.user)
        return Response({'enrolled': True})

    @action(
        detail=True, methods=['get'],
        serializer_class=CourseWithContentsSerializer,
        authentication_classes=[BasicAuthentication],
        permission_classes=[IsAuthenticated, IsEnrolled],
    )
    def contents(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)
```

`@action` 装饰器让一个 ViewSet 在标准的"列表/详情"端点之外，额外挂载自定义的业务端点：`detail=True` 表示这个动作作用于单个对象（URL 形如 `/api/courses/<id>/enroll/`）。两个自定义端点各自有**独立的鉴权配置**（覆盖了整个视图集/全局的默认设置）：

- **`enroll`**：POST 请求，`authentication_classes=[BasicAuthentication]`（HTTP Basic 认证，客户端在请求头里带用户名密码）+ `permission_classes=[IsAuthenticated]`（必须是已认证用户）。这是本章第一次在 API 层重新实现"报名"这个动作——`course.students.add(request.user)` 和 Chapter14 网页端 `StudentEnrollCourseView.form_valid` 里的那行代码**逻辑完全一样**，只是这次是给 API 客户端用的独立入口，而不是走网页表单。
- **`contents`**：GET 请求，除了要求认证，还多了一个**自定义权限类** `IsEnrolled`：

```python
# courses/api/permissions.py
class IsEnrolled(BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.students.filter(id=request.user.id).exists()
```

`has_object_permission` 是 DRF 权限类里专门用来做"对象级"权限判断的钩子（区别于"你能不能访问这个端点"这种笼统的权限，这里判断的是"你能不能访问**这一个具体的**课程对象"）——逻辑就是"当前用户是否在这门课的报名学生名单里"，和 Chapter14 网页端 `StudentCourseDetailView.get_queryset()` 里 `filter(students__in=[self.request.user])` 是**同一条业务规则的两种实现**：网页端用查询过滤（查不到就 404），API 端用显式的对象级权限检查（查得到但没权限会返回 403）。这里能看出同一条访问控制规则，在 Django 通用视图和 DRF 权限体系里各自惯用的实现方式并不相同，但背后的业务意图是一致的。

`contents` 端点复用了标准 `retrieve()` 方法（DRF `RetrieveModelMixin` 提供），但通过 `serializer_class=CourseWithContentsSerializer` 覆盖了默认的序列化器——同一个"查询单个课程详情"的底层逻辑，配上不同的序列化器就能返回不同详略程度的数据（普通详情端点只给模块列表，`contents` 端点连每个模块下的具体内容条目都嵌套返回）。

## 四、分页

```python
# courses/api/pagination.py
class StandardPagination(PageNumberPagination):
    page_size = 10
    page_size_query_param = 'page_size'
    max_page_size = 50
```

`PageNumberPagination` 是 DRF 内置的按页码分页方案（`?page=2` 这种），`page_size_query_param = 'page_size'` 额外允许客户端通过查询参数自定义每页大小（比如 `?page_size=20`），`max_page_size = 50` 给这个自定义空间设了上限，防止客户端传一个 `page_size=100000` 之类的极端值把服务端一次性查询/返回的数据量拖垮——这是设计公开 API 时一个常见的自我保护措施。

## 五、路由：`DefaultRouter`

```python
# courses/api/urls.py
router = routers.DefaultRouter()
router.register('courses', views.CourseViewSet)
router.register('subjects', views.SubjectViewSet)

urlpatterns = [
    path('', include(router.urls)),
]
```

`DefaultRouter` 是 DRF 提供的自动路由生成器——只要注册一个 ViewSet，它会自动按 REST 惯例生成一整套 URL（列表、详情、以及所有用 `@action` 声明的自定义端点），不需要像传统 Django `urlpatterns` 那样一条条手写 `path()`。文件里还留着几行被注释掉的旧式 `path()` 声明（`subject_list`/`subject_detail`/`course_enroll`），说明这是本章从"手写 Django 视图 + URL"迁移到"DRF ViewSet + Router"过程中留下的痕迹——注释而不是删除，方便读者对照两种写法的差异。

## 六、命令行客户端示例：`api_examples/enroll_all.py`

```python
import requests

base_url = 'http://127.0.0.1:8000/api/'
url = f'{base_url}courses/'
available_courses = []

while url is not None:
    r = requests.get(url)
    response = r.json()
    url = response['next']
    courses = response['results']
    available_courses += [course['title'] for course in courses]

for course in courses:
    r = requests.post(
        f'{base_url}courses/{course["id"]}/enroll/',
        auth=(username, password),
    )
```

这是一个独立于 Django 项目之外的纯 Python 脚本，用来演示"别的程序如何消费这套 API"：

- **分页遍历**：`while url is not None` 循环利用了 DRF `PageNumberPagination` 返回结果里的标准字段 `next`（下一页的完整 URL，没有下一页时是 `null`）——客户端不需要自己拼页码，只需要不断跟着 `next` 走，直到它变成 `None`，是消费分页 API 的标准写法。
- **`auth=(username, password)`**：`requests` 库对 HTTP Basic 认证的原生支持，直接对应服务端 `enroll` 端点声明的 `authentication_classes=[BasicAuthentication]`。

## 七、端到端流程

客户端 `GET /api/subjects/` → 拿到学科列表（附带每个学科的课程总数和最热门三门课的统计描述）→ `GET /api/courses/` 翻页遍历所有课程（附带每门课的模块列表）→ 对感兴趣的课程 `POST /api/courses/<id>/enroll/`（带 Basic Auth）完成报名 → 报名成功后 `GET /api/courses/<id>/contents/`（同样需要认证 + `IsEnrolled` 对象级权限校验）获取这门课完整的模块+内容详情，每条内容的 `item` 字段直接是可渲染的 HTML 片段，客户端拿到后可以直接嵌入展示。

## 八、Chapter14 → Chapter15 变化小结

| 方面 | Chapter14 | Chapter15 |
|---|---|---|
| 数据访问方式 | 仅网页（Django 模板渲染） | 新增 REST API（DRF ViewSet + Router） |
| 权限模型 | Django 内置权限（`PermissionRequiredMixin`） | 直接复用同一套权限（`DjangoModelPermissionsOrAnonReadOnly`），API 与网页共享授权数据 |
| 报名逻辑 | 仅网页表单（`StudentEnrollCourseView`） | 网页表单 + API 自定义 `@action`（`enroll`），核心逻辑重复实现了一份 |
| 访问控制粒度 | 查询过滤（`get_queryset` 收窄结果集） | 查询过滤（列表）+ 显式对象级权限类 `IsEnrolled`（`contents` 端点） |
| 内容展示 | `item.render()` 服务于 Django 模板 | `item.render()` 被 API 序列化器直接复用，返回 HTML 而非结构化 JSON |
| 消费方 | 仅浏览器 | 浏览器 + 任意 HTTP 客户端（示例：`api_examples/enroll_all.py`） |
