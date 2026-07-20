# Chapter06 教学笔记：Bookmarks 项目 —— 图片收藏、书签工具与 AJAX 交互

## 一、本章定位

Chapter05 的账号体系到这里正式派上用场：新增 `images` 应用，让登录用户能"收藏图片"，配套三件事——服务端下载存图、缩略图生成、无刷新的点赞和无限滚动。项目层面新增了两个依赖：

```diff
+ requests==2.31.0
+ easy-thumbnails==2.8.5
```

`requests` 用来在服务端下载用户提交的图片 URL；`easy-thumbnails` 提供模板标签 `{% thumbnail %}`，按需生成不同尺寸的缩略图并缓存。

## 二、数据模型：`Image`

```python
class Image(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, related_name='images_created', on_delete=models.CASCADE)
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200, blank=True)
    url = models.URLField(max_length=2000)
    image = models.ImageField(upload_to='images/%Y/%m/%d/')
    description = models.TextField(blank=True)
    created = models.DateTimeField(auto_now_add=True)
    users_like = models.ManyToManyField(settings.AUTH_USER_MODEL, related_name='images_liked', blank=True)

    class Meta:
        indexes = [models.Index(fields=['-created'])]
        ordering = ['-created']

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)

    def get_absolute_url(self):
        return reverse('images:detail', args=[self.id, self.slug])
```

几个设计点：

- `url` 字段存的是**图片的原始来源地址**（别的网站上那张图的 URL），`image` 字段才是下载下来存在自己服务器上的文件——两者都保留，一个当溯源信息，一个是真正展示用的本地资源。
- `users_like` 反向 `related_name='images_liked'`，配合 `user` 字段的 `related_name='images_created'`——一个 User 既能通过 `user.images_created` 拿到自己上传的图，也能通过 `user.images_liked` 拿到自己点赞过的图，两个方向的反查语义都起了清晰的名字，不会和默认的 `image_set` 混淆。
- `save()` 里自动补 slug——这和 Blog 项目 `Post` 模型如出一辙，但注意这里的 `slug` **没有** `unique_for_date` 或唯一约束，纯粹是为了 URL 好看，真正定位一张图靠的是 URL 里的 `id`（见 `get_absolute_url` 和 `image_detail` 视图），slug 只是附加的可读信息，写错了也不影响功能。
- `get_absolute_url` 里的 `'images:detail'` 使用了 app 命名空间——和 Blog 项目的 `'blog:post_detail'` 是同一套约定，`images/urls.py` 里 `app_name = 'images'` 对应上了。

## 三、`forms.py`——服务端下载图片这个核心动作

```python
class ImageCreateForm(forms.ModelForm):
    class Meta:
        model = Image
        fields = ['title', 'url', 'description']
        widgets = {'url': forms.HiddenInput}

    def clean_url(self):
        url = self.cleaned_data['url']
        valid_extensions = ['jpg', 'jpeg', 'png']
        extension = url.rsplit('.', 1)[1].lower()
        if extension not in valid_extensions:
            raise forms.ValidationError('The given URL does not match valid image extensions.')
        return url

    def save(self, force_insert=False, force_update=False, commit=True):
        image = super().save(commit=False)
        image_url = self.cleaned_data['url']
        name = slugify(image.title)
        extension = image_url.rsplit('.', 1)[1].lower()
        image_name = f'{name}.{extension}'
        response = requests.get(image_url)
        image.image.save(image_name, ContentFile(response.content), save=False)
        if commit:
            image.save()
        return image
```

- `widgets = {'url': forms.HiddenInput}`——`url` 字段用户不需要手填，是书签工具自动带过来的，表单里只是"隐藏地传下去"，界面上用户只填标题和描述。
- `clean_url` 只按**文件名后缀**粗略判断是不是图片——这是一个偏弱的校验，后面会讲它的风险。
- `save()` 重写：先拿到未落库的 `image` 对象，然后 `requests.get(image_url)` **服务端主动去抓取**这个 URL 的内容，把返回的字节流通过 `ContentFile` 包装后存进 `ImageField`。这是"用户提交一个链接，服务器帮你下载下来存到自己这"这类功能的标准写法（图床、附件抓取、RSS 抓图都是这个模式）。

> **安全性提醒**：这段代码有真实的 SSRF（服务端请求伪造）风险——`clean_url` 只查了字符串后缀是不是 `.jpg/.jpeg/.png`，并没有校验这个 URL 解析出的 IP 是不是内网地址（比如提交 `http://169.254.169.254/latest/meta-data/xxx.jpg` 或指向内网服务的地址），服务端会照样发起请求；`requests.get` 也没有限制响应体大小、没有设置超时，一个指向巨大文件或者慢速服务的 URL 可以拖垮下载这个请求的 worker。真实项目里这里至少要加：URL 解析后校验目标 IP 不在内网网段、`requests.get(..., timeout=5, stream=True)` 配合读取字节数上限、以及下载完之后再校验一次真实的图片 Content-Type/文件头，而不是只信任 URL 后缀字符串。

