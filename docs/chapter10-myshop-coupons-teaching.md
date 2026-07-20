# Chapter10 教学笔记：myshop 项目 —— 优惠券系统与基于 Redis 的商品推荐

Chapter10 给商城加了两个独立但都基于 Redis 的功能：**优惠券折扣**（数据库存储券，session 记录当前生效的券）和**"购买了这个的人也买了"式商品推荐**（纯 Redis 排序集合实现的协同过滤）。

## 一、新依赖与配置

```diff
+ redis==5.0.4
```

`settings.py` 新增：

```python
INSTALLED_APPS += ['coupons.apps.CouponsConfig']

# Redis settings
REDIS_HOST = 'localhost'
REDIS_PORT = 6379
REDIS_DB = 1
```

`REDIS_DB = 1`——注意这里用的是 Redis 的第 1 号逻辑数据库（Redis 默认提供 0-15 共 16 个相互隔离的逻辑库，同一个 Redis 实例可以给不同用途分库使用，避免键名冲突）。这个项目目前只有推荐系统用 Redis，选 `db=1` 更多是给未来扩展留出 `db=0` 的余地（对比 Bookmarks 项目 Chapter07 用的是 `REDIS_DB = 0`，两个项目各自独立部署，不存在真正的冲突，但这仍是一个值得注意的编号习惯）。

`myshop/urls.py` 新增 `path('coupons/', include('coupons.urls', namespace='coupons'))`。

## 二、`coupons` 应用：优惠券模型与校验

```python
# coupons/models.py
class Coupon(models.Model):
    code = models.CharField(max_length=50, unique=True)
    valid_from = models.DateTimeField()
    valid_to = models.DateTimeField()
    discount = models.IntegerField(
        validators=[MinValueValidator(0), MaxValueValidator(100)],
        help_text='Percentage vaule (0 to 100)',
    )
    active = models.BooleanField()
```

`discount` 用整数存百分比（0-100），配上 `MinValueValidator`/`MaxValueValidator` 在模型层就把范围锁死——即便管理员在 admin 后台手滑输入了 150 或 -20，`full_clean()`/表单校验会直接拒绝，不会有折扣率把订单价格算成负数这种荒谬情况。`active` 字段没给 `default`，意味着 admin 新建一张券时必须显式勾选，防止运营人员创建后忘了激活就以为已经生效（或者反过来，误以为默认是关闭的）。

```python
# coupons/views.py
@require_POST
def coupon_apply(request):
    now = timezone.now()
    form = CouponApplyForm(request.POST)
    if form.is_valid():
        code = form.cleaned_data['code']
        try:
            coupon = Coupon.objects.get(
                code__iexact=code,
                valid_from__lte=now,
                valid_to__gte=now,
                active=True,
            )
            request.session['coupon_id'] = coupon.id
        except Coupon.DoesNotExist:
            request.session['coupon_id'] = None
    return redirect('cart:cart_detail')
```

四个条件一次性在数据库层过滤完：`code__iexact`（大小写不敏感匹配，用户输入 `SAVE10` 或 `save10` 都能命中）、`valid_from__lte=now` 和 `valid_to__gte=now`（必须在有效期内）、`active=True`（未被停用）——四个条件全部满足才算一张真正可用的券。找不到匹配的券时，直接把 `session['coupon_id']` 设成 `None` 并静默重定向回购物车页，**没有通过 `messages` 框架给用户任何"优惠码无效"的提示**——用户输错码或用了过期码，页面上唯一的反馈是"折扣行没有出现"，这是一个可以改进的 UX 细节（比如像 Bookmarks 项目 Chapter05 那样接入 `django.contrib.messages` 给出明确错误提示），但目前的实现没有这层反馈。

## 三、购物车/订单里的折扣计算

`cart/cart.py` 新增：

