# Chapter08 教学笔记：第三个项目 myshop —— 商城、购物车与 Celery 异步任务

Chapter08 另起炉灶，开启本书**第三个项目 myshop**（在线商城），不再是 Bookmarks 的延续。这一章的核心是三样东西：**商品/分类模型**、**基于 session 的购物车**（不落库）、**Celery 异步发邮件**。

## 一、项目结构与新依赖

```diff
+ Pillow~=11.2       # ImageField 需要
+ flower==2.0.1       # Celery 任务监控 Web 面板
+ celery==5.4.0        # 异步任务队列
```

三个新 app：`shop`（商品/分类）、`cart`（购物车）、`orders`（订单）。`settings.py` 里能看到几处新东西：

```python
TEMPLATES = [{
    ...
    'OPTIONS': {
        'context_processors': [
            ...
            'cart.context_processors.cart',   # 让 cart 在所有模板里都能直接用
        ],
    },
}]

CART_SESSION_ID = 'cart'
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
```

## 二、商品与分类模型

```python
# shop/models.py
class Category(models.Model):
    name = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200, unique=True)
    class Meta:
        ordering = ['name']
        indexes = [models.Index(fields=['name'])]
        verbose_name = 'category'
        verbose_name_plural = 'categories'
    def get_absolute_url(self):
        return reverse('shop:product_list_by_category', args=[self.slug])

class Product(models.Model):
    category = models.ForeignKey(Category, related_name='products', on_delete=models.CASCADE)
    name = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200)
    image = models.ImageField(upload_to='products/%Y/%m/%d', blank=True)
    description = models.TextField(blank=True)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    available = models.BooleanField(default=True)
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)
    class Meta:
        ordering = ['name']
        indexes = [
            models.Index(fields=['id', 'slug']),
            models.Index(fields=['name']),
            models.Index(fields=['-created']),
        ]
    def get_absolute_url(self):
        return reverse('shop:product_detail', args=[self.id, self.slug])
```

值得注意的细节：

- `verbose_name_plural = 'categories'`——英文复数不能简单加 s（category → categories），Django admin 侧边栏、表头默认会自动给 `verbose_name` 加 s，遇到这种不规则复数必须手动指定，否则 admin 里会显示成 "Categorys"。
- `Product.slug` **没有** `unique=True`（对比 `Category.slug` 是 `unique=True`）——因为 `product_detail` 这个 URL 用的是 `<int:id>/<slug:slug>/`，真正定位记录靠的是 `id`，`slug` 只是让 URL 好看/利于 SEO，所以允许不同商品重名 slug；而 `Category` 的详情页 URL `product_list_by_category` 完全靠 `slug` 定位（`<slug:category_slug>/`，没有 `id`），所以分类的 slug 必须唯一。这是"URL 设计决定字段约束"的一个好例子。
- `Product.price` 用 `DecimalField` 而不是 `FloatField`——涉及金额永远该用 `DecimalField`，因为浮点数在做金额加减乘除时会有精度误差（`0.1 + 0.2 != 0.3` 这种经典问题），这是处理货币金额的铁律。

## 三、购物车：完全不落库，全靠 session

这是本章**设计上最值得琢磨**的地方——`cart/models.py` 是空的！购物车压根不是一张数据库表，而是一个**读写 session 字典**的普通 Python 类：

```python
# cart/cart.py
class Cart:
    def __init__(self, request):
        self.session = request.session
        cart = self.session.get(settings.CART_SESSION_ID)
        if not cart:
            cart = self.session[settings.CART_SESSION_ID] = {}
        self.cart = cart

    def add(self, product, quantity=1, override_quantity=False):
        product_id = str(product.id)
        if product_id not in self.cart:
            self.cart[product_id] = {'quantity': 0, 'price': str(product.price)}
        if override_quantity:
            self.cart[product_id]['quantity'] = quantity
        else:
            self.cart[product_id]['quantity'] += quantity
        self.save()

    def save(self):
        self.session.modified = True

    def __iter__(self):
        product_ids = self.cart.keys()
        products = Product.objects.filter(id__in=product_ids)
        cart = self.cart.copy()
        for product in products:
            cart[str(product.id)]['product'] = product
        for item in cart.values():
            item['price'] = Decimal(item['price'])
            item['total_price'] = item['price'] * item['quantity']
            yield item

    def __len__(self):
        return sum(item['quantity'] for item in self.cart.values())

    def get_total_price(self):
        return sum(Decimal(item['price']) * item['quantity'] for item in self.cart.values())
```

