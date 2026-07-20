# Chapter13 教学笔记：educa 项目 —— 课程管理 CBV、动态内容表单与拖拽排序

Chapter13 是 educa 项目**管理端**的核心一章：用一整套基于类的通用视图（Class-Based Views）+ 自定义 Mixin 实现课程的增删改查、用 Formset 批量编辑模块、用**运行时动态解析模型**的方式统一处理四种不同内容类型的表单，最后用 AJAX + 拖拽库实现模块/内容的排序调整——直接用上了 Chapter12 埋下的 `OrderField`。

## 一、新依赖

```diff
+ django-braces==1.15.0
```

`django-braces` 提供了一批可以直接和 Django 内置 CBV 组合的 Mixin，本章用到两个：`CsrfExemptMixin`（跳过 CSRF 校验）、`JsonRequestResponseMixin`（自动把请求体当 JSON 解析、自动把返回值序列化成 `JsonResponse`）。`myshop/educa/urls.py` 新增 `path('course/', include('courses.urls'))`，把 `courses` app 自己的 URL 挂载到 `/course/` 前缀下。

## 二、三个自定义 Mixin：把"只能操作自己的课程"这件事写一次、到处复用

```python
class OwnerMixin:
    def get_queryset(self):
        qs = super().get_queryset()
        return qs.filter(owner=self.request.user)

class OwnerEditMixin:
    def form_valid(self, form):
        form.instance.owner = self.request.user
        return super().form_valid(form)

class OwnerCourseMixin(OwnerMixin, LoginRequiredMixin, PermissionRequiredMixin):
    model = Course
    fields = ['subject', 'title', 'slug', 'overview']
    success_url = reverse_lazy('manage_course_list')

class OwnerCourseEditMixin(OwnerCourseMixin, OwnerEditMixin):
    template_name = 'courses/manage/course/form.html'
```

这是 Django 通用视图（Generic CBV）体系里"关注点拆分"的典型示范：

- `OwnerMixin` 覆盖 `get_queryset()`——凡是继承它的列表/详情/更新/删除视图，**查询出来的对象集合永远先被过滤成"只属于当前登录用户"**。这意味着即便一个用户篡改 URL 里的 `pk` 试图编辑别人的课程，`get_object_or_404` 在这个被收窄的 queryset 里根本查不到那条记录，会直接 404，而不是权限判断失败——**用"查不到"代替"无权访问"**，这样连"这门课程确实存在，只是不是你的"这个信息都不会泄露给攻击者，是比显式权限检查更严格的一种防护姿态。
- `OwnerEditMixin` 覆盖 `form_valid()`——凡是继承它的创建/更新视图，保存表单前**自动把 `owner` 字段设成当前登录用户**，用户完全不需要（也不能）在表单里自己选"这门课归谁"，杜绝了"伪造 owner 字段把课程据为己有"的可能。
- `OwnerCourseMixin` 把 `OwnerMixin` 和 Django 内置的 `LoginRequiredMixin`（必须登录）、`PermissionRequiredMixin`（必须有特定权限）组合在一起，还统一声明了 `model`、`fields`、`success_url`——四个具体视图（`ManageCourseListView`/`CourseCreateView`/`CourseUpdateView`/`CourseDeleteView`）只需要各自声明一个 `permission_required` 字符串，其余全部继承自这套 Mixin 组合：

```python
class ManageCourseListView(OwnerCourseMixin, ListView):
    template_name = 'courses/manage/course/list.html'
    permission_required = 'courses.view_course'

class CourseCreateView(OwnerCourseEditMixin, CreateView):
    permission_required = 'courses.add_course'

class CourseUpdateView(OwnerCourseEditMixin, UpdateView):
    permission_required = 'courses.change_course'

class CourseDeleteView(OwnerCourseMixin, DeleteView):
    template_name = 'courses/manage/course/delete.html'
    permission_required = 'courses.delete_course'
```

