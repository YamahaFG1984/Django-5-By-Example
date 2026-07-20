# Chapter04 教学笔记：Bookmarks 项目 —— 用户认证体系

## 一、本章定位

前三章的 Blog 项目到 Chapter03 就告一段落了。Chapter04 开了一个新目录 `Chapter04/bookmarks/`，是全新的 Django 项目 `bookmarks`，里面只有一个 app：`account`。这一章的唯一目标是：**注册、登录、登出、编辑资料、改密码、重置密码**——一套完整的账号体系,后续章节（图片收藏、关注、点赞、动态流）都会长在这个账号体系之上。

值得注意的是，这个项目里 `base.css` 已经提前写好了 `.image-*`、`.people-*`、`.action`、`.social` 等还没用到的样式类——说明这份 CSS 是给整个 Bookmarks 系列章节共用的"半成品"，本章只用到其中 header/表单相关部分。

## 二、目录结构

```
Chapter04/bookmarks/
├── bookmarks/              项目配置包
│   ├── settings.py
│   ├── urls.py
│   ├── wsgi.py / asgi.py
├── account/                 唯一的 app
│   ├── models.py            Profile 模型
│   ├── forms.py             4 个表单
│   ├── views.py             登录/注册/仪表盘/编辑资料
│   ├── urls.py
│   ├── admin.py
│   ├── migrations/
│   ├── static/css/base.css
│   └── templates/
│       ├── base.html
│       ├── account/         自定义模板（登录、注册、仪表盘、编辑）
│       └── registration/    覆盖 Django 内建 auth 视图用的模板
```

## 三、配置层：`settings.py`

```python
INSTALLED_APPS = [
    'account.apps.AccountConfig',
    'django.contrib.admin',
    'django.contrib.auth',
    ...
]
```

`django.contrib.auth` 是 Django 默认就带的，本章第一次真正把它用起来。三个关键配置：

```python
LOGIN_REDIRECT_URL = 'dashboard'
LOGIN_URL = 'login'
LOGOUT_URL = 'logout'
```

这三项是 Django 认证体系的"路由约定"：`@login_required` 装饰器在用户未登录时会跳到 `LOGIN_URL`；登录成功后默认跳到 `LOGIN_REDIRECT_URL`。

```python
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
```

重置密码功能需要发邮件，开发阶段用 console backend——邮件内容直接打印在终端，不需要真的配 SMTP，这是本地调试密码重置流程最省事的方式（对比 Blog 项目里 Ch02 就直接配了真实 Gmail SMTP，这里是更轻量的开发期做法）。

```python
MEDIA_URL = 'media/'
MEDIA_ROOT = BASE_DIR / 'media'
```

配合 `Profile.photo` 这个 `ImageField`。项目 `urls.py` 里对应加了：

