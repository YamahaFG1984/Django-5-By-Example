# Chapter14 教学笔记：educa 项目 —— 学生报名系统与多级缓存

Chapter14 把 educa 项目从"只有管理端"补齐成一个真正对外可用的产品：游客可以浏览的公开课程目录、学生注册/报名/学习界面，以及为了让这套读多写少的目录页面撑得住流量而引入的**多级缓存体系**（视图级 `cache_page`、模板片段缓存、底层缓存 API 三种一起用）。

## 一、新依赖

```diff
+ django-embed-video==1.4.9
+ pymemcache==4.0.0
+ django-debug-toolbar==4.3.0
+ redis==5.0.4
+ django-redisboard==8.4.0
```

- **`django-embed-video`**：把 YouTube/Vimeo 等视频链接自动转成嵌入播放器 iframe，不用自己解析各家视频网站的 URL 格式。
- **`django-debug-toolbar`**：和 Bookmarks 项目 Chapter07 一样的开发期 SQL/耗时分析工具。
- **`redis`**：Python Redis 客户端。
- **`django-redisboard`**：给 admin 后台加一个可视化查看 Redis 实例状态（内存占用、key 数量、连接数等）的面板。
- **`pymemcache`** ——这一个有点意外：`settings.py` 里最终配置的缓存后端是 Redis（`django.core.cache.backends.redis.RedisCache`），并没有用到 Memcached，`pymemcache` 这个 Memcached 客户端库看起来是**没有被实际使用的遗留依赖**（可能是作者原本考虑过用 Memcached 做缓存后端、后来改用 Redis 但忘了从 `requirements.txt` 里移除）。这是继续深读代码时又一处"依赖列表和实际实现不完全对应"的例子。

## 二、`settings.py`：缓存、Debug Toolbar、登录跳转

```python
LOGIN_REDIRECT_URL = reverse_lazy('student_course_list')

CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379',
    }
}
CACHE_MIDDLEWARE_ALIAS = 'default'
CACHE_MIDDLEWARE_SECONDS = 60 * 15  # 15 minutes
CACHE_MIDDLEWARE_KEY_PREFIX = 'educa'

INTERNAL_IPS = ['127.0.0.1']
```

`LOGIN_REDIRECT_URL` 从默认的 `/accounts/profile/` 改成了 `student_course_list`——登录成功后直接进"我的课程"列表，符合这个平台"学生登录是为了学习"的产品定位。`CACHES` 用 Django 5.x 新增的 `RedisCache` 内置后端（不再需要 `django-redis` 第三方包），`CACHE_MIDDLEWARE_*` 三兄弟是给"整站缓存中间件"用的默认配置（虽然这一章 `MIDDLEWARE` 里对应的两行中间件是**注释掉**的——`# 'django.middleware.cache.UpdateCacheMiddleware'`/`# 'django.middleware.cache.FetchFromCacheMiddleware'`，说明这套配置是为可能的整站缓存方案预留，本章实际用的是更细粒度的视图级/片段级缓存，见下）。

## 三、课程模型的两处新增：`students` 字段与多态 `render()` 方法

```python
# courses/models.py
class Course(models.Model):
    ...
    students = models.ManyToManyField(User, related_name='courses_joined', blank=True)
```

`Course.students`——课程和学生之间的多对多关系，`blank=True` 允许一门课程暂时没有任何学生报名。这是本章"报名"功能的数据基础。

```python
class ItemBase(models.Model):
    ...
    def render(self):
        return render_to_string(
            f'courses/content/{self._meta.model_name}.html',
            {'item': self},
        )
```

这是本章**设计上最精巧的一笔**——给抽象基类加了一个 `render()` 方法，让**每一种内容对象自己知道该用哪个模板渲染自己**：`self._meta.model_name` 在运行时会是 `'text'`/`'image'`/`'video'`/`'file'` 中的一个（取决于调用它的具体是哪个子类实例），拼出 `courses/content/text.html` 这样的模板路径去渲染。这样上层代码（`StudentCourseDetailView` 对应的模板）完全不需要写任何 `{% if item_type == 'text' %}...{% elif item_type == 'image' %}...` 这种分支判断，只需要对着任意一个 `content.item` 调用 `{{ item.render }}`，该显示成什么样子完全由内容对象自己决定——这是"胖模型"（Fat Models）设计思想的一个漂亮例子：把"这种类型的数据该怎么展示"这个知识放在模型这一层，而不是散落进每一处调用它的视图/模板里。四个模板都很简单：

