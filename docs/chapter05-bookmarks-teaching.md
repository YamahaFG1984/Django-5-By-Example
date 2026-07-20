# Chapter05 教学笔记：Bookmarks 项目 —— 多种登录方式与消息反馈

## 一、本章定位

Chapter05 不建新项目，是在 Chapter04 的 `bookmarks/account` 上继续加功能，主题是"**让用户能用不止一种方式登录**"：

1. 用邮箱代替用户名登录（自定义认证后端）
2. 用 Google 账号一键登录/注册（`social-auth-app-django`）
3. 编辑资料成功/失败给出提示（`django.contrib.messages`，Ch04 的 CSS 里其实早就预留了 `.messages` 相关样式，直到本章才真正用上）

## 二、新增依赖

```diff
+ python-decouple==3.8
+ social-auth-app-django==5.4.0
+ django-extensions==3.2.3
+ werkzeug==3.0.2
+ pyOpenSSL==24.1.0
```

- `python-decouple`：读环境变量，本章第一次在 Bookmarks 项目里用它管理 Google OAuth 的 Key/Secret（不落进代码库）。
- `social-auth-app-django`：Python Social Auth 的 Django 封装，处理 OAuth2 授权码交换、token 存取、pipeline 执行的整套流程。
- `django-extensions` + `werkzeug` + `pyOpenSSL`：这三个通常是一组，`django-extensions` 提供 `runserver_plus`（带交互式调试器的开发服务器），`werkzeug` 是它的调试器依赖，`pyOpenSSL` 让 `runserver_plus` 能生成自签名证书跑 HTTPS——OAuth 回调调试时经常需要本地也走 HTTPS，这组包就是为此配的。

## 三、认证后端机制：`AUTHENTICATION_BACKENDS`

```python
AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    'account.authentication.EmailAuthBackend',
    'social_core.backends.google.GoogleOAuth2',
]
```

这是本章的核心机制：Django 调用 `authenticate()` 时，会**按顺序**依次尝试列表里每一个后端的 `authenticate()` 方法，谁先返回非 `None` 的 User 就用谁，全部失败才算登录失败。三个后端分工：

- `ModelBackend`：Django 默认的，按 `username` + 密码校验。
- `EmailAuthBackend`（自定义）：把登录表单里的 `username` 字段当邮箱去查。
- `GoogleOAuth2`：不是给普通表单用的，是 Python Social Auth 内部调用的，登录入口是点击"Sign in with Google"链接触发的整套 OAuth 流程，不是走 `authenticate(username=, password=)`。

这样设计的好处是：**登录表单完全不用改**，用户在同一个用户名输入框里填用户名或邮箱都能登录，Django 会自动依次试完三个后端。

### `account/authentication.py`

```python
class EmailAuthBackend:
    def authenticate(self, request, username=None, password=None):
        try:
            user = User.objects.get(email=username)
            if user.check_password(password):
                return user
            return None
        except (User.DoesNotExist, User.MultipleObjectsReturned):
            return None

    def get_user(self, user_id):
        try:
            return User.objects.get(pk=user_id)
        except User.DoesNotExist:
            return None
```

两个方法都必须实现，这是 Django 认证后端的接口约定：`authenticate()` 负责登录时校验；`get_user()` 负责**每次请求**从 session 里的 user_id 反查用户对象（`request.user` 背后就是靠这个方法工作的）。

这里有个**值得注意的设计缺陷**：`except (User.DoesNotExist, User.MultipleObjectsReturned)` 里捕获了 `MultipleObjectsReturned`，说明作者知道"同一个邮箱可能对应多个 User"这种情况会发生——但真正的解法应该是在 `User.email` 字段上加 `unique=True` 约束，从数据层杜绝重复邮箱，而不是在查询时用 try/except 把这种本不该出现的情况"悄悄吞掉"（吞掉之后，这个用户会拿到"登录失败"提示，却不知道真正原因是自己的邮箱在系统里重复了）。默认 `auth.User.email` 字段本身确实不带 `unique=True`，这是 Django 自带的已知设计（历史遗留），本章新增的 `clean_email` 校验（下面会讲）是想在应用层补上这个约束，但只能防住"以后新注册/新编辑"的重复，堵不住已经存在的历史脏数据。