几个精心设计的细节：

- **价格存的是字符串**（`'price': str(product.price)`）——Django session 底层默认用 JSON 序列化（`django.contrib.sessions` 的 `JSONSerializer`），而 `Decimal` 类型不能直接 JSON 序列化，所以存进 session 前要 `str()`，取出来再 `Decimal()` 转回去。这也顺带解释了为什么购物车里存的是**下单那一刻的价格快照**，而不是每次都去查 `Product.price`——即使商品之后涨价了，已经在购物车里的商品价格也不会跟着变，这其实是电商场景里"该有的行为"（虽然本书目前还没有专门为这点写注释，但代码结构自然达成了）。
- `self.session.modified = True`——Django session 默认只在**顶层键**被整体赋值时才知道要重新保存 session；这里是在字典**内部**做的嵌套修改（`self.cart[product_id][...] = ...`），Django 检测不到这种深层变化，所以必须手动标记 `modified = True` 强制触发 session 重新写入，否则购物车的修改根本不会持久化。这是使用 Django session 存复杂数据结构时一个经典的坑。
- `__iter__` 是**生成器**协议的应用：先按 session 里存的所有 `product_id` 一次性批量查数据库（`filter(id__in=product_ids)`，避免 N+1），再把 `Product` 对象和 `Decimal` 价格"贴"回每个 cart item 里，最后逐个 `yield`——这样上层代码（视图、模板）能直接 `for item in cart:` 遍历，item 既有 `product`（模型实例）又有 `price`/`quantity`/`total_price`（计算好的数值），使用起来非常自然，把"session 字典"和"数据库对象"缝合成了一个统一视图。
- **购物车不需要登录**——全程没有 `@login_required`，因为购物车挂在 session 上而不是某个 User 上，这是电商网站"先逛后登录/游客下单"的常见诉求。

### `context_processors.py`——让购物车在所有模板里都能用

```python
def cart(request):
    return {'cart': Cart(request)}
```

配合 `settings.py` 里注册的 `'cart.context_processors.cart'`，每个模板渲染时都会自动往 context 里注入一个 `cart` 变量——这就是为什么 `shop/base.html`（所有页面公用的头部）里能直接写 `{% with total_items=cart|length %}` 显示购物车图标和总件数，而不需要每个视图都手动传 `cart` 进 context。这是 context processor 的标准用法：把"几乎每个页面都要用"的数据从"每个视图手动传"变成"全局自动可用"。

## 四、购物车视图：三个纯 session 操作

```python
@require_POST
def cart_add(request, product_id):
    cart = Cart(request)
    product = get_object_or_404(Product, id=product_id)
    form = CartAddProductForm(request.POST)
    if form.is_valid():
        cd = form.cleaned_data
        cart.add(product=product, quantity=cd['quantity'], override_quantity=cd['override'])
    return redirect('cart:cart_detail')
```

`CartAddProductForm` 里藏了个小机关：

```python
class CartAddProductForm(forms.Form):
    quantity = forms.TypedChoiceField(choices=PRODUCT_QUANTITY_CHOICES, coerce=int)
    override = forms.BooleanField(required=False, initial=False, widget=forms.HiddenInput)
```

`override` 是个**隐藏字段**——同一张表单在两个场景复用：商品详情页"加入购物车"时 `override` 默认 `False`（新增数量，累加到已有基础上）；购物车页面"更新数量"时后端渲染表单会显式传 `initial={'override': True}`（`cart_detail` 视图里那句 `CartAddProductForm(initial={'quantity': item['quantity'], 'override': True})`），提交后是**覆盖**而不是累加当前数量。一个表单、一个视图函数，靠一个隐藏布尔值区分"添加"和"修改"两种语义，是个很紧凑的设计。