```django
{# text.html #}
{{ item.content|linebreaks }}

{# image.html #}
<p><img src="{{ item.file.url }}" alt="{{ item.title }}"></p>

{# file.html #}
<p><a href="{{ item.file.url }}" class="button">Download file</a></p>

{# video.html #}
{% load embed_video_tags %}
{% video item.url "small" %}
```

`video.html` 里的 `{% video item.url "small" %}`（`django-embed-video` 提供的模板标签）是唯一需要额外处理逻辑的一个——它会解析 `item.url`（比如一个 YouTube 视频链接），自动生成对应视频服务商的嵌入播放器 HTML，`"small"` 是预设的尺寸规格。

## 四、公开课程目录：`CourseListView`/`CourseDetailView`

```python
# courses/views.py
class CourseListView(TemplateResponseMixin, View):
    model = Course
    template_name = 'courses/course/list.html'

    def get(self, request, subject=None):
        subjects = cache.get('all_subjects')
        if not subjects:
            subjects = Subject.objects.annotate(total_courses=Count('courses'))
            cache.set('all_subjects', subjects)
        all_courses = Course.objects.annotate(total_modules=Count('modules'))
        if subject:
            subject = get_object_or_404(Subject, slug=subject)
            key = f'subject_{subject.id}_courses'
            courses = cache.get(key)
            if not courses:
                courses = all_courses.filter(subject=subject)
                cache.set(key, courses)
        else:
            courses = cache.get('all_courses')
            if not courses:
                courses = all_courses
                cache.set('all_courses', courses)
        return self.render_to_response({'subjects': subjects, 'subject': subject, 'courses': courses})
```

这是本章**缓存策略里最细粒度的一种**——直接用 Django 底层缓存 API（`cache.get`/`cache.set`）手动实现"查缓存，没有就查数据库再写回缓存"的经典模式（Cache-Aside）：

- `Subject.objects.annotate(total_courses=Count('courses'))`/`Course.objects.annotate(total_modules=Count('modules'))`——用 ORM 的聚合注解一次性算出"每个学科下有几门课"、"每门课有几个模块"，避免在模板循环里对每个对象额外发起一次 `COUNT` 查询（典型的 N+1 优化）。
- **缓存键的设计**：`'all_subjects'`、`'all_courses'`、`f'subject_{subject.id}_courses'`——按"全部学科"、"全部课程"、"某学科下的课程"分别开独立的缓存槽位，而不是用一个大而全的键把所有变体都塞进去，这样不同筛选条件的浏览请求各自独立命中/更新自己的缓存条目。
- **`cache.set(key, courses)` 没有传第三个参数（过期时间）**，会使用 `CACHES['default']` 里没有显式配置的 `TIMEOUT`（Django 默认 300 秒/5 分钟）。

**这里有一个值得提醒的设计权衡**：整个视图**没有任何"数据变化时主动清除缓存"的逻辑**——管理员在后台新建/编辑/删除一门课程或学科后，公开课程目录页最长可能要等到默认缓存过期（约 5 分钟）才会显示最新数据。这是"读多写少"场景下用**定时过期而非主动失效**的缓存策略的典型取舍：实现简单（不需要在 `Course`/`Subject` 的 `save()`/`delete()` 里挂信号去清缓存），代价是接受几分钟的数据陈旧窗口。对于课程目录这种更新频率低、对"秒级实时"没有硬性要求的页面，这个权衡是合理的，但继续深读代码或者自己改造这套系统时，应该清楚这是一个**有意为之的最终一致性妥协**，而不是遗漏。

```python
class CourseDetailView(DetailView):
    model = Course
    template_name = 'courses/course/detail.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['enroll_form'] = CourseEnrollForm(initial={'course': self.object})
        return context
```

课程详情页对所有访客公开（没有 `LoginRequiredMixin`），但把"报名表单"预置在了 context 里——具体登不登录能不能报名，交给模板层的 `{% if request.user.is_authenticated %}` 判断（见下）。

## 五、`students` 应用：注册、报名、学习

```python
# students/forms.py
class CourseEnrollForm(forms.Form):
    course = forms.ModelChoiceField(queryset=Course.objects.none(), widget=forms.HiddenInput)

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.fields['course'].queryset = Course.objects.all()
```

`queryset=Course.objects.none()` 后又在 `__init__` 里重新赋值成 `Course.objects.all()`——乍看多此一举，其实是**惯用的延迟求值写法**：模块加载时（类体执行的那一刻）如果直接写 `queryset=Course.objects.all()`，这个查询集会在 Django 应用尚未完全初始化、数据库连接可能还没建立好的阶段就被求值，可能引发时序问题；放进 `__init__` 里，则保证每次真正实例化这个表单时才去查数据库，是 Django 表单里处理"字段的可选值依赖数据库查询"这类场景的标准写法。`widget=forms.HiddenInput` 说明这个字段的值不是用户手动选的，而是由视图传入 `initial={'course': self.object}` 预先填好、随表单一起提交回来的（对应课程详情页表单里那个隐藏的课程 ID 字段）。