```python
if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

只有 `DEBUG=True` 时 Django 才会帮你直接托管媒体文件——生产环境这段代码根本不会生效，媒体文件必须交给 Nginx/对象存储，这是标准做法。

## 四、数据模型层：`Profile`

```python
class Profile(models.Model):
    user = models.OneToOneField(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    date_of_birth = models.DateField(blank=True, null=True)
    photo = models.ImageField(upload_to='users/%Y/%m/%d/', blank=True)
```

这里有个**很值得拿出来对照的设计点**：这个项目从头到尾都在用 Django **默认的 `auth.User`**，然后额外挂一张 `Profile` 表存"User 模型里没有的字段"（生日、头像）。这正是《Two Scoops of Django》里最著名的一条建议——"项目起步就该用自定义 User 模型，避免以后拆 Profile 表、多一次 join"——**这本书的示例代码本身就没有遵循这条建议**，用的是最传统的"默认 User + OneToOne Profile"模式。这不是错，只是教学书为了让读者聚焦在"如何用 auth 体系"上，刻意选择了更简单直接的路径；但如果把这个项目当真实项目来改造，第一件该做的事就是先换成自定义 User 模型，再往上加字段，而不是继续叠 Profile 表。

`settings.AUTH_USER_MODEL` 而不是直接 `import User` 硬编码——这是 Django 的标准写法，即使当前用的是默认 User，也要通过这个配置项引用，方便未来切换。

## 五、Admin

```python
@admin.register(Profile)
class ProfileAdmin(admin.ModelAdmin):
    list_display = ['user', 'date_of_birth', 'photo']
    raw_id_fields = ['user']
```

`raw_id_fields = ['user']` 把 `user` 外键从下拉框换成一个带搜索弹窗的输入框——当 `User` 表数据量大时，下拉框会卡到没法用，这是 Django admin 里处理大表外键的标准手法（Blog 项目里 `PostAdmin` 对 `author` 字段也用了同样的技巧）。

## 六、`views.py` 逐个讲解

### `user_login`——手写的登录视图

```python
def user_login(request):
    if request.method == 'POST':
        form = LoginForm(request.POST)
        if form.is_valid():
            cd = form.cleaned_data
            user = authenticate(request, username=cd['username'], password=cd['password'])
            if user is not None:
                if user.is_active:
                    login(request, user)
                    return HttpResponse('Authenticated successfully')
                else:
                    return HttpResponse('Disabled account')
            else:
                return HttpResponse('Invalid login')
```

这是书里故意先带你手写一遍登录逻辑，讲清楚 `authenticate()`（校验用户名密码，返回 User 对象或 None）和 `login()`（把用户写进 session）两步分离的原理。但看 `urls.py` 就会发现——**这个视图在最终版本里被注释掉了**，被 Django 内建的 `LoginView` 取代。这是很典型的教学设计：先带你理解原理，再告诉你"其实框架已经内置了，不用自己写"。

### `dashboard`——最简单的登录后主页

```python
@login_required
def dashboard(request):
    return render(request, 'account/dashboard.html', {'section': 'dashboard'})
```

`section` 这个变量传给模板是为了让导航栏知道当前高亮哪个菜单项（`base.html` 里 `{% if section == "dashboard" %}class="selected"{% endif %}`），这是后续几章 Images/People 页面都会复用的约定。

### `register`——注册 + 自动建 Profile

```python
def register(request):
    if request.method == 'POST':
        user_form = UserRegistrationForm(request.POST)
        if user_form.is_valid():
            new_user = user_form.save(commit=False)
            new_user.set_password(user_form.cleaned_data['password'])
            new_user.save()
            Profile.objects.create(user=new_user)
            return render(request, 'account/register_done.html', {'new_user': new_user})
```

三个关键点：

1. `save(commit=False)` —— 和 Blog 项目 `post_comment` 视图里的手法一模一样：先拿到未落库的对象，插入额外逻辑，再手动保存。
2. `set_password()` —— **绝不能**直接把表单里的明文密码赋值给 `user.password` 再保存，必须用这个方法，它会用 Django 的密码哈希算法（PBKDF2 等）处理，明文密码永远不落库。
3. `Profile.objects.create(user=new_user)` —— User 保存成功后，紧接着手动创建对应的 Profile。这一步如果漏掉，后面所有访问 `request.user.profile` 的代码都会抛 `Profile.DoesNotExist`。这也是"User + Profile 两张表"模式的代价之一：多一步容易忘的手动关联，如果用自定义 User 模型直接把字段加进去，就不存在这个问题。

### `edit`——双表单模式

```python
@login_required
def edit(request):
    if request.method == 'POST':
        user_form = UserEditForm(instance=request.user, data=request.POST)
        profile_form = ProfileEditForm(instance=request.user.profile, data=request.POST, files=request.FILES)
        if user_form.is_valid() and profile_form.is_valid():
            user_form.save()
            profile_form.save()
```

一个页面、一次提交，同时更新两张表（`User` 和 `Profile`）——这是"User+Profile 分表"模式下必须处理的典型场景：两个 `ModelForm` 分别绑定各自的 `instance`，各自校验，都通过了才都保存。`files=request.FILES` 是因为 `ProfileEditForm` 里有 `photo` 这个文件字段，普通 `request.POST` 拿不到文件数据，必须单独传 `FILES`。

## 七、`urls.py`——从手写到内建的关键转变

```python
urlpatterns = [
    path('', include('django.contrib.auth.urls')),
    path('', views.dashboard, name='dashboard'),
    path('register/', views.register, name='register'),
    path('edit/', views.edit, name='edit'),
]
```

文件里那一大段被注释掉的代码（`LoginView`、`LogoutView`、`PasswordChangeView` 等逐个手写 path）就是最终这一行 `include('django.contrib.auth.urls')` 的展开版本——这一行等价于自动注册了这些命名 URL：

| URL name | 作用 |
|---|---|
| `login` / `logout` | 登录/登出 |
| `password_change` / `password_change_done` | 已登录用户修改密码 |
| `password_reset` / `password_reset_done` | 忘记密码，发重置邮件 |
| `password_reset_confirm` / `password_reset_complete` | 点击邮件里的链接设置新密码 |

**这是本章最重要的教学点**：Django 已经把"账号系统"里最繁琐、最容易出安全漏洞的部分（密码重置 token 生成校验、密码修改后 session 失效处理）都写好了，你只需要提供对应的模板文件（也就是 `templates/registration/` 下那一堆 html），不需要自己写视图逻辑。

## 八、`forms.py`

```python
class UserRegistrationForm(forms.ModelForm):
    password = forms.CharField(widget=forms.PasswordInput)
    password2 = forms.CharField(label='Repeat password', widget=forms.PasswordInput)

    class Meta:
        model = get_user_model()
        fields = ['username', 'first_name', 'email']

    def clean_password2(self):
        cd = self.cleaned_data
        if cd['password'] != cd['password2']:
            raise forms.ValidationError("Passwords don't match.")
        return cd['password2']
```

两个点：

- `get_user_model()` 而不是硬编码 `User`——和 model 里用 `settings.AUTH_USER_MODEL` 是同一个原则的两种写法（一个用在需要"类"的地方，一个用在需要"字符串引用"的地方）。
- `clean_password2` 是 Django Form 的字段级校验钩子，方法名规则是 `clean_<字段名>`，Django 会自动调用。两次密码不一致时在这里统一报错，比在视图里手写 if 判断更符合"表单负责校验"的分工。

`UserEditForm` 和 `ProfileEditForm` 就是把 `User`/`Profile` 各自能编辑的字段单独拆成两个 `ModelForm`，呼应上面 `edit` 视图的双表单模式。

## 九、模板层

`base.html` 里有个 Django 5 相关的细节：

```html
<form action="{% url "logout" %}" method="post">
  <button type="submit">Logout</button>
  {% csrf_token %}
</form>
```

登出用的是 **POST 表单**而不是一个 `<a href="{% url 'logout' %}">` 链接。这是因为新版 Django 的 `LogoutView` 从 GET 请求触发登出改成了要求 POST——原因是"点击一个链接就能让别人退出登录"本身是一个 CSRF 风险面（恶意页面可以嵌一个 `<img src="/logout/">` 让访客在不知情的情况下被登出），所以官方把它收紧为必须 POST + CSRF token。

`registration/` 目录下的模板都是**覆盖 Django 内建视图默认模板路径**的写法——Django 的 `LoginView` 默认找 `registration/login.html`，`PasswordResetView` 默认找 `registration/password_reset_form.html`，只要按这些约定路径放模板，视图逻辑完全不用碰。

## 十、端到端业务流程

**注册**：`register.html` 提交 → `UserRegistrationForm` 校验（用户名唯一性由 `ModelForm` 自动带出、两次密码一致性由 `clean_password2` 保证）→ 创建 User + Profile → 跳 `register_done.html` 提示去登录（注意：注册成功后**没有**自动登录，用户还要手动走一遍登录流程）。

**登录**：内建 `LoginView` 处理 POST，模板 `registration/login.html` 里那个隐藏的 `<input type="hidden" name="next">` 是配合"未登录访问需要登录的页面被重定向到登录页"场景——登录成功后会自动跳回原本想访问的页面，而不是每次都回到 dashboard。

**改密码**：已登录用户填旧密码+新密码两次 → `PasswordChangeView` 校验旧密码正确性 → 保存新密码的哈希 → 跳 `password_change_done.html`。

**重置密码**（忘记密码场景）：填邮箱 → `PasswordResetView` 生成一个基于用户信息和时间戳的一次性 token → 通过 `password_reset_email.html` 渲染邮件内容（开发环境直接打印到控制台）→ 用户点邮件里 `password_reset_confirm` 链接（带 `uidb64` + `token`）→ token 校验通过则显示设置新密码表单 → 提交后跳 `password_reset_complete.html`。这一整套 token 生成/校验逻辑全部是 Django 内建的，本章完全没有自己写一行相关代码。

## 十一、Docker/开发环境

`docker-compose.yml` 结构和 Blog 项目 Ch01/Ch02 时期一样，是 SQLite（本章 `settings.py` 里数据库还是 `sqlite3`，`requirements.txt` 也没有 `psycopg`）+ 简单的 `web_migrate`/`web_run` 两步启动，还没有引入 Postgres——说明账号体系这一章是刻意保持环境最简，把"接入 Postgres"这类基础设施变化留给后面讲图片/搜索的章节。