`'courses.view_course'`/`'courses.add_course'` 这些字符串是 Django **默认自动生成**的模型级权限（每个模型注册时会自动生成 `add_<model>`/`change_<model>`/`delete_<model>`/`view_<model>` 四条权限，`app_label.codename` 拼接成完整权限标识），`PermissionRequiredMixin` 会在请求进来时检查 `request.user.has_perm(permission_required)`，没有该权限直接返回 403。这套权限目前还没有在代码里看到"给哪个用户/组授予哪些权限"的配置——意味着权限的分配要靠管理员事后在 admin 后台的用户/用户组管理界面手动勾选，这是 Django 内置权限体系"声明检查逻辑在代码里，分配授权在数据里（运行时可调整）"的典型分工。

这套四个视图**完全没有单独写 `forms.py` 里的表单类**——`fields = ['subject', 'title', 'slug', 'overview']` 直接声明在 Mixin 上，Django 通用编辑视图会在背后自动用 `modelform_factory` 现场生成一个 `ModelForm`。这是 Django CBV 体系里"表单足够简单就不必显式定义 Form 类"的惯用做法。

## 三、模块管理：`Formset` 一次性编辑多个模块

```python
# forms.py
ModuleFormSet = inlineformset_factory(
    Course, Module, fields=['title', 'description'], extra=2, can_delete=True,
)
```

```python
class CourseModuleUpdateView(TemplateResponseMixin, View):
    template_name = 'courses/manage/module/formset.html'
    course = None

    def get_formset(self, data=None):
        return ModuleFormSet(instance=self.course, data=data)

    def dispatch(self, request, pk):
        self.course = get_object_or_404(Course, id=pk, owner=request.user)
        return super().dispatch(request, pk)

    def get(self, request, *args, **kwargs):
        formset = self.get_formset()
        return self.render_to_response({'course': self.course, 'formset': formset})

    def post(self, request, *args, **kwargs):
        formset = self.get_formset(data=request.POST)
        if formset.is_valid():
            formset.save()
            return redirect('manage_course_list')
        return self.render_to_response({'course': self.course, 'formset': formset})
```

`inlineformset_factory(Course, Module, ...)`——这是 Django 表单系统里专门处理"一对多关系批量编辑"的机制：一门课程下有多个模块，用户在一个页面上一次性新增/编辑/删除多个模块，而不用为每个模块单独打开一个页面。`extra=2` 表示表单默认多渲染两组空白的"新增模块"输入框；`can_delete=True` 会给每个已有模块加一个"删除"复选框，勾选后提交表单会把对应模块删除。

这个视图**没有继承任何 Django 内置的通用编辑视图**（`CreateView`/`UpdateView` 都不适合"一次编辑一组关联对象"这种场景），而是直接从最底层的 `View` + `TemplateResponseMixin` 手写 `get`/`post`。`dispatch()` 被重写用来提前拿到并校验 `self.course`（同时兼顾了"确认这门课程存在"和"确认属于当前用户"两件事，同样是用查询过滤而不是显式权限判断的方式做隔离），这个属性之后在 `get`/`post` 两个方法里都能直接复用，避免重复查询。

## 四、`ContentCreateUpdateView`：本章技术含量最高的一个视图

这是全书目前为止**最动态**的一段代码——需要用**同一个视图类**处理文本、图片、视频、文件四种完全不同的模型，还要同时兼顾"新建"和"编辑"两种模式：