```python
def __init__(self, request):
    ...
    self.coupon_id = self.session.get('coupon_id')

@property
def coupon(self):
    if self.coupon_id:
        try:
            return Coupon.objects.get(id=self.coupon_id)
        except Coupon.DoesNotExist:
            pass
    return None

def get_discount(self):
    if self.coupon:
        return (self.coupon.discount / Decimal(100)) * self.get_total_price()
    return Decimal(0)

def get_total_price_after_discount(self):
    return self.get_total_price() - self.get_discount()
```

`coupon` 做成 `@property`——每次访问 `cart.coupon` 都会重新查一次数据库（而不是缓存结果），这样如果优惠券在用户浏览购物车的过程中被管理员在后台停用/过期，`cart.coupon` 立刻会返回 `None`，折扣立刻消失，不会出现"券已失效但购物车还按旧折扣算价"的不一致状态。`get_discount()` 用 `Decimal(100)` 而不是普通整数 `100` 做除法——延续了整个项目"金额计算全程用 `Decimal`，绝不引入 `float`"的一贯纪律。

**下单那一刻的折扣同样要做"快照"**，`orders/views.py`：

```python
order = form.save(commit=False)
if cart.coupon:
    order.coupon = cart.coupon
    order.discount = cart.coupon.discount
order.save()
```

`Order` 新增两个字段：

```python
coupon = models.ForeignKey(Coupon, related_name='orders', null=True, blank=True, on_delete=models.SET_NULL)
discount = models.IntegerField(default=0, validators=[MinValueValidator(0), MaxValueValidator(100)])
```

`order.discount` 是从 `coupon.discount` **复制**过来的一份独立整数，而不是每次都通过 `order.coupon.discount` 现查——这是本书里第三次出现"下单时把易变数据做快照"的设计（前两次是 `OrderItem.price` 和购物车 session 里的 `price`）：即便这张优惠券以后被管理员改了折扣比例甚至删除，历史订单的折扣记录必须保持不变。`on_delete=models.SET_NULL`（配合 `null=True`）也印证了这个意图——优惠券被删除后，`order.coupon` 变成 `NULL`，但 `order.discount` 这个整数字段完全不受影响，订单金额计算依然正确。

`Order.get_total_cost()` 现在拆成了三层：

```python
def get_total_cost_before_discount(self):
    return sum(item.get_cost() for item in self.items.all())

def get_discount(self):
    total_cost = self.get_total_cost_before_discount()
    if self.discount:
        return total_cost * (self.discount / Decimal(100))
    return Decimal(0)

def get_total_cost(self):
    return self.get_total_cost_before_discount() - self.get_discount()
```

这套"折扣前小计 / 折扣金额 / 折扣后总计"三段式计算，在购物车页、结算页、支付摘要页、PDF 发票、admin 订单详情**五个模板里原样复用**（`{% if cart.coupon %}`/`{% if order.coupon %}` 判断块几乎逐字重复），是这一章模板层面最明显的重复模式。

### Stripe 侧同步应用折扣

`payment/views.py` 新增：

```python
if order.coupon:
    stripe_coupon = stripe.Coupon.create(
        name=order.coupon.code,
        percent_off=order.discount,
        duration='once',
    )
    session_data['discounts'] = [{'coupon': stripe_coupon.id}]
```

这里有个容易被忽略但很关键的点：**折扣计算了两遍，而且是有意为之的**——本地数据库/模板层已经按 `order.discount` 算出了"折后总价"给用户看，但真正决定 Stripe 收多少钱的是 Stripe 自己的 Checkout Session。所以这里没有直接把折后总价传给 Stripe，而是把订单明细（原价）和"这张券打几折"分别传过去，**让 Stripe 在它自己的系统里重新计算一遍折扣**（`stripe.Coupon.create(percent_off=order.discount, duration='once')` 现场在 Stripe 那边创建一张一次性生效的优惠券对象）。这样做的好处是 Stripe 的支付页面上用户能清楚看到"原价多少、优惠多少、实付多少"的明细，而不是本地暗中算好一个折后价直接甩给 Stripe——用户体验和账目透明度都更好，也符合支付合规审计里"金额构成要清晰可追溯"的要求。

