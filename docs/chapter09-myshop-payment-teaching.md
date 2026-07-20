# Chapter09 教学笔记：myshop 项目 —— Stripe 支付、Webhook 与 PDF 发票

Chapter09 引入了完整的 **Stripe 支付流程**——结算不再是"提交表单就完事"，而是要走"下单 → 跳转 Stripe 托管支付页 → Stripe 异步回调确认支付"这一整套真实电商支付链路，外加 PDF 发票生成和后台管理增强。

## 一、本章定位与新依赖

```diff
+ stripe==9.3.0
+ python-decouple==3.8
+ WeasyPrint==61.2
```

`Dockerfile` 也相应新增了一大串系统级 C 库依赖（`libcairo2`、`libpango-1.0-0`、`libpangocairo-1.0-0`、`libgdk-pixbuf2.0-0` 等）——这些不是 Python 包，是 **WeasyPrint 把 HTML/CSS 渲染成 PDF 时依赖的底层图形渲染库**（Cairo/Pango 是 Linux 桌面级排版渲染引擎），这也是"深读代码"该养成的习惯：看到 `requirements.txt` 加了某个包，如果这个包做的是"渲染/图形/多媒体处理"这类工作，往往意味着 Dockerfile 也得跟着加系统依赖，纯 `pip install` 是不够的。

`settings.py` 新增：

```python
from decouple import config
...
INSTALLED_APPS += ['payment.apps.PaymentConfig']
STATIC_ROOT = BASE_DIR / 'static'

# Stripe settings
STRIPE_PUBLISHABLE_KEY = config('STRIPE_PUBLISHABLE_KEY')
STRIPE_SECRET_KEY = config('STRIPE_SECRET_KEY')
STRIPE_API_VERSION = '2024-04-10'
STRIPE_WEBHOOK_SECRET = config('STRIPE_WEBHOOK_SECRET')
```

三个 Stripe 密钥全部走 `python-decouple` 从环境变量读取，不硬编码进代码库——这是继续沿用 Bookmarks 项目 Chapter05 里管理 Google OAuth 密钥的同一套做法。`docker-compose.yml` 里对应加了：

```yaml
environment:
  - STRIPE_PUBLISHABLE_KEY=key
  - STRIPE_SECRET_KEY=secret
  - STRIPE_WEBHOOK_SECRET=secret-hook
```

这三个值只是占位符（`key`/`secret`/`secret-hook`），跑真实支付时需要替换成 Stripe 后台真正的测试/生产密钥。

## 二、结算流程改道：从"直接完成"变成"先建单再付款"

对比 Chapter08，`orders/views.py` 的 `order_create` 有个关键改动：

```python
cart.clear()
order_created.delay(order.id)
# set the order in the session
request.session['order_id'] = order.id
# redirect for payment
return redirect('payment:process')
```

Chapter08 时提交结算表单后直接渲染 `orders/order/created.html`（"谢谢下单"页），Chapter09 改成把 `order.id` 存进 session 后**跳转到支付页**——这意味着"订单创建成功"和"订单已付款"被拆成了两个独立阶段（`Order.paid` 字段默认 `False`，直到 Stripe webhook 确认付款才会被置为 `True`）。

**顺带发现一处死代码**：`orders/order/created.html` 这个模板文件依然留在项目里，但搜遍整个代码库已经**没有任何 Python 代码再渲染它**——`order_create` 视图现在总是重定向到 `payment:process`，不会走到渲染 `created.html` 这条分支了。这是一处典型的"重构后忘记清理"的遗留文件，属于继续深读代码时值得留意的细节。

## 三、`payment` 应用：Stripe Checkout 集成

```python
# payment/views.py
stripe.api_key = settings.STRIPE_SECRET_KEY
stripe.api_version = settings.STRIPE_API_VERSION

def payment_process(request):
    order_id = request.session.get('order_id')
    order = get_object_or_404(Order, id=order_id)

    if request.method == 'POST':
        success_url = request.build_absolute_uri(reverse('payment:completed'))
        cancel_url = request.build_absolute_uri(reverse('payment:canceled'))

        session_data = {
            'mode': 'payment',
            'client_reference_id': order.id,
            'success_url': success_url,
            'cancel_url': cancel_url,
            'line_items': [],
        }
        for item in order.items.all():
            session_data['line_items'].append({
                'price_data': {
                    'unit_amount': int(item.price * Decimal('100')),
                    'currency': 'usd',
                    'product_data': {'name': item.product.name},
                },
                'quantity': item.quantity,
            })

        session = stripe.checkout.Session.create(**session_data)
        return redirect(session.url, code=303)
    else:
        return render(request, 'payment/process.html', locals())
```

