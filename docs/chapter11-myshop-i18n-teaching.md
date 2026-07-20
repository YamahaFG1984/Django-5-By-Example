# Chapter11 教学笔记：myshop 项目 —— 国际化、本地化与多语言商品

Chapter11 是纯粹的"让商城支持多语言"这一章：UI 文案翻译（i18n）、地区相关表单校验（l10n，本章示例是美国邮编）、以及**商品/分类内容本身的多语言**（用 `django-parler` 让同一件商品能有英文、西班牙文两套 name/slug/description）。

## 一、新依赖与整体配置

```diff
+ django-rosetta==0.10.0
+ django-parler==2.3
+ django-localflavor==4.0
```

- **`django-rosetta`**：给 admin 后台加一个"翻译管理界面"，可以直接在浏览器里编辑 `.po` 翻译文件的每一条词条，不用手动打开文本文件或跑 `msgfmt`。
- **`django-parler`**：让 Django 模型的某些字段拥有"每种语言一份独立取值"的能力（本章用来翻译 `Category.name`/`Product.name` 等）。
- **`django-localflavor`**：提供各国/地区特化的表单字段（本章用它的 `USZipCodeField` 校验美国邮编格式）。

`settings.py`：

```python
from django.utils.translation import gettext_lazy as _
...
INSTALLED_APPS += ['localflavor', 'parler', 'rosetta']
MIDDLEWARE += ['django.middleware.locale.LocaleMiddleware']

LANGUAGE_CODE = 'en'
LANGUAGES = [
    ('en', _('English')),
    ('es', _('Spanish')),
]
LOCALE_PATHS = [BASE_DIR / 'locale']

# django-parler settings
PARLER_LANGUAGES = {
    None: ({'code': 'en'}, {'code': 'es'}),
    'default': {'fallback': 'en', 'hide_untranslated': False},
}
```

- `LocaleMiddleware` **必须**加进 `MIDDLEWARE`——它负责在每个请求进来时，根据 URL 前缀（本章用的方案）或浏览器 `Accept-Language` 头，判断这次请求该用哪种语言，然后把结果写进 `request.LANGUAGE_CODE`，后续模板渲染、`gettext` 调用都靠这个值决定输出哪种语言的文案。
- `LANGUAGES` 显式列出这个项目支持哪些语言——不写这个的话 Django 会假设支持它内置的几十种语言，这里限定成只有英语和西班牙语两种。
- `PARLER_LANGUAGES` 是给 `django-parler` 单独配置的（和 `LANGUAGES` 概念上对应但是独立的一份配置），`fallback: 'en'` 表示如果某条商品数据没有西班牙语翻译，显示英文兜底而不是空白；`hide_untranslated: False` 表示未翻译的对象仍然会出现在查询结果里（用回退语言显示），而不是被直接隐藏。

## 二、URL 层：`i18n_patterns` 与语言前缀

```python
# myshop/urls.py
urlpatterns = i18n_patterns(
    path('admin/', admin.site.urls),
    path(_('cart/'), include('cart.urls', namespace='cart')),
    path(_('orders/'), include('orders.urls', namespace='orders')),
    path(_('payment/'), include('payment.urls', namespace='payment')),
    path(_('coupons/'), include('coupons.urls', namespace='coupons')),
    path('rosetta/', include('rosetta.urls')),
    path('', include('shop.urls', namespace='shop')),
)

urlpatterns += [
    path('payment/webhook/', webhooks.stripe_webhook, name='stripe-webhook'),
]
```

- `i18n_patterns()` 把传进去的所有路由自动加上**语言代码前缀**（比如 `/en/cart/`、`/es/cart/`），并且会根据当前语言自动选择正确的语言版本——这是本章实现"每种语言一个独立 URL 空间"的核心机制，用户访问 `/es/` 开头的网址，`LocaleMiddleware` 就会把整段会话的语言切到西班牙语。
- **更值得注意的是 `path(_('cart/'), ...)` 里的这个 `_()`** ——不仅仅是页面文案会翻译，**连 URL 路径本身的单词也会被翻译**！`cart/urls.py`、`orders/urls.py` 等各 app 内部的 URL（比如 `path(_('create/'), ...)`）同理。查 `es/LC_MESSAGES/django.po` 能找到对应词条把 `cart/` 翻成西班牙语路径片段——这样西班牙语用户看到的购物车页面 URL 会是本地化过的路径而不是英文单词，这是很多欧洲电商网站常见的 SEO/本地化实践（搜索引擎更容易判断这是一个西班牙语页面）。
- **`stripe_webhook` 这条路由被特意放在 `i18n_patterns()` 之外**，直接追加到 `urlpatterns` 里，且没有语言前缀，固定是 `payment/webhook/`。原因很直接：调用这个 URL 的是 **Stripe 的服务器**，不是浏览器里的真人用户，Stripe 后台配置的 webhook 地址是写死的一个固定 URL，不可能，也不应该跟着"当前用户选择的语言"变来变去——如果这条路由也被 `i18n_patterns()` 包裹，Stripe 的回调请求就得精确匹配某个语言前缀（比如 `/en/payment/webhook/`），一旦这个前缀因为语言配置调整而改变，webhook 就直接失效。**这是一个很值得记住的模式：面向机器/第三方服务的固定端点要放在 `i18n_patterns()` 之外。**