```python
# students/views.py
class StudentRegistrationView(CreateView):
    template_name = 'students/student/registration.html'
    form_class = UserCreationForm
    success_url = reverse_lazy('student_course_list')

    def form_valid(self, form):
        result = super().form_valid(form)
        cd = form.cleaned_data
        user = authenticate(username=cd['username'], password=cd['password1'])
        login(self.request, user)
        return result
```

直接复用 Django 内置的 `UserCreationForm`（不需要自己写注册表单），`form_valid` 里在 `super().form_valid(form)` 真正把新用户存进数据库**之后**，紧接着用同一次提交里拿到的明文密码（`cd['password1']`，此时尚未从 `cleaned_data` 里清除）调用 `authenticate()` + `login()`——**注册成功后立刻自动登录**，用户不需要再单独走一次登录流程。这是一个很常见、体验更好的注册流程设计，对比 Bookmarks 项目 Chapter04 的 `register` 视图（注册成功后只是渲染一个"完成"页面，需要用户自己再去登录），educa 这里做得更完整。

```python
class StudentEnrollCourseView(LoginRequiredMixin, FormView):
    course = None
    form_class = CourseEnrollForm

    def form_valid(self, form):
        self.course = form.cleaned_data['course']
        self.course.students.add(self.request.user)
        return super().form_valid(form)

    def get_success_url(self):
        return reverse_lazy('student_course_detail', args=[self.course.id])
```

`self.course.students.add(self.request.user)`——多对多字段的 `.add()` 方法本身是**幂等**的（重复调用不会插入重复的关联记录，Django 底层会用 `INSERT ... ON CONFLICT`/等效机制处理），所以即便用户不小心对同一门课重复提交报名表单，也不会出现异常或者产生脏数据，这个视图不需要额外写"是否已经报过名"的判断逻辑。

```python
class StudentCourseListView(LoginRequiredMixin, ListView):
    model = Course
    template_name = 'students/course/list.html'
    def get_queryset(self):
        qs = super().get_queryset()
        return qs.filter(students__in=[self.request.user])

class StudentCourseDetailView(LoginRequiredMixin, DetailView):
    model = Course
    template_name = 'students/course/detail.html'
    def get_queryset(self):
        qs = super().get_queryset()
        return qs.filter(students__in=[self.request.user])

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        course = self.get_object()
        if 'module_id' in self.kwargs:
            context['module'] = course.modules.get(id=self.kwargs['module_id'])
        else:
            context['module'] = course.modules.all()[0]
        return context
```

这两个视图沿用了和 Chapter13 `OwnerMixin` **完全相同的思路**（用 `get_queryset()` 过滤而不是显式权限检查）——只不过这里过滤的条件从"我是课程的创建者"变成了"我是课程的报名学生"（`students__in=[self.request.user]`），同一种防护模式在不同业务语境下被复用。`get_context_data` 里"如果 URL 没带 `module_id` 就取第一个模块"（`course.modules.all()[0]`）是一个隐含的假设——**如果一门课程被报名了但还没有任何模块**，这里会因为对空列表取索引 `[0]` 而抛 `IndexError`。当前代码没有对这种边界情况做保护，是继续深读或改造这部分代码时值得注意的一个潜在问题。

## 六、视图级缓存与模板片段缓存

```python
# students/urls.py
path('course/<pk>/', cache_page(60 * 15)(views.StudentCourseDetailView.as_view()), name='student_course_detail'),
path('course/<pk>/<module_id>/', cache_page(60 * 15)(views.StudentCourseDetailView.as_view()), name='student_course_detail_module'),
```

`cache_page(60 * 15)`——**整个视图的响应结果**被缓存 15 分钟，第二次访问同一个 URL 时直接从缓存返回渲染好的 HTML，完全不会执行视图函数（不查数据库、不渲染模板）。这是本章缓存策略里**粒度最粗**的一种，直接包一层装饰器即可生效。

```django
{# students/course/detail.html #}
{% load cache %}
...
{% cache 600 module_contents module %}
  {% for content in module.contents.all %}
    {% with item=content.item %}
      <h2>{{ item.title }}</h2>
      {{ item.render }}
    {% endwith %}
  {% endfor %}
{% endcache %}
```