几个关键点：

- **本项目本身不处理任何银行卡信息**——`payment_process` 只是调用 `stripe.checkout.Session.create()` 在 Stripe 那边创建一个"结账会话"，然后把用户**重定向到 Stripe 托管的支付页面**（`session.url`）。真正输入卡号、CVV 的表单是 Stripe 的域名下渲染的，这是电商接入第三方支付网关的标准做法——**永远不要自己接触/存储用户的银行卡数据**，一是安全风险极高，二是自己处理卡数据需要通过 PCI-DSS 合规认证，成本极高，绝大多数中小站点都靠这种"跳转到支付方托管页面"的集成方式规避这个问题。
- `unit_amount': int(item.price * Decimal('100'))`——Stripe API 要求金额以**最小货币单位**（美元是"分"）传整数，不能传浮点小数。`item.price` 是 `Decimal`，乘以 `Decimal('100')` 转成"分"再转 `int`，全程避免了浮点数精度问题（如果直接用 `float` 乘 100 再取整，理论上可能因为浮点误差算错一分钱）。
- `client_reference_id': order.id`——这是把"我方系统的订单 ID"塞进 Stripe 会话里的关键字段，后面 webhook 收到 Stripe 的回调事件时，就是靠这个字段反查到底是哪一笔本地订单。
- `redirect(session.url, code=303)`——显式指定 `303 See Other` 而不是默认的 `302`。原因：当前请求是**表单 POST 过来的**，如果用 302，某些浏览器/中间代理在处理重定向时会保留原始请求方法（继续用 POST 访问新地址），303 明确规定"用 GET 请求重定向目标"，避免浏览器把这次 POST 请求的表单数据也带到 Stripe 的 URL 上去。

`payment_completed`/`payment_canceled` 两个视图非常简单，只是渲染静态提示页——**它们只是"用户浏览器被 Stripe 重定向回来时看到的页面"，跟订单是否真的被标记为已付款是两码事**（下一节会讲这一点为什么重要）。

## 四、Webhook：真正标记订单已付款的地方

```python
# payment/webhooks.py
@csrf_exempt
def stripe_webhook(request):
    payload = request.body
    sig_header = request.META['HTTP_STRIPE_SIGNATURE']
    try:
        event = stripe.Webhook.construct_event(
            payload, sig_header, settings.STRIPE_WEBHOOK_SECRET
        )
    except ValueError:
        return HttpResponse(status=400)
    except stripe.error.SignatureVerificationError:
        return HttpResponse(status=400)

    if event.type == 'checkout.session.completed':
        session = event.data.object
        if session.mode == 'payment' and session.payment_status == 'paid':
            try:
                order = Order.objects.get(id=session.client_reference_id)
            except Order.DoesNotExist:
                return HttpResponse(status=404)
            order.paid = True
            order.stripe_id = session.payment_intent
            order.save()
            payment_completed.delay(order.id)

    return HttpResponse(status=200)
```

这是本章**安全设计上最值得细讲**的一段代码：

- **为什么这个视图要 `@csrf_exempt`？** 因为这个 URL 不是给"登录用户在自己浏览器里"提交表单用的，是**Stripe 的服务器直接向本网站发起的服务器对服务器 HTTP 请求**——Stripe 服务器没有、也不可能带上本站签发的 CSRF token，Django 的 CSRF 保护机制本身就是为了防"第三方网站伪造本站用户的请求"，这里的请求方向恰恰相反（是本站信任的第三方主动来通知），所以必须显式豁免。
- **但豁免了 CSRF 不代表不设防**——`stripe.Webhook.construct_event(payload, sig_header, settings.STRIPE_WEBHOOK_SECRET)` 这一步是**验证这个请求真的是 Stripe 发来的，而不是别人伪造的**：Stripe 会用只有 Stripe 和你共享的 `STRIPE_WEBHOOK_SECRET` 对请求体做签名，放在 `Stripe-Signature` 请求头里，`construct_event` 会重新计算签名并比对，任何一点篡改（金额、订单 ID、签名本身）都会导致 `SignatureVerificationError`。**这一步验证如果被跳过，会是个严重的安全漏洞**——任何知道 webhook URL 的人都能直接 POST 一个伪造的"支付成功"事件，把任意订单标记为已付款而不用真的付一分钱。这是本项目里**唯一一处"服务器信任外部输入前必须验证来源"的关键校验点**，值得和之前 Chapter06 的 SSRF 隐患对照着看——一边是"该校验没校验"，一边是"确实做了校验"的正面例子。
- **`session.mode == 'payment' and session.payment_status == 'paid'` 双重判断**——Stripe 的 `checkout.session.completed` 事件在多种场景都会触发（比如 `mode='subscription'` 的订阅类支付），这里显式限定只处理"一次性付款模式且确实付款成功"的事件，避免把不相关的会话类型误判成订单付款。
- **异步任务 `payment_completed.delay(order.id)`**——和 Chapter08 的 `order_created` 是同一个模式：webhook 处理逻辑本身要尽快返回 `200`（Stripe 对 webhook 响应有超时限制，长时间不响应会被判定失败并重试），而"生成 PDF + 发邮件"这种耗时操作丢给 Celery 后台执行。

