# Chapter17 教学笔记：educa 项目 —— 生产部署（Nginx + uWSGI + Daphne）

这是全书最后一章——**生产部署**：把 educa 从"能在本地用 `runserver` 跑起来的教学项目"补齐成一套真正能上线的架构：分环境的 settings、PostgreSQL、Nginx + uWSGI + Daphne 的多进程拓扑、HTTPS、子域名路由，还有一个用于运营召回的管理命令。这也把全书从 Chapter08 起就在 TDD/TSD 讨论里反复提到的"settings 该按环境拆分"这个建议真正落了地。

## 一、`settings` 从单文件拆成包：`base.py`/`local.py`/`prod.py`

```python
# educa/settings/local.py
from .base import *
DEBUG = True
DATABASES = {'default': {'ENGINE': 'django.db.backends.sqlite3', 'NAME': BASE_DIR / 'db.sqlite3'}}
```

```python
# educa/settings/prod.py
from decouple import config
from .base import *

DEBUG = False
ADMINS = [('Antonio M', 'email@mydomain.com')]
ALLOWED_HOSTS = ['.educaproject.com']

DATABASES = {'default': {
    'ENGINE': 'django.db.backends.postgresql',
    'NAME': config('POSTGRES_DB'), 'USER': config('POSTGRES_USER'), 'PASSWORD': config('POSTGRES_PASSWORD'),
    'HOST': 'db', 'PORT': 5432,
}}

REDIS_URL = 'redis://cache:6379'
CACHES['default']['LOCATION'] = REDIS_URL
CHANNEL_LAYERS['default']['CONFIG']['hosts'] = [REDIS_URL]

CSRF_COOKIE_SECURE = True
SESSION_COOKIE_SECURE = True
SECURE_SSL_REDIRECT = True
```

`base.py` 装所有环境共用的配置，`local.py`/`prod.py` 各自 `from .base import *` 后再覆盖差异项——这正是我们在 Blog 项目 TDD 文档里标注过的 TSD 建议（"settings 应该按环境拆分"），在这本书第四个项目、第十七章才第一次真正落地。几个值得展开的点：

- **`ALLOWED_HOSTS = ['.educaproject.com']`**——开头的 `.` 是通配写法，匹配 `educaproject.com` 本身以及**任意子域名**（`python.educaproject.com`、`www.educaproject.com` 等）。这个设计不是随手写的，直接为下面要讲的"课程子域名"中间件铺路。
- **`REDIS_URL = 'redis://cache:6379'`**——`cache` 不是一个真实主机名，而是 `docker-compose.yml` 里 Redis 服务的**服务名**（Docker Compose 内置的服务发现机制会把服务名解析成对应容器的内网 IP）。`local.py` 里 `CACHES`/`CHANNEL_LAYERS` 沿用 `base.py` 默认的 `127.0.0.1`（本地开发假设 Redis 就跑在本机），`prod.py` 则统一把这两处地址替换成 Docker 网络内的服务名——这是"同一份配置结构，不同环境替换掉具体连接地址"的典型写法。
- **`CSRF_COOKIE_SECURE`/`SESSION_COOKIE_SECURE`/`SECURE_SSL_REDIRECT`** 三行——只在生产环境打开：前两个让浏览器只在 HTTPS 连接下才会把 CSRF/Session Cookie 发送出去（HTTP 明文连接不发送，防止中间人窃听到这些敏感 Cookie），`SECURE_SSL_REDIRECT` 让 Django 自动把所有 HTTP 请求 301 跳转到 HTTPS。本地开发环境没有 HTTPS，所以这三项只在 `prod.py` 里打开，`local.py` 保持默认（关闭）。

### 一个需要显式设置才能工作的"契约"

`educa/settings/__init__.py` 是**完全空的文件**，而 `manage.py`/`wsgi.py`/`asgi.py` 里 `os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'educa.settings')` 默认指向的正是这个空的 `educa.settings` **包本身**（不是 `local.py` 也不是 `prod.py`）。这意味着**如果不显式设置 `DJANGO_SETTINGS_MODULE` 环境变量指向 `educa.settings.local` 或 `educa.settings.prod`，Django 启动时会因为缺少 `DATABASES`、`SECRET_KEY` 等必需配置直接报错**——这不是遗漏，而是刻意的设计："默认值"故意留空，逼着每个真正启动应用的入口都必须显式声明自己跑在哪个环境下，避免"忘记指定环境，结果不小心用本地配置连上了生产数据库"这类事故。`docker-compose.yml` 里能看到这份契约的落实：

