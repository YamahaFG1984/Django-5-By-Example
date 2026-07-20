# Chapter16 教学笔记：educa 项目 —— 用 Django Channels 实现课程实时聊天室

Chapter16 给 educa 加上了一个**实时聊天室**功能：每门课程一个专属聊天室，报名的学生能看到彼此发的消息并实时收到新消息推送。这是全书第一次涉及 **WebSocket** 和**异步（ASGI）**编程模型——和之前十五章清一色的"请求-响应"式 HTTP 视图完全是两套不同的编程范式。

## 一、新依赖与 ASGI 化改造

```diff
+ channels[daphne]==4.1.0
+ channels-redis==4.2.0
```

`django-channels` 是 Django 官方项目组维护的扩展库，把 Django 从"只能处理短生命周期的 HTTP 请求-响应"扩展成能处理**长连接**（WebSocket）。`daphne` 是 Channels 官方推荐的 ASGI 服务器（替代传统的 WSGI 服务器，能同时服务普通 HTTP 请求和 WebSocket 长连接）。`channels-redis` 提供了"Channel Layer"的 Redis 实现——这是让**多个不同的 WebSocket 连接互相通信**的关键基础设施（下面详细讲）。

```python
INSTALLED_APPS = ['daphne', ..., 'chat.apps.ChatConfig', ...]

ASGI_APPLICATION = 'educa.asgi.application'

CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {'hosts': [('127.0.0.1', 6379)]},
    },
}
```

`'daphne'` 被放在 `INSTALLED_APPS` **最前面**——这是官方要求的顺序，`daphne` 需要在 Django 自己的 `runserver`/静态文件处理逻辑加载之前完成对开发服务器命令的接管（把默认的 WSGI 开发服务器替换成能同时处理 HTTP 和 WebSocket 的 ASGI 版本）。

`educa/asgi.py` 的改动是本章"接入 Channels"最核心的一步：

```python
django_asgi_app = get_asgi_application()

from chat.routing import websocket_urlpatterns

application = ProtocolTypeRouter({
    'http': django_asgi_app,
    'websocket': AuthMiddlewareStack(URLRouter(websocket_urlpatterns)),
})
```

`ProtocolTypeRouter` 是整个应用的顶层入口——**按协议类型分流**：普通的 HTTP 请求依然交给原来那套 Django 应用（`django_asgi_app`，本质上还是走 URL 路由 → 视图 → 模板这条老路）处理，而 WebSocket 连接被分流到一套**完全独立的路由体系**（`chat/routing.py` 里的 `websocket_urlpatterns`）。这意味着这一章并不是把整个网站都改造成异步的，而是**只在需要长连接的这一小块功能上叠加了一套并行的处理管线**，其余十五章积累下来的所有 HTTP 视图完全不受影响，继续按原来的方式运行。

`AuthMiddlewareStack` 是关键——它让 WebSocket 连接也能拿到和普通 HTTP 请求一样的**基于 session 的用户认证信息**（浏览器发起 WebSocket 握手时依然会带上 session cookie），后面在 Consumer 里能直接通过 `self.scope['user']` 拿到当前登录用户，用法上和 `request.user` 几乎一致。

## 二、`Message` 模型：聊天记录持久化

```python
# chat/models.py
class Message(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.PROTECT, related_name='chat_messages')
    course = models.ForeignKey('courses.Course', on_delete=models.PROTECT, related_name='chat_messages')
    content = models.TextField()
    sent_on = models.DateTimeField(auto_now_add=True)
```

**`on_delete=models.PROTECT`**（两个外键都是）——这是本书目前为止第一次使用 `PROTECT`（之前见过的都是 `CASCADE`/`SET_NULL`）。`PROTECT` 的行为是：如果一个 `User` 或 `Course` 还关联着任何聊天记录，**禁止删除**这个 User/Course，数据库层面会直接抛 `ProtectedError`，阻止删除操作执行。这是一个刻意的数据保留策略——聊天记录被当作某种需要长期保存的"历史档案"，管理员不能通过删除课程或用户来顺带悄悄清空聊天记录，如果真的要删除，必须先显式处理掉相关的聊天消息。这和之前项目里"删除即级联清理干净"（`CASCADE`）的默认思路形成了有意思的对比。

## 三、`ChatConsumer`：本章的核心

```python
# chat/consumers.py
class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.user = self.scope['user']
        self.id = self.scope['url_route']['kwargs']['course_id']
        self.room_group_name = f'chat_{self.id}'
        await self.channel_layer.group_add(self.room_group_name, self.channel_name)
        await self.accept()

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(self.room_group_name, self.channel_name)

    async def persist_message(self, message):
        await Message.objects.acreate(user=self.user, course_id=self.id, content=message)

    async def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json['message']
        now = timezone.now()
        await self.channel_layer.group_send(
            self.room_group_name,
            {'type': 'chat_message', 'message': message, 'user': self.user.username, 'datetime': now.isoformat()},
        )
        await self.persist_message(message)

    async def chat_message(self, event):
        await self.send(text_data=json.dumps(event))
```