## 四、`views.py`

```python
@login_required
def image_create(request):
    if request.method == 'POST':
        form = ImageCreateForm(data=request.POST)
        if form.is_valid():
            new_image = form.save(commit=False)
            new_image.user = request.user
            new_image.save()
            messages.success(request, 'Image added successfully')
            return redirect(new_image.get_absolute_url())
    else:
        form = ImageCreateForm(data=request.GET)
    return render(request, 'images/image/create.html', {'section': 'images', 'form': form})
```

同一个视图承担两种输入来源：**GET** 时表单数据来自书签工具跳转过来的 URL 查询参数（预填标题和图片地址，用户只需要确认/补充描述后提交）；**POST** 时才是真正的表单提交。提交成功后 `redirect()` 到详情页——经典的 **PRG（Post/Redirect/Get）** 模式，避免用户刷新页面时重复提交表单。

```python
def image_detail(request, id, slug):
    image = get_object_or_404(Image, id=id, slug=slug)
    return render(request, 'images/image/detail.html', {'section': 'images', 'image': image})
```

注意这个视图**没有** `@login_required`——收藏图片需要登录，但**浏览**图片详情页是公开的，任何人都能看。这是刻意的产品设计：站点内容公开展示，互动行为才要求身份。

```python
@login_required
@require_POST
def image_like(request):
    image_id = request.POST.get('id')
    action = request.POST.get('action')
    if image_id and action:
        try:
            image = Image.objects.get(id=image_id)
            if action == 'like':
                image.users_like.add(request.user)
            else:
                image.users_like.remove(request.user)
            return JsonResponse({'status': 'ok'})
        except Image.DoesNotExist:
            pass
    return JsonResponse({'status': 'error'})
```

这是本书**第一个纯 AJAX 接口**——不返回 HTML，返回 `JsonResponse`。`users_like.add()`/`.remove()` 直接操作 M2M 中间表，一次调用搞定关联关系的增删，不需要手动查中间表判断是否已存在（`add()` 本身是幂等的，重复 add 不会报错）。

```python
@login_required
def image_list(request):
    images = Image.objects.all()
    paginator = Paginator(images, 8)
    page = request.GET.get('page')
    images_only = request.GET.get('images_only')
    try:
        images = paginator.page(page)
    except PageNotAnInteger:
        images = paginator.page(1)
    except EmptyPage:
        if images_only:
            return HttpResponse('')
        images = paginator.page(paginator.num_pages)
    if images_only:
        return render(request, 'images/image/list_images.html', {'section': 'images', 'images': images})
    return render(request, 'images/image/list.html', {'section': 'images', 'images': images})
```

分页逻辑还是 Blog Ch02 那套手写三异常处理，但多了一个 `images_only` 分支——**同一个视图**根据这个参数决定返回"完整页面"还是"只有图片列表这一小段 HTML 片段"。这是给下面讲的无限滚动服务的：首次访问走完整页，滚动到底部后前端用 `fetch('?images_only=1&page=2')` 只要那一小块 HTML，直接拼接到页面里，不用整页刷新。`EmptyPage` 分支里对 `images_only` 请求直接返回空字符串——前端用"收到空响应"当作"没有更多图片了"的信号，停止继续发请求。

## 五、书签工具（Bookmarklet）——本章真正的巧思

`dashboard.html` 里那个"Bookmark it"按钮不是普通链接：

```html
<a href="javascript:{% include "bookmarklet_launcher.js" %}" class="button">Bookmark it</a>
```

`href` 直接是一段 `javascript:` 协议的代码——这就是"书签工具"的本质：把一段 JS 存成浏览器书签，点击时不是跳转页面，而是在**当前所在的任意网站**上执行这段脚本。`bookmarklet_launcher.js` 做的事很简单：

```js
if(!window.bookmarklet) {
    bookmarklet_js = document.body.appendChild(document.createElement('script'));
    bookmarklet_js.src = '//127.0.0.1:8000/static/js/bookmarklet.js?r=' + Math.random();
    window.bookmarklet = true;
}
else {
    bookmarkletLaunch();
}
```

第一次点击：往当前页面（不管这是哪个网站）动态插入一个 `<script>` 标签，去加载**你自己站点**的 `bookmarklet.js`；`?r=随机数` 是缓存穿透（cache busting），避免浏览器用旧版本的脚本缓存。`window.bookmarklet` 标记位保证同一个页面上重复点书签不会重复插入脚本。

真正的逻辑在 `bookmarklet.js` 的 `bookmarkletLaunch()` 里：