## 四、真正的 Bug 1：`clean_email` 里的 `NameError`

```python
# account/forms.py
from django import forms
from django.contrib.auth import get_user_model
from .models import Profile
```

文件顶部只导入了 `get_user_model`，**没有导入 `User`**。但 `clean_email` 里写的是：

```python
class UserRegistrationForm(forms.ModelForm):
    ...
    def clean_email(self):
        data = self.cleaned_data['email']
        if User.objects.filter(email=data).exists():   # ← User 未定义
            raise forms.ValidationError('Email already in use.')
        return data
```

`UserEditForm.clean_email` 同样用了裸的 `User`。这是**书里示例代码的真实 bug**：只要注册或编辑资料时填了邮箱字段，`clean_email` 就会执行到这一行，直接抛 `NameError: name 'User' is not defined`，而不是预期的表单校验错误。修复很简单，把 `Meta.model = get_user_model()` 改成显式引用即可：

```python
from django.contrib.auth import get_user_model
User = get_user_model()
```

或者干脆把两处 `User.objects` 换成 `get_user_model().objects`。这是继续深读代码时值得留意的——教学书的示例代码不代表没有 bug，读的时候自己跑一遍表单校验分支就能发现。

## 五、真正的 Bug/不一致 2：`photo` 字段类型变了，迁移没跟上

```python
# account/models.py
class Profile(models.Model):
    ...
    # photo = models.ImageField(upload_to='users/%Y/%m/%d/', blank=True)
    photo = models.CharField(max_length=250, blank=True)
```

从 `ImageField` 改成了 `CharField`——原因很合理：Google 登录拿到的头像是**一个图片 URL**，不是一个可以存进 `MEDIA_ROOT` 的上传文件，`social_core.pipeline.user.user_details` 这一步会把 Google 返回的头像地址直接写进 `profile.photo`，用字符串字段存 URL 比用 `ImageField` 合适。

但 `account/migrations/0001_initial.py` 里 `photo` 字段**仍然是** `models.ImageField(...)`——模型改了，迁移文件没有同步重新生成。如果把这个仓库当一个真实项目 clone 下来直接 `makemigrations`，Django 会告诉你 `Profile.photo` 字段有未应用的变更。这也是"深读代码"该养成的习惯：模型定义和迁移文件必须对应，看到注释掉的字段定义旁边留着旧类型，就要留意背后是不是有没跟上的迁移。

## 六、Social Auth Pipeline

```python
SOCIAL_AUTH_PIPELINE = [
    'social_core.pipeline.social_auth.social_details',
    'social_core.pipeline.social_auth.social_uid',
    'social_core.pipeline.social_auth.auth_allowed',
    'social_core.pipeline.social_auth.social_user',
    'social_core.pipeline.user.get_username',
    'social_core.pipeline.user.create_user',
    'account.authentication.create_profile',      # ← 插入的自定义步骤
    'social_core.pipeline.social_auth.associate_user',
    'social_core.pipeline.social_auth.load_extra_data',
    'social_core.pipeline.user.user_details',
]
```

这是一条**流水线**，Google 回调回来之后，会按顺序执行这十步：拿到第三方资料 → 拿到第三方唯一 ID → 检查是否允许授权 → 查是否已有关联账号 → 生成用户名 → **创建 User** → …

看到 `create_user` 和 `associate_user` 中间插了一步 `account.authentication.create_profile`：

```python
def create_profile(backend, user, *args, **kwargs):
    Profile.objects.get_or_create(user=user)
```