## 五、`Order.stripe_id` 与 `get_stripe_url`

```python
# orders/models.py
stripe_id = models.CharField(max_length=250, blank=True)

def get_stripe_url(self):
    if not self.stripe_id:
        return ''
    if '_test_' in settings.STRIPE_SECRET_KEY:
        path = '/test/'
    else:
        path = '/'
    return f'https://dashboard.stripe.com{path}payments/{self.stripe_id}'
```

`stripe_id` 存的是 webhook 里拿到的 `session.payment_intent`（Stripe 那笔支付的唯一标识），`get_stripe_url` 借助 `STRIPE_SECRET_KEY` 里是否包含 `_test_` 字样（Stripe 的测试密钥形如 `sk_test_...`，正式密钥是 `sk_live_...`）来判断该拼测试环境还是正式环境的 Dashboard 链接——这样管理员在后台点这个链接，能直接跳到 Stripe 控制台查看这笔支付的详情，不用先手动判断当前是测试还是生产环境。

## 六、PDF 发票：WeasyPrint

同一段 PDF 生成逻辑在两个地方几乎一模一样地出现：

```python
# orders/views.py —— 管理员手动下载
@staff_member_required
def admin_order_pdf(request, order_id):
    order = get_object_or_404(Order, id=order_id)
    html = render_to_string('orders/order/pdf.html', {'order': order})
    response = HttpResponse(content_type='application/pdf')
    response['Content-Disposition'] = f'filename=order_{order.id}.pdf'
    weasyprint.HTML(string=html).write_pdf(
        response, stylesheets=[weasyprint.CSS(finders.find('css/pdf.css'))],
    )
    return response
```

```python
# payment/tasks.py —— 支付成功后自动发邮件附件
@shared_task
def payment_completed(order_id):
    order = Order.objects.get(id=order_id)
    subject = f'My Shop - Invoice no. {order.id}'
    message = 'Please, find attached the invoice for your recent purchase.'
    email = EmailMessage(subject, message, 'admin@myshop.com', [order.email])
    html = render_to_string('orders/order/pdf.html', {'order': order})
    out = BytesIO()
    stylesheets = [weasyprint.CSS(finders.find('css/pdf.css'))]
    weasyprint.HTML(string=html).write_pdf(out, stylesheets=stylesheets)
    email.attach(f'order_{order.id}.pdf', out.getvalue(), 'application/pdf')
    email.send()
```

两处的核心手法一致：`render_to_string` 把一个普通 Django 模板（`orders/order/pdf.html`，纯 HTML + 单独一份 `pdf.css`）渲染成字符串，再交给 `weasyprint.HTML(string=html).write_pdf(...)` 转成 PDF 二进制——**用 Django 模板系统生成 PDF 内容，而不是用专门的 PDF 绘图 API 一行行摆坐标**，这是个很讨巧的做法：设计 PDF 排版就跟写一个网页一样，用 HTML/CSS 布局，WeasyPrint 负责把这套网页渲染结果转换成分页的 PDF。

两处唯一的区别是**输出目标**：`admin_order_pdf` 把 PDF 直接写进 HTTP `response`（用户点击链接直接下载/预览）；`payment_completed` 里把 PDF 写进内存里的 `BytesIO()` 缓冲区，再作为邮件附件发送，都没有真的把 PDF 文件存到磁盘上——生成即用即弃，不占用服务器存储。

`finders.find('css/pdf.css')`——用的是 Django 静态文件查找器（staticfiles finder），能在开发环境（`DEBUG=True`，直接从各 app 的 `static/` 目录查找）和生产环境（`collectstatic` 之后从 `STATIC_ROOT` 查找）下都能定位到这个 CSS 文件的真实磁盘路径，因为 WeasyPrint 需要的是文件系统路径而不是一个 URL。

## 七、Admin 后台增强