## 四、商品推荐系统：`shop/recommender.py`

这是本章**技术含量最高的部分**——一个完全基于 Redis 有序集合（Sorted Set）实现的"购买了 A 的人也购买了 B"协同过滤推荐，不涉及任何机器学习库。

```python
class Recommender:
    def get_product_key(self, id):
        return f'product:{id}:purchased_with'

    def products_bought(self, products):
        product_ids = [p.id for p in products]
        for product_id in product_ids:
            for with_id in product_ids:
                if product_id != with_id:
                    r.zincrby(self.get_product_key(product_id), 1, with_id)
```

- 每个商品 ID 对应一个独立的 Redis 有序集合，键名 `product:{id}:purchased_with`，集合成员是"和它一起被买过的其他商品 ID"，分数是"一起被买过的次数"。
- `products_bought` 在**一笔订单支付成功后**被调用一次（在 `payment/webhooks.py` 里，见下），传入这笔订单买了哪些商品。双重循环遍历"每个商品 × 每个商品"的所有组合（排除自己和自己配对），给每一对商品互相在对方的 Sorted Set 里加一分——这本质上是在离线维护一张**商品共现频次的稀疏矩阵**，但用 Redis 的数据结构天然就是"矩阵的一行"（每个商品自己的 Sorted Set），完全不需要专门的矩阵库或离线批处理任务，每次订单支付成功时增量更新即可。

```python
def suggest_products_for(self, products, max_results=6):
    product_ids = [p.id for p in products]
    if len(products) == 1:
        suggestions = r.zrange(self.get_product_key(product_ids[0]), 0, -1, desc=True)[:max_results]
    else:
        flat_ids = ''.join([str(id) for id in product_ids])
        tmp_key = f'tmp_{flat_ids}'
        keys = [self.get_product_key(id) for id in product_ids]
        r.zunionstore(tmp_key, keys)
        r.zrem(tmp_key, *product_ids)
        suggestions = r.zrange(tmp_key, 0, -1, desc=True)[:max_results]
        r.delete(tmp_key)
    suggested_products_ids = [int(id) for id in suggestions]
    suggested_products = list(Product.objects.filter(id__in=suggested_products_ids))
    suggested_products.sort(key=lambda x: suggested_products_ids.index(x.id))
    return suggested_products
```

这是整章最精巧的一段：

- **单商品场景**（商品详情页，"看了这个的人也看了"）：直接 `zrange` 取这一个商品对应 Sorted Set 里分数最高的几个 ID，就是它的推荐列表。
- **多商品场景**（购物车页，"根据你购物车里的所有商品综合推荐"）：这里不能简单地对每个商品各自的推荐列表取交集或者简单拼接，而是用 `r.zunionstore(tmp_key, keys)`——**Redis 原生的多 Sorted Set 求并集并把对应分数相加**的操作，把购物车里每个商品各自的"共现列表"合并成一个临时的综合排行榜（同一个候选商品如果在多个购物车商品的共现列表里都出现，分数会累加，天然体现"这个商品和购物车里好几件商品都常被一起买"这个更强的推荐信号）。
- `r.zrem(tmp_key, *product_ids)`——合并之后必须把"购物车里已经有的商品自己"从推荐结果里剔除，不然会出现"给你推荐你购物车里已经有的东西"这种没有意义的推荐。
- `r.delete(tmp_key)`——`tmp_key` 只是这一次计算过程中的临时中间结果（键名里嵌入了具体商品 ID 组合，理论上不会长期占用内存，但显式删除更干净，避免每次不同购物车组合都新建一个从不清理的临时键，长期攒着造成 Redis 内存泄漏）。
- 和 Chapter07 图片排行榜那次一样的**手动重排序**问题再次出现：`Product.objects.filter(id__in=suggested_products_ids)` 不保证按传入 ID 顺序返回，所以还是要 `.sort(key=lambda x: suggested_products_ids.index(x.id))` 在 Python 层按 Redis 给出的分数顺序重新排一遍——这是这本书里第二次用一模一样的手法解决"跨存储排序对齐"问题，值得作为一个可复用的模式记住。