```python
class ContentCreateUpdateView(TemplateResponseMixin, View):
    module = None
    model = None
    obj = None
    template_name = 'courses/manage/content/form.html'

    def get_model(self, model_name):
        if model_name in ['text', 'video', 'image', 'file']:
            return apps.get_model(app_label='courses', model_name=model_name)
        return None

    def get_form(self, model, *args, **kwargs):
        Form = modelform_factory(model, exclude=['owner', 'order', 'created', 'updated'])
        return Form(*args, **kwargs)

    def dispatch(self, request, module_id, model_name, id=None):
        self.module = get_object_or_404(Module, id=module_id, course__owner=request.user)
        self.model = self.get_model(model_name)
        if id:
            self.obj = get_object_or_404(self.model, id=id, owner=request.user)
        return super().dispatch(request, module_id, model_name, id)

    def get(self, request, module_id, model_name, id=None):
        form = self.get_form(self.model, instance=self.obj)
        return self.render_to_response({'form': form, 'object': self.obj})

    def post(self, request, module_id, model_name, id=None):
        form = self.get_form(self.model, instance=self.obj, data=request.POST, files=request.FILES)
        if form.is_valid():
            obj = form.save(commit=False)
            obj.owner = request.user
            obj.save()
            if not id:
                Content.objects.create(module=self.module, item=obj)
            return redirect('module_content_list', self.module.id)
        return self.render_to_response({'form': form, 'object': self.obj})
```

- **`apps.get_model(app_label='courses', model_name=model_name)`**——Django 应用注册表（App Registry）提供的运行时反射能力，根据字符串（`'text'`/`'video'`/`'image'`/`'file'`，来自 URL 路径参数）**动态找到对应的模型类**，而不是在代码里写死 `if model_name == 'text': model = Text elif ...` 这种分支判断。`get_model` 方法先用白名单 `if model_name in [...]` 校验合法性——这一步很关键，防止有人往 URL 里塞一个任意字符串（比如 `'user'` 或 `'course'`）试图通过这个通用入口去创建/编辑本不该被这样操作的模型。
- **`modelform_factory(model, exclude=[...])`**——同样是运行时动态生成表单类，根据传入的具体模型类（`Text`/`Video`/`Image`/`File` 中的一个）自动生成对应的 `ModelForm`，排除掉 `owner`/`order`/`created`/`updated` 这几个不该让用户在表单里手动填写的字段。这样**一个视图类就能同时服务四种模型**，不需要为每种内容类型单独写四份几乎一样的视图代码。
- **`dispatch()` 里 `if id:` 判断是否处于编辑模式**——这个视图靠 URL 是否带 `id` 参数（`module_content_create` vs `module_content_update` 两条不同的 URL 规则，但指向**同一个视图类**）区分"新建"还是"编辑"：新建时 `self.obj` 是 `None`，表单是空的；编辑时提前查出已有对象作为表单的 `instance`。
- **`post()` 里 `if not id:` 决定要不要新建 `Content` 包装记录**——保存具体内容对象（`Text`/`Image`/...）本身之后，**只有在"新建"这个分支**才需要额外创建一条 `Content` 记录把它和所属 `Module` 关联起来（通过 `GenericForeignKey` 的 `item=obj` 赋值方式）；如果是编辑已有内容，`Content` 记录早就存在，不需要重复创建。

```python
class ContentDeleteView(View):
    def post(self, request, id):
        content = get_object_or_404(Content, id=id, module__course__owner=request.user)
        module = content.module
        content.item.delete()
        content.delete()
        return redirect('module_content_list', module.id)
```

删除时需要**两次删除**——`content.item.delete()` 删掉真正的内容对象（`Text`/`Image`/`Video`/`File` 表里的那一行），`content.delete()` 再删掉 `Content` 这条"包装/索引"记录本身。这是 `GenericForeignKey` 关联方式的必然代价：它不像普通外键那样能通过 `on_delete=CASCADE` 自动级联删除，因为 `Content` 和它指向的具体对象之间**没有真正的数据库外键约束**（`ContentType` + `object_id` 只是逻辑关联，数据库层面并不知道这是一种关联关系），所以级联删除的逻辑必须由应用代码手动处理。

### 模板里配合的自定义模板过滤器