`{% cache 600 module_contents module %}...{% endcache %}`——**模板片段缓存**，比 `cache_page` 更细的粒度：只缓存页面里"模块内容列表"这一块 HTML（600 秒），页面的其余部分（比如模块导航栏）每次仍然正常渲染。`module_contents` 是这个缓存片段的名字，`module` 是额外的缓存键变量（不同的 `module` 对象会生成不同的缓存条目，缓存键实际上是 `module_contents` + `module` 的哈希组合）——这样即便整个页面因为 `cache_page` 已经被缓存了 15 分钟，这一段片段缓存本身单独还有一层 10 分钟的时效性（虽然在这个具体场景下，外层 `cache_page` 15 分钟已经完全覆盖了内层片段缓存的 10 分钟，内层缓存实际上很难有机会真正命中——这是**两层缓存粒度设计上有一点重叠冗余**的地方，比较适合作为"多级缓存要考虑各级的生效范围是否合理嵌套"这个原则的反面教材）。

## 七、登出方式的再次修复

对比 Chapter13（把登出改回了 GET 链接），Chapter14 的 `base.html` **又改回了 POST 表单**：

```django
<li>
  <form action="{% url "logout" %}" method="post">
    <button type="submit">Sign out</button>
    {% csrf_token %}
  </form>
</li>
```

这一处此前在 Chapter13 教学笔记里被标记为"安全回退"，本章又恢复成了正确写法——进一步印证了这本书的示例代码在不同章节之间会有反复调整、并不总是单调地朝着"更完善"的方向演进，继续深读代码时保持这种"逐章对比、不预设正确性"的习惯是有必要的。

## 八、`django-redisboard` 与 Debug Toolbar

`INSTALLED_APPS` 里加入 `'redisboard'` 后，admin 后台会自动出现一个 Redis 服务器监控面板（`redisboard` 内部自带 admin 注册逻辑），可以直接在浏览器里查看 Redis 实例的内存使用、连接数、键总数等指标——因为本章引入的缓存全部落在 Redis 上，这个面板能帮助在开发/调试阶段直接观察缓存的实际状态（比如确认某个 URL 是否真的被 `cache_page` 缓存住了）。`debug_toolbar` 的接入方式和 Bookmarks 项目 Chapter07 完全一致（`INSTALLED_APPS` 加注册、`MIDDLEWARE` 里放在合适位置、`INTERNAL_IPS` 限制访问、`urls.py` 加 `__debug__/` 路由）。

## 九、端到端流程

游客访问首页 → `CourseListView` 展示所有学科和课程（走 Redis 底层缓存 API，同类查询短期内不重复打数据库）→ 点进某门课的详情页（`CourseDetailView`，公开可见）→ 未登录会看到"注册以报名"的引导链接，已登录会看到报名表单 → 提交报名表单 → `StudentEnrollCourseView` 把当前用户加进 `course.students` → 跳转到"我的课程"里对应这门课的学习页 → `StudentCourseDetailView`（登录必需 + 报名必需，双重 `get_queryset` 过滤）展示模块导航和当前模块的内容列表，内容列表这一块走模板片段缓存，整个页面响应额外再走 `cache_page` 视图级缓存，每个内容条目通过 `item.render()` 自主决定渲染哪个类型专属的模板。

## 十、Chapter13 → Chapter14 变化小结

| 方面 | Chapter13 | Chapter14 |
|---|---|---|
| 可见范围 | 仅课程创建者本人可访问自己的管理后台 | 新增公开课程目录（游客可浏览）+ 学生学习界面 |
| 用户体系 | 仅"课程所有者"这一种角色 | 新增"学生"角色，`Course.students` M2M + 报名流程 |
| 内容展示方式 | 管理端表单编辑，无展示逻辑 | `ItemBase.render()` 多态自渲染，四种类型各自独立模板 |
| 性能优化 | 无 | 三层缓存：底层 API（课程目录）/ 视图级 `cache_page`（学习页）/ 模板片段（内容列表） |
| 视频内容 | `Video.url` 只是原始 URL 字段 | `django-embed-video` 自动生成嵌入播放器 |
| 登出方式 | 回退为 GET 链接（Ch13 标记的安全回退） | 再次修复为 POST 表单 |
| 运维可观测性 | 无 | `django-debug-toolbar` + `django-redisboard` |
| 已知遗留问题 | CSRF 豁免的排序接口 | 缓存无主动失效机制（陈旧窗口）；`pymemcache` 疑似未使用的遗留依赖；`modules.all()[0]` 潜在 `IndexError` |