```python
# orders/admin.py
def export_to_csv(modeladmin, request, queryset):
    ...
    writer = csv.writer(response)
    fields = [f for f in opts.get_fields() if not f.many_to_many and not f.one_to_many]
    writer.writerow([field.verbose_name for field in fields])
    for obj in queryset:
        ...
    return response
export_to_csv.short_description = 'Export to CSV'

def order_payment(obj):
    url = obj.get_stripe_url()
    if obj.stripe_id:
        return mark_safe(f'<a href="{url}" target="_blank">{obj.stripe_id}</a>')
    return ''

def order_detail(obj):
    return mark_safe(f'<a href="{reverse("orders:admin_order_detail", args=[obj.id])}">View</a>')

def order_pdf(obj):
    return mark_safe(f'<a href="{reverse("orders:admin_order_pdf", args=[obj.id])}">PDF</a>')

@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = [..., order_payment, ..., order_detail, order_pdf]
    actions = [export_to_csv]
```

- `export_to_csv` 是一个**自定义 admin action**——`opts.get_fields()` 动态反射出模型的所有字段（排除多对多和反向一对多这类不能直接塞进一行 CSV 单元格的关系字段），批量导出管理员在 admin 列表页勾选的订单为 CSV，这是运营/财务导数据的常见需求，不用为此单独写一个视图。
- `order_payment`/`order_detail`/`order_pdf` 是三个**当作"伪字段"塞进 `list_display` 的函数**——`mark_safe` 把拼出来的 `<a>` 标签当 HTML 直接渲染（不转义），让 admin 列表页里直接出现"跳转 Stripe 控制台"、"查看详情"、"下载 PDF"三个可点击链接。这是 Django admin 里"在列表页里塞自定义交互元素"的标准手法。
- `admin_order_detail`/`admin_order_pdf` 两个视图用的是 `@staff_member_required` 而不是之前熟悉的 `@login_required`——这是权限层级的差异：`@login_required` 只要求"已登录"，`@staff_member_required` 额外要求 `user.is_staff=True`，因为这两个视图暴露的是**所有客户的订单详情**，必须限制只有后台管理人员能访问，而不是任何注册用户都能看。

## 八、端到端流程

1. 结算表单提交 → 创建 `Order` + 逐条 `OrderItem`（价格快照） → 清空购物车 → 派发 `order_created` 异步任务（下单确认邮件） → `order.id` 存入 session → 跳转 `payment:process`。
2. 用户在结算摘要页点击"Pay now" → 后端用订单明细在 Stripe 创建 Checkout Session → `303` 跳转到 Stripe 托管的支付表单页（本站不接触卡号）。
3. 用户在 Stripe 页面完成支付：
   - **浏览器侧**：Stripe 把用户重定向回 `success_url`（`payment:completed`），展示"支付成功"静态提示页。
   - **服务器侧（独立、异步）**：Stripe 服务器直接 POST 一个 `checkout.session.completed` 事件到 `payment:stripe-webhook` → 验证签名 → 查到对应 `Order` → 标记 `paid=True`、写入 `stripe_id` → 派发 `payment_completed` 异步任务（生成 PDF 发票并作为邮件附件发送）。

**这两条路径互相独立、没有先后保证**——浏览器跳转和服务器 webhook 是 Stripe 并行触发的两个通知渠道，`payment_completed.html` 只是个"提示用户"的页面，真正代表"这笔钱确实到账了"的状态变更，永远只发生在 webhook 处理成功之后。这也是为什么电商系统的标准实践是"**永远以 webhook/服务器端回调作为支付状态的唯一真相来源，而不是相信浏览器跳转本身**"——如果用户在跳转回 `success_url` 之前直接关闭了浏览器标签页，只要 Stripe 那边的支付确实成功了，webhook 依然会把订单标记为已付款；反过来，如果只靠"用户跳回了 success_url 就认为付款成功"，则完全可能被伪造（自己在浏览器里手动访问一下 `success_url` 就行，压根不需要真的付钱）——这也是为什么 `payment_completed`/`payment_canceled` 两个视图里没有一行代码去改 `order.paid`。

## 九、Chapter08 → Chapter09 变化小结

| 方面 | Chapter08 | Chapter09 |
|---|---|---|
| 支付能力 | 无（下单即"完成"） | Stripe Checkout 托管支付 + Webhook 异步确认 |
| 订单状态流转 | 提交即视为完成，渲染感谢页 | 提交→跳转支付→（webhook 异步）标记 `paid` |
| 新增字段 | 无 | `Order.stripe_id` + `get_stripe_url()` |
| 发票/单据 | 无 | WeasyPrint 生成 PDF（人工下载 + 邮件附件两处复用） |
| Admin 能力 | 基础字段列表 + inline | + CSV 导出 action、Stripe 链接列、详情/PDF 链接列 |
| 安全关注点 | SSRF（Chapter06 遗留）、Celery broker 缺失 | Webhook 签名校验（**正面范例**）、`created.html` 死代码 |