```python
# templatetags/course.py
@register.filter
def model_name(obj):
    try:
        return obj._meta.model_name
    except AttributeError:
        return None
```

`content_list.html` 里 `{{ item|model_name }}` 用它来判断一个内容对象具体是 `Text`/`Image`/`Video`/`File` 中的哪一个，从而拼出正确的 `module_content_update` 编辑链接（这个 URL 需要显式带上 `model_name` 参数）——这是模板层"反向识别 `GenericForeignKey` 具体指向哪种模型"的常见手法，因为在模板语境里没法直接调用 Python 的 `type()`，需要一个自定义过滤器把 `obj._meta.model_name` 这个 Python 层的元信息暴露出来。

## 五、拖拽排序：AJAX + `OrderField` 的落地

`content_list.html` 引入了一个 CDN 托管的第三方库：

```html
{% block include_js %}
  <script src="https://cdnjs.cloudflare.com/ajax/libs/html5sortable/0.13.3/html5sortable.min.js"></script>
{% endblock %}
```

`html5sortable` 是一个轻量的"让某个 HTML 列表支持鼠标拖拽调整顺序"的 JS 库，配合模块列表页新出现的 `{% block include_js %}` 钩子（写在 `<script>` 主逻辑块之前，专门用来引入外部 JS 库文件）。拖拽结束后触发 `sortupdate` 事件：

```javascript
sortable('#modules', {forcePlaceholderSize: true, placeholderClass: 'placeholder'})[0]
  .addEventListener('sortupdate', function(e) {
    modulesOrder = {};
    document.querySelectorAll('#modules li').forEach(function (module, index) {
      modulesOrder[module.dataset.id] = index;
      module.querySelector('.order').innerHTML = index + 1;
    });
    options['body'] = JSON.stringify(modulesOrder);
    fetch(moduleOrderUrl, options)
  });
```

拖完之后，遍历 DOM 里当前的 `<li>` 顺序，用 `data-id`（对应数据库主键）和当前下标拼出一个 `{模块ID: 新顺序}` 的字典，`JSON.stringify` 之后通过 `fetch()` POST 给后端。后端接收方：

```python
class ModuleOrderView(CsrfExemptMixin, JsonRequestResponseMixin, View):
    def post(self, request):
        for id, order in self.request_json.items():
            Module.objects.filter(id=id, course__owner=request.user).update(order=order)
        return self.render_json_response({'saved': 'OK'})
```

`JsonRequestResponseMixin`（`django-braces` 提供）自动把请求体 JSON 解析成 `self.request_json` 字典，`self.render_json_response(...)` 自动包成 `JsonResponse` 返回——省去手动 `json.loads(request.body)`/`JsonResponse(...)` 这些样板代码。视图本身逻辑很简单：遍历前端传来的 `{id: order}` 映射，逐条用 `.filter(id=id, course__owner=request.user).update(order=order)` 更新——**同样带上了 `course__owner=request.user` 这个过滤条件**，即便攻击者伪造请求体里的 `id`，也只能改动自己名下课程的模块顺序，不会影响别人的数据。`Content` 的排序（`ContentOrderView`）是完全对称的实现。

### 一处值得关注的 CSRF 处理

`ModuleOrderView`/`ContentOrderView` 都用了 `CsrfExemptMixin`（`django-braces` 提供，效果等同于给这个视图整体加了 `@csrf_exempt`），而前端 `fetch()` 调用里也**没有设置任何 CSRF token 请求头**。回顾 Bookmarks 项目 Chapter06/Chapter07 的 AJAX 交互（点赞、关注）——那些视图都是老老实实走 Django 默认的 CSRF 保护，前端通过 `js-cookie` 读取 `csrftoken` cookie 并塞进请求头，而这里选择了完全跳过 CSRF 校验。这意味着：如果一个已登录的用户被诱导访问了一个恶意页面，那个恶意页面上的 JS 完全可以对着 `/course/module/order/` 发起一次跨站 POST 请求（浏览器会自动带上该用户的 session cookie），后端因为跳过了 CSRF 校验会照单全收——**这是一处真实的 CSRF 隐患**（虽然由于视图内部用 `course__owner=request.user` 做了归属过滤，攻击者能造成的破坏被限制在"打乱受害者自己课程模块的顺序"，达不到跨用户篡改数据的程度，但这依然是一次未经用户本意确认的写操作被伪造执行）。更稳妥的做法是让前端在 `fetch()` 里带上 CSRF token（就像 Chapter06 那样），而不是直接豁免整个视图的 CSRF 保护。