```yaml
web:
  environment:
    - DJANGO_SETTINGS_MODULE=educa.settings.prod
daphne:
  environment:
    - DJANGO_SETTINGS_MODULE=educa.settings.prod
```

### 一处遗留的"该拆但没拆干净"

`debug_toolbar`、`redisboard`、`rest_framework` 都在 `base.py` 的 `INSTALLED_APPS`/`MIDDLEWARE` 里**无条件声明**，`prod.py` 并没有把 `debug_toolbar` 从生产环境的 `INSTALLED_APPS`/`MIDDLEWARE` 里摘掉——虽然 `debug_toolbar` 自身默认要求 `DEBUG=True` 才会真正渲染工具栏（`prod.py` 里 `DEBUG=False`，实际效果上工具栏不会显示给生产用户），但这依然是"开发期专属工具本该在生产 settings 里彻底移除，而不是靠另一个开关间接兜底"这条原则没有被贯彻到底的例子——和 Bookmarks 项目 Chapter07 首次引入 `debug_toolbar` 时留下的同一处隐患一脉相承，直到全书最后一章也没有被修复。

## 二、`subdomain_course_middleware`：课程专属子域名

```python
# courses/middleware.py
def subdomain_course_middleware(get_response):
    def middleware(request):
        host_parts = request.get_host().split('.')
        if len(host_parts) > 2 and host_parts[0] != 'www':
            course = get_object_or_404(Course, slug=host_parts[0])
            course_url = reverse('course_detail', args=[course.slug])
            url = '{}://{}{}'.format(request.scheme, '.'.join(host_parts[1:]), course_url)
            return redirect(url)
        response = get_response(request)
        return response
    return middleware
```

这是一个**函数式中间件**（Django 中间件的现代写法，直接是一个接收 `get_response` 返回内层函数的闭包，不需要写类）。逻辑是：把当前请求的域名按 `.` 拆开，如果域名段数大于 2（说明带了子域名，比如 `python-basics.educaproject.com` 拆出来是 `['python-basics', 'educaproject', 'com']`）且这个子域名不是 `www`，就把这个子域名当成**课程的 slug** 去查课程，查到了就 301（`redirect` 默认）跳转回主域名下对应的课程详情页 URL；查不到直接 404。

这正是前面 `ALLOWED_HOSTS = ['.educaproject.com']` 通配符存在的原因——每门课程理论上都能有一个形如 `<课程slug>.educaproject.com` 的专属短链接，方便讲师在课程宣传材料上使用一个好记的独立域名，而不需要给每门课单独注册一个真实域名（子域名在 DNS 上通常只需要一条通配符解析记录，`*.educaproject.com` 指向同一台服务器即可）。这个中间件被放在 `MIDDLEWARE` 列表的**最后一位**（`courses.middleware.subdomain_course_middleware`），这样只有在 `AuthenticationMiddleware`/`SessionMiddleware` 等基础设施都处理完之后才轮到它判断子域名，避免影响其他中间件的正常工作。

## 三、生产网络拓扑：Nginx + uWSGI + Daphne 三件套

这是本章**架构层面最重要**的部分——因为 Chapter16 引入了 WebSocket（异步 ASGI），而 Django 主体依然是同步代码（管理端、REST API 等大部分视图都是普通同步视图），生产环境**没法只用一种服务器进程类型**把所有流量都伺候好，于是拆成了两条独立的处理链路，前面用 Nginx 统一分流：

```nginx
upstream uwsgi_app { server unix:/code/educa/uwsgi_app.sock; }
upstream daphne { server daphne:9001; }

server {
    listen 80;
    server_name *.educaproject.com educaproject.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    ssl_certificate /code/educa/ssl/educa.crt;
    ssl_certificate_key /code/educa/ssl/educa.key;
    server_name *.educaproject.com educaproject.com;

    location / {
        include /etc/nginx/uwsgi_params;
        uwsgi_pass uwsgi_app;
    }
    location /ws/ {
        proxy_pass http://daphne;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    location /static/ { alias /code/educa/static/; }
    location /media/ { alias /code/educa/media/; }
}
```