## 三、`django-parler`：让模型字段本身支持多语言

```python
# shop/models.py
class Category(TranslatableModel):
    translations = TranslatedFields(
        name=models.CharField(max_length=200),
        slug=models.SlugField(max_length=200, unique=True),
    )
    ...

class Product(TranslatableModel):
    translations = TranslatedFields(
        name=models.CharField(max_length=200),
        slug=models.SlugField(max_length=200),
        description=models.TextField(blank=True),
    )
    category = models.ForeignKey(Category, related_name='products', on_delete=models.CASCADE)
    image = models.ImageField(upload_to='products/%Y/%m/%d', blank=True)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    ...
```

这是本章**架构层面最值得琢磨**的改动。之前 `Category`/`Product` 是普通 `models.Model`，`name`/`slug`/`description` 直接是模型上的字段；现在改成继承 `TranslatableModel`，并且把"需要按语言区分的字段"统一搬进一个 `translations = TranslatedFields(...)` 声明块里，而"不需要翻译的字段"（`category`、`image`、`price`、`available`、`created`、`updated`）依然留在类体里正常声明。

`django-parler` 的实现原理是：对每一个 `TranslatableModel`，它在背后自动生成一张**独立的翻译表**（比如 `Product` 对应 `ProductTranslation`，包含 `master`（指回 `Product` 的外键）+ `language_code` + `name`/`slug`/`description`），一件商品有几种语言的翻译，这张表里就有几行记录。`product.name` 这种访问方式在运行时会自动根据当前语言去翻译表里查对应那一行——这是"一份主记录 + N 份按语言拆分的翻译记录"模式，跟很多 CMS/多语言电商系统的数据库设计思路是一致的（相比另一种常见方案——给每个字段直接加 `name_en`/`name_es` 这种语言后缀列——`django-parler` 这种方式在未来增加第三种语言时不需要改表结构，只需要多插入一批翻译行）。

注意 `Meta` 里原来的 `ordering`/`indexes` 大多被**注释掉**了：

```python
class Meta:
    # ordering = ['name']
    # indexes = [
    #     models.Index(fields=['name']),
    # ]
    verbose_name = 'category'
    verbose_name_plural = 'categories'
```

这是因为 `name` 现在已经不是 `Category` 表自己的字段了，而是躺在关联的翻译表里——普通的 `Meta.ordering = ['name']`/`models.Index(fields=['name'])` 直接指向一个本表不存在的字段，会报错，所以作者把它们注释掉而不是删除（保留痕迹说明"这里本来是有排序/索引的，多语言化之后需要用 parler 提供的专门方式重新实现，暂未处理"）——这其实是引入 `django-parler` 之后一个**尚未被填平的技术债**：分类列表现在没有确定的排序方式了，产品明细页的"按创建时间索引"还保留着（因为 `created` 不是翻译字段），但"按名称索引/排序"这个能力被牺牲掉了。

### Admin 适配

```python
# shop/admin.py
@admin.register(Category)
class CategoryAdmin(TranslatableAdmin):
    list_display = ['name', 'slug']
    def get_prepopulated_fields(self, request, obj=None):
        return {'slug': ('name',)}
```

`TranslatableAdmin`（`django-parler` 提供）取代了普通的 `admin.ModelAdmin`，后台编辑页会自动出现"语言切换标签页"，让管理员分别填写每种语言版本的字段。有意思的是 `prepopulated_fields`（原来是类属性 `prepopulated_fields = {'slug': ('name',)}`）在这里变成了一个**方法** `get_prepopulated_fields`——这是因为 `TranslatableAdmin` 内部对 `prepopulated_fields` 的静态类属性处理方式和多语言字段有冲突（翻译字段是动态生成的，admin 基类要求通过方法在运行时提供，而不是在类定义时就写死），这是接入第三方"字段级多语言"方案时经常要面对的一类适配细节：不能假设第三方库和 Django admin 的每一个原生特性都无缝兼容，遇到冲突时通常需要按对方文档要求的方式改写。