```python
def clear_purchases(self):
    for id in Product.objects.values_list('id', flat=True):
        r.delete(self.get_product_key(id))
```

清空所有推荐数据的运维工具方法，目前没有被任何视图调用，应该是留给命令行/管理任务手动调用重置推荐数据用的。

### 推荐数据从哪里"学习"

```python
# payment/webhooks.py —— stripe_webhook 里，订单确认支付成功之后
product_ids = order.items.values_list('product_id')
products = Product.objects.filter(id__in=product_ids)
r = Recommender()
r.products_bought(products)
```

关键设计：**推荐系统的"学习"信号绑定在支付成功（webhook 确认）这一刻，而不是下单提交、也不是加入购物车**。这个选择是有意义的——只有真正完成支付的订单才代表"用户确实一起买了这些东西"这个可信的行为信号，如果绑定在"加入购物车"甚至"提交结算表单"，会混入大量后来放弃支付、取消订单的噪声数据，污染推荐的准确性。这也是为什么这段代码出现在 `stripe_webhook` 里而不是 `order_create` 视图里的原因。

### 推荐结果展示的两个位置

- 商品详情页（`shop/views.py` 的 `product_detail`）：`r.suggest_products_for([product], 4)`，单商品场景，"看了这个的人也买了"。
- 购物车页（`cart/views.py` 的 `cart_detail`）：`r.suggest_products_for(cart_products, max_results=4)`（购物车为空时 `recommended_products = []`），多商品综合推荐，"购买了这些商品的人也买了"。

两处模板（`product/detail.html`、`cart/detail.html`）都用了几乎一样的 `.recommendations` 区块结构渲染推荐商品的图片+名称+链接，是又一处"同一 UI 模式跨页面复用"的例子。

## 五、端到端流程

**应用优惠券**：购物车页填写券码提交 → `coupon_apply` 校验券码合法性/有效期/是否激活 → 命中就把 `coupon.id` 写进 `session['coupon_id']`，未命中静默清空（无错误提示）→ 重定向回购物车页 → 购物车页通过 `cart.coupon` 这个 property 实时反查显示折扣明细。

**下单支付**：结算提交时把当前 `cart.coupon`/折扣百分比复制进 `Order.coupon`/`Order.discount`（快照）→ 支付页把订单明细 + 折扣百分比一起传给 Stripe，让 Stripe 现场创建一次性优惠券并在它自己的支付页上展示折扣明细 → webhook 确认支付成功后，除了原有的"标记已付款、发邮件"，额外调用 `Recommender.products_bought()` 把这笔订单的商品组合喂给推荐系统，供以后的购物车/商品详情页做推荐。

## 六、Chapter09 → Chapter10 变化小结

| 方面 | Chapter09 | Chapter10 |
|---|---|---|
| 折扣能力 | 无 | 优惠券系统（数据库存储 + session 记录当前生效券 + 下单时快照） |
| 金额计算 | 单一总价 | 三段式（折扣前小计 / 折扣金额 / 折扣后总计），五处模板复用 |
| Stripe 集成深度 | 仅创建 Checkout Session | + 现场创建一次性 `stripe.Coupon`，把折扣显式传给 Stripe |
| 商品推荐 | 无 | 基于 Redis Sorted Set 的"一起购买"协同过滤（单品/多品两种场景） |
| Redis 用途 | Bookmarks 项目用于浏览计数/排行榜（另一个项目） | myshop 项目首次引入 Redis，专用于推荐系统（`REDIS_DB=1`） |
| 已知薄弱点 | Webhook 未接线 broker（Ch08 遗留，Ch09 已解决）；`created.html` 死代码 | 优惠码无效时无用户可见反馈（静默失败） |