**Consumer** 是 Channels 里的核心概念，大致相当于"WebSocket 版的视图"——不是处理一次性的请求-响应，而是管理一条**持续存在的连接**的生命周期，`connect`/`disconnect`/`receive` 分别对应"连接建立"、"连接断开"、"收到客户端发来的消息"三个事件钩子，全部用 `async def` 定义（异步方法），这也是全书第一次出现原生的 Python 异步语法。

### Channel Layer：让多个连接互相"广播"

这是理解这段代码最关键的一点——**一个 WebSocket 连接只能和它自己的客户端通信**，但聊天室需要"A 发的消息，B、C、D 都要实时收到"，这就需要一种**跨连接的消息分发机制**，这正是 `channel_layer`（底层由 `channels-redis` 提供，本质上是用 Redis 做消息中转）存在的意义：

- `self.channel_layer.group_add(self.room_group_name, self.channel_name)`——每个 WebSocket 连接一建立，就把自己（`self.channel_name`，Channels 自动分配的这条连接的唯一标识）加入一个以课程 ID 命名的**分组**（`chat_{course_id}`）。同一门课程的所有正在聊天的用户，连接都会被加进同一个分组。
- `self.channel_layer.group_send(self.room_group_name, {...})`——`receive()` 收到某个用户发来的一条消息后，不是直接回传给发送者自己，而是**把这条消息广播给整个分组**（这门课程聊天室里所有当前在线的连接）。
- `chat_message(self, event)`——**这是 Channels 的一个约定**：`group_send` 传入的字典里 `'type': 'chat_message'` 这个键决定了分组里每个连接接下来会调用**自己 Consumer 实例上同名的方法** `chat_message`（Channels 自动把下划线命名 `chat_message` 映射过去）。也就是说，`receive()` 广播出去的消息，最终会在**每一个**订阅了这个分组的连接对象上触发一次它自己的 `chat_message()` 方法，`self.send(text_data=json.dumps(event))` 才是真正把消息通过 WebSocket 推给对应客户端浏览器的那一步。这套"发送方法名即路由键"的设计是 Channels 分组广播机制的核心约定，第一次接触容易觉得绕，但理解了就会发现这是一种很简洁的"发布-订阅"实现。
- `Message.objects.acreate(...)`——用的是 Django 5.x 提供的**异步 ORM 方法**（`acreate`，对应同步版本的 `create`），因为整个 Consumer 都跑在异步事件循环里，如果直接调用同步的 `Message.objects.create(...)` 会阻塞整个事件循环，`acreate` 让这次数据库写入以异步方式执行、不阻塞其他并发连接的处理。持久化和广播是**独立的两步**（先广播用于实时展示，再落库），这样即使写库稍慢，也不会拖慢消息的实时送达。

## 四、Consumer 的路由与视图入口

```python
# chat/routing.py
websocket_urlpatterns = [
    re_path(r'ws/chat/room/(?P<course_id>\d+)/$', consumers.ChatConsumer.as_asgi()),
]
```

WebSocket 路由用的是 `re_path`（正则表达式路由）而不是 Django 3.1+ 之后更现代的路径转换器写法（`path('ws/chat/room/<int:course_id>/', ...)`）——这是 Channels 生态里比较常见的历史遗留写法，`\d+` 这个正则本身完全等价于 `<int:course_id>` 的效果。

```python
# chat/views.py
@login_required
def course_chat_room(request, course_id):
    try:
        course = request.user.courses_joined.get(id=course_id)
    except Course.DoesNotExist:
        return HttpResponseForbidden()
    latest_messages = course.chat_messages.select_related('user').order_by('-id')[:5]
    latest_messages = reversed(latest_messages)
    return render(request, 'chat/room.html', {'course': course, 'latest_messages': latest_messages})
```

这是一个**普通的同步 Django 视图**（不是 Consumer），负责渲染聊天室页面的初始 HTML（包括最近 5 条历史消息，用于用户一进聊天室就能看到一点上下文，而不是从空白开始）——`request.user.courses_joined.get(id=course_id)` 这行同时完成了"课程是否存在"和"当前用户是否报名了这门课"两个检查，查不到直接走 `except` 分支返回 `403 Forbidden`。`latest_messages` 取最新 5 条（按 `-id` 倒序取出后再 `reversed()` 一次，让最终展示顺序变回"从旧到新"，符合聊天记录的自然阅读顺序）。

### 一处值得关注的授权缺口：WebSocket 连接本身没有重新校验报名资格

对比一下：`course_chat_room` 这个**HTTP 入口视图**认真做了"用户是否报名了这门课程"的校验，但真正建立 WebSocket 连接的 `ChatConsumer.connect()` **完全没有重复这个检查**：

```python
async def connect(self):
    self.user = self.scope['user']
    self.id = self.scope['url_route']['kwargs']['course_id']
    self.room_group_name = f'chat_{self.id}'
    await self.channel_layer.group_add(self.room_group_name, self.channel_name)
    await self.accept()
```