### 视图层查询方式的变化

```python
# shop/views.py
def product_list(request, category_slug=None):
    ...
    if category_slug:
        language = request.LANGUAGE_CODE
        category = get_object_or_404(
            Category,
            translations__language_code=language,
            translations__slug=category_slug,
        )
        ...

def product_detail(request, id, slug):
    language = request.LANGUAGE_CODE
    product = get_object_or_404(
        Product,
        id=id,
        translations__language_code=language,
        translations__slug=slug,
        available=True,
    )
```

因为 `slug` 现在存在翻译表里，原来简单的 `Category.objects.get(slug=category_slug)` 就不够用了——必须显式带上 `translations__language_code=request.LANGUAGE_CODE` 这个条件，通过 Django ORM 的跨表查询语法（`translations__` 前缀是 parler 生成的翻译表的反向关联名）同时限定"语言代码"和"该语言下的 slug"两个条件，才能查到正确的一条记录。这也解释了为什么 `product_detail` URL 里的 `slug` 参数现在天然是"当前语言版本"的 slug（一件商品的英文 slug 和西班牙语 slug 可以完全不同，比如 `/en/2/blue-shirt/` 和 `/es/2/camisa-azul/`），如果不带语言条件查询，很可能用一种语言的 URL slug 却匹配到另一种语言下面完全不相关的记录，或者根本查不到。

## 四、UI 文案国际化（i18n）

模板层大量使用 `{% load i18n %}` 后的 `{% translate %}`/`{% blocktranslate %}` 标签：

```django
{% load i18n static %}
{% translate "My shop" %}
{% translate "Your cart" %}:
{% blocktranslate with total=cart.get_total_price count items=total_items %}
  {{ items }} item, ${{ total }}
{% plural %}
  {{ items }} items, ${{ total }}
{% endblocktranslate %}
```

- `{% translate "..." %}` 用于单个简单字符串。
- `{% blocktranslate %}...{% endblocktranslate %}` 用于**带变量插值、甚至带单复数变化**的句子——上面这段就是"购物车里有 1 件商品"和"购物车里有 N 件商品"两种单复数表达（`count` + `{% plural %}` 语法），因为不同语言的单复数规则不一样（英语只有单数/复数两态，有些语言有更多态），Django 的 i18n 系统能通过 `.po` 文件里的 `Plural-Forms` 规则正确处理。
- 语言切换器：

```django
{% get_current_language as LANGUAGE_CODE %}
{% get_available_languages as LANGUAGES %}
{% get_language_info_list for LANGUAGES as languages %}
<ul class="languages">
  {% for language in languages %}
    <li>
      <a href="/{{ language.code }}/" {% if language.code == LANGUAGE_CODE %}class="selected"{% endif %}>
        {{ language.name_local }}
      </a>
    </li>
  {% endfor %}
</ul>
```

`{% get_language_info_list %}` 这几个 i18n 模板标签直接从 `settings.LANGUAGES` 读取配置，动态生成语言切换链接列表，`language.name_local` 显示的是"该语言用自己的文字写出的名字"（比如西班牙语显示"Español"而不是"Spanish"），是比较地道的多语言站点惯例。**这里语言切换链接是硬编码 `/{{ language.code }}/`（直接跳回首页）**，而不是"切换语言但停留在当前页面"，是一个可以改进但本章没做的细节。

## 五、`orders/forms.py`：地区特化表单字段（l10n）

```python
from localflavor.us.forms import USZipCodeField

class OrderCreateForm(forms.ModelForm):
    postal_code = USZipCodeField()
    class Meta:
        model = Order
        fields = ['first_name', 'last_name', 'email', 'address', 'postal_code', 'city']
```

`USZipCodeField` 是 `django-localflavor` 提供的现成校验字段，专门校验美国邮编格式（5 位数字，或"5位-4位"的 ZIP+4 格式）——这是 i18n（界面语言）和 l10n（本地化格式规则，比如邮编、电话号码、身份证号）的区别所在：这里国际化的是界面文案，但这个表单字段本身只针对美国邮编格式做校验，**并没有随着 `LANGUAGES` 切到西班牙语就换成对应国家的邮编规则**——如果商城真的要面向说西班牙语的用户（很可能是拉美或西班牙本土用户），这里的邮编校验逻辑实际上还停留在"只认美国格式"的阶段，这是"翻译了界面文字，但没有真正做地区适配"的一个典型局限，值得作为语言支持和地区支持是两件独立事情的例子。