- **80 端口的 server 块**只做一件事——把所有 HTTP 流量 301 强制跳转到 HTTPS（和 `settings.SECURE_SSL_REDIRECT` 是同一个目的的两层保险，一层在 Nginx 网关层拦截，一层在 Django 应用层兜底）。
- **`/` 走 `uwsgi_pass`**——所有普通 HTTP 请求（网页、REST API、admin 后台）交给 uWSGI 进程处理，通过 Unix Socket（而不是 TCP 端口）通信，同机进程间通信用文件系统 socket 通常比 TCP loopback 略快、也不占用额外端口。
- **`/ws/` 走 `proxy_pass` 到 Daphne**，而且**显式设置了 `Upgrade`/`Connection: upgrade` 请求头**——这是 HTTP 协议升级为 WebSocket 协议必须的握手信息，Nginx 默认的反向代理不会自动转发这两个头，必须手动声明，否则 WebSocket 连接会在 Nginx 这一层就握手失败。这一段配置直接对应 `chat/routing.py` 里 `ws/chat/room/...` 这条 URL 前缀。
- **`/static/`、`/media/` 直接用 `alias` 从磁盘伺服**，完全不经过 Python 进程——静态文件/媒体文件交给 Nginx 直接读盘返回，比让 Django/uWSGI 进程去处理这类请求效率高得多，这是 Django 生产部署的标准实践。

对应 `docker-compose.yml` 里两个独立的应用容器：

```yaml
web:
  command: ["./wait-for-it.sh", "db:5432", "--", "uwsgi", "--ini", "/code/config/uwsgi/uwsgi.ini"]
daphne:
  command: ["../wait-for-it.sh", "db:5432", "--", "daphne", "-b", "0.0.0.0", "-p", "9001", "educa.asgi:application"]
```

`uwsgi.ini`：

```ini
[uwsgi]
socket=/code/educa/uwsgi_app.sock
chdir = /code/educa/
module=educa.wsgi:application
master=true
chmod-socket=666
uid=www-data
gid=www-data
vacuum=true
```

`uid`/`gid` 设成 `www-data` 而不是 `root`——**权限最小化**，即便应用进程被攻破，攻击者拿到的也只是一个低权限系统账户，而不是能为所欲为的 root；`vacuum=true` 让 uWSGI 进程退出时自动清理它创建的 socket 文件，避免残留文件导致下次启动时的冲突。

`wait-for-it.sh` 是一个通用的开源 shell 脚本（本身不是 Django/本书特有的东西），作用是"阻塞后续命令执行，直到指定的 `host:port` 真正能连上为止"——`docker-compose.yml` 里的 `depends_on: [db]` **只保证容器按顺序启动**，不保证 PostgreSQL 进程在容器启动的那一瞬间就已经完全就绪、能接受连接（数据库初始化通常需要几秒到几十秒），所以 `web`/`daphne` 两个服务的启动命令都先跑一遍 `wait-for-it.sh db:5432 --` 探测数据库端口，确认真的能连通了才继续执行后面的 `uwsgi`/`daphne` 命令——这是 Docker Compose 环境下"服务依赖就绪等待"的经典解决方案。

## 四、`asgi.py` 的一处安全补强

```python
application = ProtocolTypeRouter({
    'http': django_asgi_app,
    'websocket': AllowedHostsOriginValidator(
        AuthMiddlewareStack(URLRouter(websocket_urlpatterns))
    ),
})
```

对比 Chapter16 的版本，这里多包了一层 `AllowedHostsOriginValidator`——它会检查发起 WebSocket 连接请求的 `Origin` 请求头是否在 `settings.ALLOWED_HOSTS` 允许的域名范围内，不匹配就直接拒绝握手。这是为了防止**跨站 WebSocket 劫持**（CSWSH）——如果没有这层校验，任何第三方网站的 JS 代码都能对着 `wss://educaproject.com/ws/chat/room/1/` 发起连接（浏览器的同源策略对 WebSocket 连接本身并不强制拦截，跟发起 AJAX 请求不一样），`AllowedHostsOriginValidator` 补上了这道"来源域名是否可信"的防线。

**但这层校验和 Chapter16 教学笔记里标注的那个授权缺口是两个不同维度的问题**——`AllowedHostsOriginValidator` 只回答"这次连接是不是从我们信任的域名发起的"，并不回答"这个已登录用户是不是真的报名了 `course_id` 对应的这门课"。也就是说，即便加了这层校验，一个**已登录但没有报名某门课的用户**，依然可以在自己浏览器的开发者工具里手动打开一条指向该课程聊天室的 WebSocket 连接——`ChatConsumer.connect()` 里那处"未校验报名资格"的问题，直到全书最后一章也没有被修复。

## 五、运营召回：自定义管理命令 `enroll_reminder`