`@require_POST` 用在 `cart_add`/`cart_remove` 上，是因为这两个操作会修改状态（购物车内容），不该允许用 GET 触发（防止链接预抓取、CSRF 类问题意外清空/修改购物车）。

## 五、订单：购物车 → 数据库的落地时刻

```python
# orders/models.py
class Order(models.Model):
    first_name = models.CharField(max_length=50)
    ...
    paid = models.BooleanField(default=False)
    def get_total_cost(self):
        return sum(item.get_cost() for item in self.items.all())

class OrderItem(models.Model):
    order = models.ForeignKey(Order, related_name='items', on_delete=models.CASCADE)
    product = models.ForeignKey('shop.Product', related_name='order_items', on_delete=models.CASCADE)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    quantity = models.PositiveIntegerField(default=1)
    def get_cost(self):
        return self.price * self.quantity
```

`OrderItem` 自己也存了一份 `price`（而不是通过 `product.price` 现查）——这是购物车"价格快照"设计的延续，且这里的理由更硬：**订单一旦生成，历史订单的金额永远不能因为商品后续改价而跟着变**，所以 `OrderItem.price` 必须是下单那一刻从购物车里搬过来的固定值，是电商订单系统里典型的"反规范化换正确性"设计。

`orders/views.py`：

```python
def order_create(request):
    cart = Cart(request)
    if request.method == 'POST':
        form = OrderCreateForm(request.POST)
        if form.is_valid():
            order = form.save()
            for item in cart:
                OrderItem.objects.create(
                    order=order, product=item['product'],
                    price=item['price'], quantity=item['quantity'],
                )
            cart.clear()
            order_created.delay(order.id)
            return render(request, 'orders/order/created.html', {'order': order})
    else:
        form = OrderCreateForm()
    return render(request, 'orders/order/create.html', {'cart': cart, 'form': form})
```

流程很清楚：表单校验通过 → 建 `Order` → 遍历购物车里每个 item 建对应 `OrderItem`（把购物车快照的价格原样搬过去）→ `cart.clear()` 清空 session 购物车 → **`order_created.delay(order.id)`** 派发异步任务 → 渲染"感谢下单"页面。

## 六、Celery：异步发确认邮件

这是本章引入的新基础设施。`myshop/celery.py`：

```python
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myshop.settings')
app = Celery('myshop')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()
```

`myshop/__init__.py`：

```python
from .celery import app as celery_app
__all__ = ['celery_app']
```

这个 `__init__.py` 里的 import 是**必须的**，且是 Celery 官方推荐的标准接线方式：确保 Django 项目启动（比如跑 `manage.py` 任何命令）时 `celery_app` 一定会被创建，这样 `@shared_task` 装饰的任务才能正确关联到这个 Celery 实例上。

```python
# orders/tasks.py
@shared_task
def order_created(order_id):
    order = Order.objects.get(id=order_id)
    subject = f'Order nr. {order.id}'
    message = (f'Dear {order.first_name},\n\n'
               f'You have successfully placed an order.'
               f'Your order ID is {order.id}.')
    mail_sent = send_mail(subject, message, 'admin@myshop.com', [order.email])
    return mail_sent
```

`@shared_task` 而不是 `@app.task`——`shared_task` 是给"可复用 app"用的装饰器写法，不需要在任务定义处直接导入某个具体的 Celery 实例，任务会在真正执行时才绑定到当前项目的 Celery app 上，这样 `orders` 这个 app 理论上可以被拷到别的项目里用而不用改这行代码。

**为什么发邮件要放到 Celery 异步任务而不是直接在视图里同步调 `send_mail`？** 因为发邮件涉及连接外部 SMTP 服务器，网络延迟不可控（慢的时候可能几秒甚至超时）——如果放在 `order_create` 视图同步执行，用户提交订单后浏览器要一直等邮件发送成功才能看到"谢谢下单"页面，体验很差，一旦邮件服务商抽风还可能导致整个下单请求超时失败。用 `.delay()` 把这个任务丢进消息队列，视图立刻继续渲染确认页返回给用户，真正发邮件的动作在后台 worker 进程里异步进行，两者解耦。