`Order`/`OrderItem` 模型字段也加上了可翻译的 `verbose_name`：

```python
first_name = models.CharField(_('first name'), max_length=50)
last_name = models.CharField(_('last name'), max_length=50)
email = models.EmailField(_('e-mail'))
...
```

这样表单标签（`form.as_p` 渲染出来的 `<label>`）在不同语言下也会显示对应语言的字段名。

## 六、`django-rosetta`：浏览器里编辑翻译

`path('rosetta/', include('rosetta.urls'))` 挂载后，管理员登录 admin 后可以访问 `/rosetta/` 直接在网页表单里逐条编辑 `.po` 文件里的翻译词条，保存后立即生效（内部帮你跑了编译 `.po → .mo` 的步骤），不需要开发者手动用命令行工具维护翻译文件。`locale/es/LC_MESSAGES/django.po` 文件本身也印证了这一点——文件头部写着 `X-Translated-Using: django-rosetta 0.9.8`，说明这些翻译词条本身就是通过 Rosetta 界面维护产出的。

## 七、终于补上的 Redis 服务

顺带一提一处**跨章节的遗留问题在本章被修复**：`docker-compose.yml` 新增了：

```yaml
cache:
  image: redis:7.2.4
  restart: always
  volumes:
    - ./data/cache:/data
```

Chapter10 就已经引入了基于 Redis 的商品推荐系统（`shop/recommender.py`），但当时的 `docker-compose.yml` 里始终**没有**配套的 Redis 服务——直到这一章才补上。这是继续深读代码时又一次印证的"仓库配置滞后于代码功能一个章节"的模式（此前 Chapter08 的 Celery broker 也是类似情况，虽然那个直到目前为止依然没有被补上）。

## 八、`prompts/task.md`——又一份与本章主题无关的练习草稿

和之前几章一样，这不是本章正文实现，是作者留的 AI 提示词草稿，内容和 i18n 完全无关——问的是"如何给 `Product` 加一个按克计重的 `weight` 字段，并根据订单总重量计算运费，同时让 Stripe 收款金额包含运费"。这份草稿本身描述的 `Product`/`Order` 模型代码还是 Chapter10 版本（`Product` 还不是 `TranslatableModel`），说明这份草稿是在本章开发之前、独立于 i18n 这条主线写的一次探索性提问，目前完全没有被落实到代码里。继续深读后续章节时，如果看到"运费计算"相关功能出现，可以回头对照这份草稿。

## 九、端到端流程

用户访问首页 → `LocaleMiddleware` 根据 URL 前缀（或浏览器语言）决定 `request.LANGUAGE_CODE` → 所有 `{% translate %}`/`{% blocktranslate %}` 标签按此语言输出文案，商品/分类的 `name`/`description`/`slug` 通过 `translations__language_code` 查询对应语言的翻译记录 → 用户点击顶部语言切换链接 → 跳转到另一语言前缀的首页，重新走一遍上述流程，同一件商品会展示为不同语言的名称和描述、不同的 slug 路径。管理员可以在 admin 后台的多语言标签页里为每件商品逐语言填写内容，或用 Rosetta 界面维护界面文案的翻译词条。

## 十、Chapter10 → Chapter11 变化小结

| 方面 | Chapter10 | Chapter11 |
|---|---|---|
| 界面语言 | 仅英语（硬编码字符串） | 英语/西班牙语双语（`{% translate %}`/`{% blocktranslate %}` + `.po` 翻译文件） |
| 商品/分类数据 | 单一语言字段（普通 `CharField`） | `TranslatableModel` + `TranslatedFields`，每种语言独立一份 name/slug/description |
| URL 结构 | 无语言前缀 | `i18n_patterns()` 加语言前缀，且路径片段本身也被翻译 |
| 表单校验 | 通用字段 | 引入地区特化字段（`USZipCodeField`），但语言切换未联动地区规则 |
| 翻译维护方式 | 无 | Rosetta 后台可视化编辑 `.po` 文件 |
| 遗留问题状态 | Redis 服务未接入 `docker-compose.yml`（推荐系统需要） | **已修复**：新增 `cache` (Redis) 服务 |
| 新增遗留问题 | — | `Category`/`Product` 的 `ordering`/`按名称索引` 被注释掉，多语言化后未重新实现 |