```python
# students/management/commands/enroll_reminder.py
class Command(BaseCommand):
    help = 'Sends an e-mail reminder to users registered more than N days that are not enrolled into any courses yet'

    def add_arguments(self, parser):
        parser.add_argument('--days', dest='days', type=int)

    def handle(self, *args, **options):
        emails = []
        subject = 'Enroll in a course'
        date_joined = timezone.now().today() - datetime.timedelta(days=options['days'] or 0)
        users = User.objects.annotate(course_count=Count('courses_joined')) \
            .filter(course_count=0, date_joined__date__lte=date_joined)
        for user in users:
            message = f"""Dear {user.first_name}, We noticed that you didn't enroll in any courses yet..."""
            emails.append((subject, message, settings.DEFAULT_FROM_EMAIL, [user.email]))
        send_mass_mail(emails)
        self.stdout.write(f'Sent {len(emails)} reminders')
```

这是全书**第一次自定义 Django 管理命令**（`python manage.py enroll_reminder --days 7`）——`BaseCommand` 子类 + `add_arguments` 声明命令行参数 + `handle` 实现具体逻辑，是 Django 管理命令的标准三段式写法。业务逻辑本身很直接：`annotate(course_count=Count('courses_joined')).filter(course_count=0, ...)` 一次查询找出"注册超过 N 天、但一门课都没报名"的所有用户，`send_mass_mail`（Django 内置，专门优化过的批量邮件发送函数，会复用同一个 SMTP 连接依次发送多封邮件，比循环调用 `send_mail` 更高效）逐一发送召回提醒邮件。

这是典型的"运营侧召回"场景（把注册了但没有转化行为的用户找出来主动触达），设计上通常要配合定时任务（cron/Celery Beat）每天跑一次——但这个仓库里同样**没有看到任何调度这个命令的配置**（`docker-compose.yml` 里没有额外的 cron 服务或类似机制），命令本身完整可用，只是"如何让它自动定期执行"这一层留给了读者自己在真正部署时补上，是全书里又一处"讲清楚了工具本身，没有把最后一环完全接好线"的例子。

`settings.DEFAULT_FROM_EMAIL` 在整个项目的三份 settings 文件里都没有显式定义——不过这是 Django 自带默认值的设置项（默认为 `'webmaster@localhost'`），不会报错，只是这个默认发件地址大概率不是运营真正想要的寄件人地址，实际部署时需要显式配置。

## 六、`prompts/task.md`：又一份未落地的练习草稿

延续全书的固定模式，这一章同样留了一份作者的 AI 提示词草稿——这次想问的是"如何用 Redis 记录学生在每门课程里上次学习到哪个模块，实现断点续学"。草稿里引用的 `StudentCourseDetailView` 代码还是 Chapter14 的版本（`course.modules.all()[0]` 那处潜在 `IndexError` 依然原样存在），说明这个"断点续学"想法始终停留在探索阶段，没有被实现进正式代码——如果要落地，大概率会复用 Chapter14 讲过的"Redis 存少量高频读写状态"的模式（类似 Chapter07 图片浏览计数、Chapter10 商品推荐），用一个 `student:{user_id}:course:{course_id}:last_module` 之类的键记录最近访问的模块 ID。

## 七、Chapter16 → Chapter17（全书收尾）变化小结

| 方面 | Chapter16 | Chapter17 |
|---|---|---|
| Settings 组织 | 单文件 `settings.py` | 拆分为 `base.py`/`local.py`/`prod.py`，`__init__.py` 留空强制显式声明环境 |
| 数据库 | SQLite | PostgreSQL（生产），密钥走 `python-decouple` 读环境变量 |
| 服务拓扑 | 单一 `runserver` 进程 | Nginx（TLS 终结+分流+静态文件）+ uWSGI（同步 HTTP）+ Daphne（WebSocket）三进程协作 |
| WebSocket 安全 | 无 Origin 校验，无报名资格校验 | 新增 `AllowedHostsOriginValidator`（**部分修复**，报名校验缺口仍未补上） |
| 多租户/路由 | 无 | 子域名中间件，支持课程专属短链接 |
| 运营工具 | 无 | 自定义管理命令 `enroll_reminder`（未接调度） |
| 已知遗留问题 | WebSocket 未校验报名资格 | 同一问题依然存在；`debug_toolbar` 等开发工具未在 `prod.py` 中彻底移除；召回命令未接定时调度 |

全书四个项目（Blog → Bookmarks → myshop → educa）从最基础的博客 CRUD，一路走到社交关系、支付网关、国际化、REST API、WebSocket 实时通信、多进程生产部署——技术栈逐章累加的同时，也贯穿着不少值得记住的反面教材（SSRF、CSRF 豁免、缓存陈旧窗口、授权缺口),这些"没做完/没做对"的地方本身也是这套教学代码库里很有价值的一部分。