### 一处配置缺口：没有配置消息代理（Broker）

逐个文件核对过，`settings.py` 里**没有任何 `CELERY_BROKER_URL`（或 `CELERY_*` 相关）配置**，`docker-compose.yml` 里也**没有** RabbitMQ/Redis 这类消息代理服务，`requirements.txt` 里也没有对应的客户端库（比如 `redis`）。而 Celery **必须**依赖一个消息代理才能把 `.delay()` 派发的任务从 Web 进程传递给 worker 进程——单靠这个仓库里的代码，`order_created.delay(order.id)` 这行会在找不到 broker 时报连接错误。

这说明书的正文（讲义部分）大概率会讲到"启动 RabbitMQ/Redis 作为 broker，并设置 `CELERY_BROKER_URL`，再单独起一个 `celery -A myshop worker` 进程"，但**这一章配套仓库的代码没有把这部分接线完整落到 `docker-compose.yml`/`settings.py` 里**。跟着这一章的代码直接 `docker compose up` 是能跑起来商城本身的（浏览商品、加购物车、下单都不受影响，因为这些操作本身不依赖 Celery），只有真正触发 `.delay()` 那一刻会报错。这是继续深读代码时要留意的一个"代码仓库落后于书本正文"的例子，和之前 Chapter03/07 里 `prompts/task.md` 草稿是同一类问题的另一种表现——书和配套仓库不完全同步。

## 七、模板与前端

- `shop/base.html`：`{% with total_items=cart|length %}` 用了自定义的 `Cart.__len__`（返回商品总件数之和，不是种类数）；购物车为空时用 `{% elif not order %}` 判断——因为下单成功页 `created.html` 里 `cart` 这个 context 变量其实已经不存在了（视图渲染 `created.html` 时只传了 `order`，没传 `cart`），加这个 `not order` 判断是为了避免"下单成功页"上仍然显示一句"购物车是空的"这种奇怪文案。
- `product/detail.html`：`{% if product.image %}...{% else %}{% static "img/no_image.png" %}{% endif %}`——给没上传图的商品准备了一张占位图，这个模式在 `product/list.html` 和 `cart/detail.html` 里重复了三次，是个可以抽成模板 include 或自定义模板标签的小优化点，不过对教学项目来说影响不大。
- `orders/order/create.html`：结算页把购物车明细和收货信息表单放在同一页，`form.as_p` 直接渲染 `OrderCreateForm` 的六个字段，提交后端做校验、创建订单、清空购物车、派发异步任务，一气呵成。

## 八、端到端流程

浏览商品（`shop:product_list` / `product_list_by_category`）→ 商品详情页选数量提交 `cart:cart_add`（POST，session 里累加）→ 购物车页可继续调整数量或删除（`cart_add` 覆盖模式 / `cart_remove`）→ 点击"Checkout"进入 `orders:order_create` 填收货信息 → 提交后创建 `Order` + 逐条 `OrderItem`（价格来自购物车快照）→ 清空 session 购物车 → 派发 `order_created` Celery 任务（需要 broker，当前仓库配置缺失）→ 展示"Thank you"页。

## 九、与 Bookmarks 项目的设计对比

| 方面 | Bookmarks（Ch04-07） | myshop（Ch08） |
|---|---|---|
| 核心数据存储 | 全部在数据库（含"临时性"的浏览计数用 Redis） | 购物车**完全不落库**，只存 session；下单那一刻才落库 |
| 用户体系 | 贯穿始终，几乎每个操作都要登录 | 购物车/浏览商品全程不需要登录，下单表单直接收集联系方式 |
| 价格/金额字段 | 无 | 首次出现 `DecimalField`，且反复出现"快照价格"设计（购物车 session、`OrderItem.price`） |
| 异步任务 | 无 | 首次引入 Celery（`.delay()` 异步发邮件） |
| 已知代码缺口 | `add_to_class`/`ABSOLUTE_URL_OVERRIDES`（因未用自定义 User 模型） | Celery broker 未接线（`settings.py`/`docker-compose.yml`/`requirements.txt` 均缺失） |