它只是从 URL 里拿到 `course_id`，既不检查 `self.user.is_authenticated`（`AuthMiddlewareStack` 对未登录用户会把 `scope['user']` 填成 `AnonymousUser`，而不是拒绝连接），也不检查这个用户是否真的报名了这门课程，就直接 `await self.accept()` 接受了连接并加入对应分组。这意味着：**只要知道或猜到某门课程的 `course_id`，任何人（哪怕是未登录的匿名访客，或者登录了但没有报名这门课的用户）都可以直接向 `ws://.../ws/chat/room/<任意course_id>/` 发起 WebSocket 连接**，加入对应聊天室分组，实时旁听甚至发言、把消息写进数据库——**完全绕开了网页入口视图里那道"必须已报名"的权限检查**。

这是继续深读代码时又一个值得记录的安全隐患：**Web 视图层的权限校验，不会自动延伸到与之配套但独立接入的 WebSocket Consumer 上**，两条通往同一份数据的路径需要**分别**做鉴权。对照 Chapter15 API 层的 `IsEnrolled` 权限类（专门为 `contents` 端点显式做了"是否已报名"的对象级校验）能更清楚地看出，这里的 Consumer 本该在 `connect()` 里补上一段类似 `await self.user.courses_joined.filter(id=self.id).aexists()` 的校验，不通过就 `await self.close()` 拒绝连接——但当前代码没有做这一步。

## 五、前端：原生 WebSocket API

```javascript
const url = 'ws://' + window.location.host + '/ws/chat/room/' + courseId + '/';
const chatSocket = new WebSocket(url);

chatSocket.onmessage = function(event) {
  const data = JSON.parse(event.data);
  ...
  chat.innerHTML += '<div class="message ' + source + '">...';
  chat.scrollTop = chat.scrollHeight;
};

submitButton.addEventListener('click', function(event) {
  const message = input.value;
  if (message) {
    chatSocket.send(JSON.stringify({'message': message}));
    input.value = '';
  }
});
```

不依赖任何第三方 JS 库，直接用浏览器原生的 `WebSocket` API：连接建立后，`onmessage` 回调处理服务端（`chat_message` 方法）推送过来的每一条消息，动态往 `#chat` 容器里追加一段 HTML 并自动滚动到底部；发送消息则是简单的 `chatSocket.send(JSON.stringify({...}))`。`{{ course.id|json_script:"course-id" }}` 是 Django 内置的 `json_script` 过滤器，把 Python 值安全地序列化成一个 `<script type="application/json">` 标签，前端用 `JSON.parse(document.getElementById(...).textContent)` 读取——这是把服务端渲染的数据安全传递给前端 JS 的标准做法，比直接把变量值拼进 JS 代码字符串里更能避免 XSS 风险（`json_script` 会对内容做适当转义）。

**一个值得留意的小细节**：`url` 硬编码用的是 `'ws://'` 前缀，而不是根据当前页面协议动态判断（`window.location.protocol === 'https:' ? 'wss://' : 'ws://'`）。如果这个网站部署在 HTTPS 环境下，浏览器通常会因为"混合内容"安全策略拒绝从一个 HTTPS 页面发起不加密的 `ws://` 连接，实际生产部署这里需要相应调整为 `wss://`。这是教学示例里为了简化而先不处理生产部署细节的又一个例子。

## 六、端到端流程

学生在"我的课程"详情页看到"Course chat room"链接 → 点击进入 `course_chat_room` 视图（校验登录 + 报名资格）→ 渲染聊天室页面，附带最近 5 条历史消息 → 页面加载完毕，前端 JS 立即发起一条 WebSocket 连接 → 服务端 `ChatConsumer.connect()` 把这条连接加入以课程 ID 命名的分组 → 用户输入并发送一条消息 → `receive()` 触发，先把消息广播给分组内所有连接（每个连接各自的 `chat_message()` 被调用，实时推送到各自浏览器），再异步落库 → 分组内其他正在聊天的用户浏览器 `onmessage` 触发，实时看到这条新消息出现在聊天窗口里。

## 七、Chapter15 → Chapter16 变化小结

| 方面 | Chapter15 | Chapter16 |
|---|---|---|
| 通信协议 | 纯 HTTP（含 REST API） | 新增 WebSocket（长连接），HTTP/WebSocket 并行由 `ProtocolTypeRouter` 分流 |
| 编程模型 | 全部同步 | 首次引入原生 `async`/`await` 与异步 ORM 方法（`acreate`） |
| 跨连接通信 | 无此需求 | Channel Layer（Redis 支撑）实现分组广播 |
| 权限校验 | HTTP 视图 + DRF 对象级权限类（`IsEnrolled`）两处独立实现同一规则 | HTTP 视图有校验，**WebSocket Consumer 未重复校验**（新发现的授权缺口） |
| 数据删除策略 | 沿用 `CASCADE`/`SET_NULL` | 首次使用 `on_delete=PROTECT`，聊天记录不可被连带删除 |
| 前端技术 | 无新增 | 原生 `WebSocket` API + `json_script` 过滤器安全传值 |