## 六、一处需要留意的回退：登出改回了 GET 链接

对比 Chapter12 的 `base.html`：

```diff
-  <li>
-    <form action="{% url "logout" %}" method="post">
-      <button type="submit">Sign out</button>
-      {% csrf_token %}
-    </form>
-  </li>
+  <li><a href="{% url "logout" %}">Sign out</a></li>
```

Chapter12 一开始就采用了 POST 表单来触发登出（Django 5 的 CSRF 加固要求登出这种会改变状态的操作不能用 GET 链接触发，避免类似"一个 `<img src="/logout/">` 就能让第三方页面偷偷把受害者登出"这类隐患——这一点在 Bookmarks 项目 Chapter04 教学里详细讲过），但 Chapter13 的 `base.html` **把这处改回了普通的 `<a href>` 链接**，等于是撤销了这个安全实践。这是继续深读代码时又一个值得记录的"书本示例代码内部不完全一致"的例子——多个项目、多个章节由不同上下文推进时，个别已经改对的细节偶尔会在后续修改里被无意间打回原样。

## 七、端到端流程

1. 讲师登录后台，`ManageCourseListView` 显示"我的课程"列表（`OwnerMixin` 保证只看到自己的）。
2. 点击"Create new course" → `CourseCreateView`（自动记录 `owner`）→ 保存成功跳回列表。
3. 点击"Edit modules" → `CourseModuleUpdateView` 用 `ModuleFormSet` 一次性增删改这门课下的所有模块。
4. 进入某个模块的内容管理页（`ModuleContentListView`）→ 点击"Text"/"Image"/"Video"/"File" 中的一种 → `ContentCreateUpdateView` 动态解析出对应模型、动态生成表单 → 保存后既存了具体内容对象，也建了一条 `Content` 索引记录挂到当前模块下。
5. 在模块列表或内容列表页直接拖拽调整顺序 → `html5sortable` 触发 `sortupdate` → `fetch()` 把新顺序 POST 给 `ModuleOrderView`/`ContentOrderView` → 后端批量 `update(order=...)`，`OrderField` 在 Chapter12 定义的"新建时自动排到最后"只是初始默认值，真正的顺序调整靠这里的拖拽接口完成。

## 八、Chapter12 → Chapter13 变化小结

| 方面 | Chapter12 | Chapter13 |
|---|---|---|
| 视图 | 无（仅模型/admin） | 完整 CBV 体系：通用编辑视图 + 手写 `View` 子类混用 |
| 权限控制 | 无 | `LoginRequiredMixin` + `PermissionRequiredMixin` + 自定义 `OwnerMixin`/`OwnerEditMixin` |
| 多态内容编辑 | 仅数据模型（`GenericForeignKey`） | `apps.get_model()` + `modelform_factory()` 运行时动态解析/生成，一个视图服务四种模型 |
| 排序能力 | 仅 `OrderField` 提供默认排序值 | 拖拽 UI（`html5sortable`）+ AJAX 接口批量更新 `order` |
| 登出方式 | POST 表单（CSRF 安全） | 回退为 GET 链接（**安全回退**） |
| CSRF 处理 | 未涉及 AJAX | 排序接口用 `CsrfExemptMixin` 完全跳过校验（**新增隐患**） |