1. 往当前页面注入一个悬浮框（`bookmarklet.css` 定义样式，`z-index` 拉到最大保证盖在最上层）。
2. `document.querySelectorAll('img[src$=".jpg"], ...')` 扫描当前页面 DOM 里所有图片标签，过滤掉太小的（`naturalWidth/Height >= 250`，避免把图标、装饰小图也列进来）。
3. 把符合条件的图片缩略展示在悬浮框里，用户点选一张。
4. 点选后 `window.open(siteUrl + 'images/create/?url=' + encodeURIComponent(imageSelected.src) + '&title=' + encodeURIComponent(document.title), '_blank')`——**新开一个标签页**跳回你自己的 Bookmarks 站点，带着选中图片的地址和当前页标题作为 GET 参数，直接命中前面讲的 `image_create` 视图的 GET 分支，表单被自动预填好。

这一整套设计的巧妙之处在于：**完全不需要跨域权限、不需要浏览器扩展**，纯靠"往当前页面注入脚本 + 新开标签页回到自己域名"这两个动作绕开了跨站的限制。

## 六、无刷新交互：AJAX 的两个场景

`base.html` 新增了公共基础设施：

```html
<script src="//cdn.jsdelivr.net/npm/js-cookie@3.0.5/dist/js.cookie.min.js"></script>
<script>
  const csrftoken = Cookies.get('csrftoken');
  document.addEventListener('DOMContentLoaded', (event) => {
    {% block domready %}
    {% endblock %}
  })
</script>
```

因为 `fetch()` 发起的请求不会像 `<form>` 那样自动带上 `{% csrf_token %}`，必须手动从 cookie 里读出 CSRF token 塞进请求头——这是 Django 官方文档里 AJAX 场景处理 CSRF 的标准做法。`{% block domready %}` 是一个约定：任何子模板想在页面加载完后跑一段 JS，就重写这个 block，不用各自重复写 `DOMContentLoaded` 监听。

**点赞（`detail.html`）**：

```js
fetch(url, options).then(response => response.json()).then(data => {
  if (data['status'] === 'ok') {
    var action = previousAction === 'like' ? 'unlike' : 'like';
    likeButton.dataset.action = action;
    likeButton.innerHTML = action;
    likeCount.innerHTML = previousAction === 'like' ? totalLikes + 1 : totalLikes - 1;
  }
})
```

请求成功后**直接在前端更新按钮文字和计数**，不重新渲染整个页面，也不重新发请求去查最新点赞数——这是乐观更新的思路：既然后端返回了 `ok`，前端就相信操作成功了，直接同步 UI 状态。

**无限滚动（`list.html`）**：

```js
window.addEventListener('scroll', function(e) {
  var margin = document.body.clientHeight - window.innerHeight - 200;
  if(window.pageYOffset > margin && !emptyPage && !blockRequest) {
    blockRequest = true;
    page += 1;
    fetch('?images_only=1&page=' + page).then(...).then(html => {
      if (html === '') { emptyPage = true; }
      else { imageList.insertAdjacentHTML('beforeEnd', html); blockRequest = false; }
    })
  }
});
const scrollEvent = new Event('scroll');
window.dispatchEvent(scrollEvent);
```

`blockRequest` 防止用户疯狂滚动时并发发出多个重复请求；`emptyPage` 一旦为 true 就永久停止继续请求（对应后端 `image_list` 视图 `EmptyPage` 分支返回空字符串）；最后手动 `dispatchEvent` 触发一次 `scroll` 事件，是为了处理"首屏内容本来就没填满一屏，用户根本没法产生真实滚动"的边界情况，手动补一次检查。

## 七、缩略图：`easy-thumbnails`

```html
<!-- detail.html：按宽度等比缩放 -->
{% thumbnail image.image 300x0 %}

<!-- list_images.html：固定尺寸智能裁剪 -->
{% thumbnail image.image 300x300 crop="smart" as im %}
```

`300x0` 里的 `0` 表示"只限制宽度，高度等比缩放"；`crop="smart"` 则是裁成固定的 300×300 正方形，用智能裁剪算法（分析图片内容找视觉重心）而不是简单居中裁剪。缩略图不是上传时预生成的，是**第一次被模板请求时按需生成并缓存到磁盘**，后续同尺寸请求直接读缓存。

## 八、Chapter05 → Chapter06 变化小结

| 方面 | Chapter05 | Chapter06 |
|---|---|---|
| 新增 app | 无 | `images`（图片收藏、点赞、无限滚动） |
| `account.Profile.photo` | `CharField`（存 URL） | 改回 `ImageField`（真实文件上传）—— Ch05 那次迁移不同步的问题在这里被"顺带"解决了 |
| 前端 | 纯服务端渲染 | 首次引入 `fetch` AJAX（点赞、无限滚动）+ 书签工具（跨站注入 JS） |
| 图片处理 | 无 | `requests` 服务端下载 + `easy-thumbnails` 动态缩略图 |
| 安全关注点 | `clean_email` NameError | `ImageCreateForm.save()` 的 SSRF/资源耗尽风险（见上方提醒） |