这一步的作用和 `register` 视图里手动 `Profile.objects.create(user=new_user)` 完全对应——**普通注册和社交登录是两条完全独立的用户创建路径**，如果不在 pipeline 里插这一步，通过 Google 登录创建的新用户就没有 Profile，后面任何访问 `request.user.profile` 的代码（比如 `edit` 视图）都会报错。这里用 `get_or_create` 而不是 `create`，是因为社交登录 pipeline 在"已有账号再次登录"时也会走到这一步，用 `get_or_create` 保证不会对老用户重复建 Profile 报唯一约束冲突。

## 七、Messages 框架首次登场

```python
# account/views.py
@login_required
def edit(request):
    if request.method == 'POST':
        ...
        if user_form.is_valid() and profile_form.is_valid():
            user_form.save()
            profile_form.save()
            messages.success(request, 'Profile updated successfully')
        else:
            messages.error(request, 'Error updating your profile')
```

```html
<!-- base.html -->
{% if messages %}
  <ul class="messages">
    {% for message in messages %}
      <li class="{{ message.tags }}">
        {{ message|safe }}
        <a href="#" class="close">x</a>
      </li>
    {% endfor %}
  </ul>
{% endif %}
```

`messages.success`/`messages.error` 把提示信息存进 session，下一次渲染任意页面时 `{% if messages %}` 都能取到并显示一次（显示后自动清空）——这是 Django "一次性横幅提示"的标准做法，本章第一次用上，而 `message.tags` 输出的正是 `success`/`error` 这个级别名，直接对应 Ch04 时 `base.css` 里就写好的 `ul.messages li.success`/`li.error` 样式。之前几章那段 CSS 一直是"预留但没用上"的状态，到这里终于闭环。

## 八、URL 层新增

```python
# bookmarks/urls.py
path('social-auth/', include('social_django.urls', namespace='social')),
```

配合模板里的 `{% url "social:begin" "google-oauth2" %}`——点击"Sign in with Google"实际访问 `/social-auth/login/google-oauth2/`，这个 URL 完全由 `social_django` 内部处理（跳转 Google 授权页、接收回调、换 token、跑 pipeline），业务代码里不需要写一行视图。

## 九、端到端流程对比

**普通邮箱/用户名注册登录**（有 bug 未修复的情况下）：填注册表单 → 一旦邮箱字段被 `clean_email` 校验 → **直接 500（NameError）**，走不通——这也是为什么要把这个 bug 单独拎出来讲，它不是"边缘情况"，是注册这条主链路当前就跑不通。

**Google 一键登录**（全新用户）：点击"Sign in with Google" → 跳 Google 同意页 → 授权后回调到 `/social-auth/complete/google-oauth2/` → pipeline 十步依次执行，途中创建 User + Profile（`photo` 字段写入 Google 头像 URL）→ 自动 `login()` → 跳转 `LOGIN_REDIRECT_URL`（dashboard）。全程不经过 `account/views.py` 里任何一个视图。

**编辑资料**：提交成功/失败都会在下一次页面渲染时看到顶部的绿色/红色提示条——这是本章唯一新增、且**真正跑得通**的用户可见变化。

## 十、Chapter04 → Chapter05 变化小结

| 方面 | Chapter04 | Chapter05 |
|---|---|---|
| 登录方式 | 仅用户名+密码 | 用户名/邮箱+密码（多后端）、Google 一键登录 |
| `AUTHENTICATION_BACKENDS` | 未配置（默认仅 ModelBackend） | 三个后端顺序尝试 |
| `Profile.photo` | `ImageField`（真实文件上传） | `CharField`（存 URL，但迁移文件未同步，是个遗留问题） |
| 操作反馈 | 无 | `messages` 框架，成功/失败提示 |
| 新用户创建入口 | 仅 `register` 一条路径 | `register` 视图 + 社交登录 pipeline 两条路径，都要保证建 Profile |
| 已知代码问题 | 无 | `clean_email` 缺 `User` 导入（NameError）；`photo` 迁移与模型不一致 |
