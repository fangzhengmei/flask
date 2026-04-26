# Flask 上下文机制深度分析

## 目录

1. [问题背景：为什么需要上下文？](#1-问题背景为什么需要上下文)
2. [核心技术栈概览](#2-核心技术栈概览)
3. [全局代理对象定义](#3-全局代理对象定义)
4. [AppContext 上下文实现](#4-appcontext-上下文实现)
5. [LocalProxy 透明代理原理](#5-localproxy-透明代理原理)
6. [上下文生命周期管理](#6-上下文生命周期管理)
7. [为什么并发时不会"串"数据？](#7-为什么并发时不会串数据)
8. [总结与架构图](#8-总结与架构图)

---

## 1. 问题背景：为什么需要上下文？

### 1.1 传统全局变量的问题

在 Web 开发中，我们经常需要在视图函数、模板、工具函数中访问：
- 当前请求信息 (`request.method`, `request.args`)
- 当前应用实例 (`current_app.config`)
- 请求级别的全局存储 (`g.user`)
- 用户会话 (`session`)

**传统做法的问题：**

```python
# ❌ 错误示例：使用全局变量
request = None

def handle_request(environ):
    global request
    request = parse_request(environ)  # 多线程下会覆盖！
    return view_function()
```

在多线程/多协程服务器（Gunicorn, Uvicorn, Werkzeug dev server）中：
- 线程 A 设置 `request` 为请求 A
- 线程 B 同时设置 `request` 为请求 B
- 线程 A 访问 `request` 时，读到的是请求 B 的数据！

**这就是"数据串掉"的根本原因。**

### 1.2 需求：线程/协程级别的隔离

我们需要一种机制：
- **看似全局变量**：`from flask import request` 直接导入使用
- **实则局部隔离**：每个线程/协程有自己独立的副本
- **动态绑定**：在请求处理开始时绑定，结束时解绑

这就是 **"上下文本地" (Context Local)** 模式。

---

## 2. 核心技术栈概览

Flask 的上下文机制依赖三层技术：

| 层级 | 技术 | 职责 |
|------|------|------|
| **底层存储** | Python `contextvars.ContextVar` | 线程/协程级别的变量存储 |
| **代理层** | Werkzeug `LocalProxy` | 透明代理，动态查找实际对象 |
| **业务层** | Flask `AppContext` | 封装请求/应用数据的上下文对象 |

### 2.1 技术演进：从 threading.local 到 contextvars

| 技术 | 支持线程 | 支持协程 | Python 版本 |
|------|---------|---------|------------|
| `threading.local` | ✅ | ❌ | 2.0+ |
| `contextvars.ContextVar` | ✅ | ✅ | 3.7+ |

**为什么 contextvars 更好？**

```
调用链视角：
┌─────────────────────────────────────────────────────────┐
│  线程 A                                                    │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │ 协程 Task1  │    │ 协程 Task2  │    │ 协程 Task3  │  │
│  │  ContextVar │    │  ContextVar │    │  ContextVar │  │
│  │   独立副本   │    │   独立副本   │    │   独立副本   │  │
│  └─────────────┘    └─────────────┘    └─────────────┘  │
└─────────────────────────────────────────────────────────┘

threading.local: 整个线程只有一份副本 → 协程之间会串
contextvars: 每个任务(协程)有独立副本 → 真正的协程安全
```

---

## 3. 全局代理对象定义

查看 `src/flask/globals.py`：

### 3.1 核心定义

```python
# src/flask/globals.py

from contextvars import ContextVar
from werkzeug.local import LocalProxy

# 唯一的 ContextVar，存储当前的 AppContext
_cv_app: ContextVar[AppContext] = ContextVar("flask.app_ctx")

# ============================================
# 应用上下文相关的代理对象
# ============================================

# 直接代理整个 AppContext 对象
app_ctx: AppContextProxy = LocalProxy(
    _cv_app, unbound_message=_no_app_msg
)

# 代理 AppContext 的 app 属性
current_app: FlaskProxy = LocalProxy(
    _cv_app, "app", unbound_message=_no_app_msg
)

# 代理 AppContext 的 g 属性
g: _AppCtxGlobalsProxy = LocalProxy(
    _cv_app, "g", unbound_message=_no_app_msg
)

# ============================================
# 请求上下文相关的代理对象
# ============================================

# 代理 AppContext 的 request 属性
request: RequestProxy = LocalProxy(
    _cv_app, "request", unbound_message=_no_req_msg
)

# 代理 AppContext 的 session 属性
session: SessionMixinProxy = LocalProxy(
    _cv_app, "session", unbound_message=_no_req_msg
)
```

### 3.2 关键洞察：单一 ContextVar

注意：Flask 3.2 之后 **只有一个 `_cv_app` ContextVar**！

```
旧版 (Flask < 3.2):
┌─────────────────┐     ┌─────────────────┐
│ _cv_app (AppCtx)│     │ _cv_req (ReqCtx)│  ← 两个独立的 ContextVar
└─────────────────┘     └─────────────────┘

新版 (Flask ≥ 3.2):
┌─────────────────────────────────────────┐
│         _cv_app (AppContext)            │
│  ┌───────────────────────────────────┐  │
│  │ .app      → Flask 实例            │  │
│  │ .g        → 请求级全局存储        │  │
│  │ .request  → Request 对象 (可选)   │  │
│  │ .session  → Session 对象 (可选)   │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
                    ↑
           单一 ContextVar，统一管理
```

**设计优势：**
1. 简化实现：不再需要维护两个独立的上下文栈
2. 统一管理：AppContext 既可以包含请求信息，也可以不包含
3. 用 `has_request` 属性区分：
   ```python
   @property
   def has_request(self) -> bool:
       return self._request is not None
   ```

---

## 4. AppContext 上下文实现

查看 `src/flask/ctx.py`：

### 4.1 类结构概览

```python
# src/flask/ctx.py

class AppContext:
    """统一的上下文类，包含应用信息和（可选的）请求信息"""
    
    def __init__(
        self,
        app: Flask,
        *,
        request: Request | None = None,
        session: SessionMixin | None = None,
    ) -> None:
        # ========== 核心数据 ==========
        self.app = app              # Flask 应用实例
        self.g = app.app_ctx_globals_class()  # 请求级全局存储
        
        # ========== 请求相关数据（可选） ==========
        self._request: Request | None = request
        self._session: SessionMixin | None = session
        
        # ========== 生命周期管理 ==========
        self._cv_token: contextvars.Token[AppContext] | None = None
        self._push_count: int = 0  # 支持嵌套推入
```

### 4.2 _AppCtxGlobals (g 对象)

```python
class _AppCtxGlobals:
    """g 对象的实际类型：一个简单的命名空间对象"""
    
    def __getattr__(self, name: str) -> t.Any:
        try:
            return self.__dict__[name]
        except KeyError:
            raise AttributeError(name) from None
    
    def __setattr__(self, name: str, value: t.Any) -> None:
        self.__dict__[name] = value
    
    # 还支持 dict 风格的操作：
    # - g.get('key', default)
    # - g.pop('key')
    # - g.setdefault('key', value)
    # - 'key' in g
    # - for key in g: ...
```

**使用方式：**
```python
from flask import g

@app.before_request
def load_user():
    user_id = session.get('user_id')
    g.user = User.query.get(user_id) if user_id else None

@app.route('/profile')
def profile():
    return f"Hello, {g.user.name}"  # 同一请求内共享
```

### 4.3 push() 方法：推入上下文

```python
def push(self) -> None:
    """将此上下文设为当前活跃上下文"""
    
    self._push_count += 1
    
    # 如果已经推入过（嵌套场景），直接返回
    if self._cv_token is not None:
        return
    
    # ========== 核心：设置 ContextVar ==========
    # _cv_app.set() 返回一个 Token，用于后续恢复
    self._cv_token = _cv_app.set(self)
    
    # 发送信号：appcontext_pushed
    appcontext_pushed.send(self.app, _async_wrapper=self.app.ensure_sync)
    
    # ========== 如果是请求上下文 ==========
    if self._request is not None:
        # 1. 懒加载 session
        self._get_session()
        
        # 2. 执行 URL 路由匹配
        if self.url_adapter is not None:
            self.match_request()
```

**关键：ContextVar.set() 的返回值**

```python
# ContextVar 的工作方式
from contextvars import ContextVar

cv = ContextVar("my_var")

# 设置新值，返回 Token（记录旧状态）
token = cv.set("new_value")

# 获取当前值
assert cv.get() == "new_value"

# 恢复到设置前的状态
cv.reset(token)
```

这就是 **栈式管理** 的基础！

### 4.4 pop() 方法：弹出上下文

```python
def pop(self, exc: BaseException | None = None) -> None:
    """弹出上下文，执行清理工作"""
    
    # 安全检查
    if self._cv_token is None:
        raise RuntimeError(f"Cannot pop this context ({self!r}), it is not pushed.")
    
    # 支持嵌套推入：只有 _push_count 归零时才真正清理
    self._push_count -= 1
    if self._push_count > 0:
        return
    
    # ========== 执行清理回调 ==========
    collect_errors = _CollectErrors()
    
    # 1. 请求级别的 teardown（如果有请求）
    if self._request is not None:
        with collect_errors:
            self.app.do_teardown_request(self, exc)
        with collect_errors:
            self._request.close()
    
    # 2. 应用级别的 teardown
    with collect_errors:
        self.app.do_teardown_appcontext(self, exc)
    
    # ========== 恢复 ContextVar ==========
    _cv_app.reset(self._cv_token)
    self._cv_token = None  # 标记已弹出
    
    # 发送信号：appcontext_popped
    with collect_errors:
        appcontext_popped.send(self.app, _async_wrapper=self.app.ensure_sync)
    
    collect_errors.raise_any("Errors during context teardown")
```

### 4.5 上下文管理器协议

```python
def __enter__(self) -> te.Self:
    self.push()
    return self

def __exit__(
    self,
    exc_type: type[BaseException] | None,
    exc_value: BaseException | None,
    tb: TracebackType | None,
) -> None:
    self.pop(exc_value)
```

这让我们可以使用 `with` 语句：

```python
with app.app_context():
    # current_app, g 现在可用
    print(current_app.config['DEBUG'])
    g.some_data = 'value'
# 退出 with 块后，上下文自动弹出
```

---

## 5. LocalProxy 透明代理原理

### 5.1 什么是透明代理？

**使用方式：**
```python
from flask import request, current_app

# 看起来像普通对象
print(request.method)      # GET
print(request.args.get('id'))  # '123'
print(current_app.config)  # {...}
```

**本质：** `request`, `current_app` 都不是真正的 Request/Flask 对象，而是 **LocalProxy 代理对象**。

### 5.2 LocalProxy 的核心实现

根据 Werkzeug 源码分析，`LocalProxy` 通过**魔术方法重载**实现透明代理：

```python
# Werkzeug local.py 核心逻辑（简化版）

class LocalProxy:
    def __init__(self, local, name=None, *, unbound_message=None):
        # local 可以是 ContextVar、函数、Local 对象等
        self._local = local
        self._name = name
        self._unbound_message = unbound_message
    
    def _get_current_object(self):
        """获取当前实际绑定的对象（核心方法）"""
        
        # 情况1：如果是 ContextVar
        if isinstance(self._local, ContextVar):
            obj = self._local.get()  # 从当前上下文获取值
            if self._name is not None:
                return getattr(obj, self._name)  # 访问指定属性
            return obj
        
        # 情况2：如果是可调用对象（老版本用法）
        if callable(self._local):
            return self._local()
        
        # 情况3：如果是 Local 对象
        return getattr(self._local, self._name)
    
    # ============================================
    # 代理所有属性访问
    # ============================================
    
    def __getattr__(self, name):
        """访问属性时转发"""
        if name == '__members__':
            return dir(self._get_current_object())
        return getattr(self._get_current_object(), name)
    
    def __setattr__(self, name, value):
        """设置属性时转发"""
        # 跳过代理自己的私有属性
        if name.startswith('_'):
            return object.__setattr__(self, name, value)
        setattr(self._get_current_object(), name, value)
    
    def __delattr__(self, name):
        """删除属性时转发"""
        delattr(self._get_current_object(), name)
    
    # ============================================
    # 代理所有运算符和特殊方法
    # ============================================
    
    def __repr__(self):
        try:
            return repr(self._get_current_object())
        except RuntimeError:
            return f'<LocalProxy unbound>'
    
    def __bool__(self):
        try:
            return bool(self._get_current_object())
        except RuntimeError:
            return False
    
    def __call__(self, *args, **kwargs):
        return self._get_current_object()(*args, **kwargs)
    
    # ... 还有更多魔术方法：
    # __str__, __hash__, __eq__, __ne__, __lt__, __le__, __gt__, __ge__
    # __add__, __sub__, __mul__, __truediv__, ...
    # __getitem__, __setitem__, __delitem__, __iter__, __len__, ...
    # __enter__, __exit__, ...
```

### 5.3 动态查找演示

```
访问 request.method 的执行流程：

1. request.method
   ↓
2. LocalProxy.__getattr__('method')
   ↓
3. self._get_current_object()
   ↓
4. _cv_app.get()  → 获取当前线程/协程的 AppContext
   ↓
5. getattr(ctx, 'request')  → 访问 AppContext.request 属性
   ↓
6. 返回真正的 Request 对象
   ↓
7. getattr(request_obj, 'method')  → 返回 'GET'
```

**关键点：每次访问都会重新查找！**

```python
# 这不是一次赋值，而是每次动态计算
request = LocalProxy(_cv_app, 'request')

# 每次访问都执行：
# _cv_app.get().request
# 不同线程/协程的 _cv_app.get() 返回不同的 AppContext！
```

### 5.4 版本对比：函数 vs ContextVar

**老版本 (Flask < 2.2 风格)：**
```python
# 使用 lambda 函数，每次调用时从栈中获取
from werkzeug.local import LocalStack

_app_ctx_stack = LocalStack()

current_app = LocalProxy(lambda: _app_ctx_stack.top.app)
request = LocalProxy(lambda: _app_ctx_stack.top.request)
```

**新版本 (Flask ≥ 2.2 风格)：**
```python
# 直接使用 ContextVar，更高效
from contextvars import ContextVar

_cv_app = ContextVar("flask.app_ctx")

current_app = LocalProxy(_cv_app, "app")
request = LocalProxy(_cv_app, "request")
```

**优势：**
- 性能更好：ContextVar 是 Python 内置实现，比 LocalStack 更快
- 语义更清晰：直接表达"上下文变量"的概念
- 协程安全：contextvars 原生支持 asyncio

---

## 6. 上下文生命周期管理

### 6.1 WSGI 请求处理流程

查看 `src/flask/app.py` 的 `wsgi_app` 方法：

```python
def wsgi_app(
    self, environ: WSGIEnvironment, start_response: StartResponse
) -> cabc.Iterable[bytes]:
    """WSGI 应用入口，每个请求调用一次"""
    
    # ========== 1. 创建上下文 ==========
    ctx = self.request_context(environ)
    error: BaseException | None = None
    
    try:
        try:
            # ========== 2. 推入上下文 ==========
            ctx.push()
            
            # ========== 3. 处理请求 ==========
            response = self.full_dispatch_request(ctx)
        
        except Exception as e:
            error = e
            response = self.handle_exception(ctx, e)
        except:
            error = sys.exc_info()[1]
            raise
        
        # ========== 4. 返回响应 ==========
        return response(environ, start_response)
    
    finally:
        # ========== 5. 弹出上下文（无论成功失败）==========
        ctx.pop(error)
```

### 6.2 完整生命周期时序图

```
请求到达
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  wsgi_app(environ, start_response)                       │
│                                                           │
│  1. ctx = self.request_context(environ)                  │
│     └─► 创建 AppContext，包含 Request 对象               │
│                                                           │
│  2. ctx.push()                                            │
│     ├─► _cv_token = _cv_app.set(ctx)  ◄── 关键！        │
│     ├─► 发送 appcontext_pushed 信号                      │
│     ├─► 加载 session                                      │
│     └─► 路由匹配 (match_request)                          │
│                                                           │
│  ┌─────────────────────────────────────────────────┐     │
│  │  3. full_dispatch_request(ctx)                   │     │
│  │     ├─► 发送 request_started 信号                │     │
│  │     ├─► preprocess_request (before_request)     │     │
│  │     ├─► dispatch_request → 执行视图函数          │     │
│  │     │         └─► request, current_app, g,     │     │
│  │     │             session 都可正常使用！         │     │
│  │     ├─► process_response (after_request)        │     │
│  │     └─► 发送 request_finished 信号               │     │
│  └─────────────────────────────────────────────────┘     │
│                                                           │
│  4. return response(environ, start_response)             │
│     └─► 返回响应迭代器给 WSGI 服务器                      │
│                                                           │
│  5. finally: ctx.pop(error)                              │
│     ├─► 执行 teardown_request                            │
│     ├─► 执行 teardown_appcontext                         │
│     ├─► _cv_app.reset(_cv_token)  ◄── 关键！恢复上下文  │
│     └─► 发送 appcontext_popped 信号                      │
└─────────────────────────────────────────────────────────┘
    │
    ▼
请求结束
```

### 6.3 手动创建上下文的场景

**场景1：测试或脚本中使用 app_context**
```python
from flask import Flask, current_app

app = Flask(__name__)

# ❌ 错误：没有上下文
# print(current_app.config)  # RuntimeError!

# ✅ 正确：手动推入上下文
with app.app_context():
    print(current_app.config)  # 正常工作
    # current_app 指向上面的 app
```

**场景2：测试请求相关功能**
```python
with app.test_request_context('/api/user?id=123', method='GET'):
    print(request.path)       # '/api/user'
    print(request.args)       # ImmutableMultiDict([('id', '123')])
    print(request.method)     # 'GET'
```

### 6.4 嵌套上下文

`_push_count` 允许同一个上下文被多次推入：

```python
ctx = app.app_context()

ctx.push()        # _push_count = 1
print(current_app)  # 可用

ctx.push()        # _push_count = 2，实际不重复推入
print(current_app)  # 仍可用

ctx.pop()         # _push_count = 1，不真正弹出
print(current_app)  # 仍可用！

ctx.pop()         # _push_count = 0，真正弹出
# print(current_app)  # RuntimeError!
```

**设计目的：**
- 支持在已有的上下文中再次推入相同上下文
- 例如：在某些调试场景或中间件中可能需要

---

## 7. 为什么并发时不会"串"数据？

### 7.1 核心原理：ContextVar 的隔离性

```python
# 模拟两个并发请求
from contextvars import ContextVar
import threading
import time

cv = ContextVar("request_id")

def process_request(request_id):
    # 设置当前请求的 id
    token = cv.set(request_id)
    try:
        # 模拟处理时间
        time.sleep(0.1)
        # 读取当前请求的 id
        print(f"Thread {threading.current_thread().name}: request_id = {cv.get()}")
    finally:
        cv.reset(token)

# 启动两个线程
t1 = threading.Thread(target=process_request, args=(1,), name="T1")
t2 = threading.Thread(target=process_request, args=(2,), name="T2")

t1.start()
t2.start()
t1.join()
t2.join()

# 输出（顺序可能不同，但值不会串）：
# Thread T1: request_id = 1
# Thread T2: request_id = 2
```

**关键：每个线程的 `cv.get()` 返回的是该线程 `cv.set()` 设置的值！**

### 7.2 Flask 的实际隔离机制

```
并发模型：多线程服务器（Werkzeug dev server, Gunicorn sync worker）

┌─────────────────────────────────────────────────────────────────┐
│                         WSGI 服务器                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌───────────────────┐    ┌───────────────────┐                 │
│  │    线程 A         │    │    线程 B         │                 │
│  │  (处理请求 1)      │    │  (处理请求 2)      │                 │
│  │                   │    │                   │                 │
│  │ _cv_app 的独立副本 │    │ _cv_app 的独立副本 │                 │
│  │                   │    │                   │                 │
│  │ ┌───────────────┐ │    │ ┌───────────────┐ │                 │
│  │ │ AppContext 1  │ │    │ │ AppContext 2  │ │                 │
│  │ │  ├─► app      │ │    │ │  ├─► app      │ │                 │
│  │ │  ├─► g (请求1)│ │    │ │  ├─► g (请求2)│ │                 │
│  │ │  ├─► request1 │ │    │ │  ├─► request2 │ │                 │
│  │ │  └─► session1 │ │    │ │  └─► session2 │ │                 │
│  │ └───────────────┘ │    │ └───────────────┘ │                 │
│  │                   │    │                   │                 │
│  │ request.method    │    │ request.method    │                 │
│  │   = 'GET' (请求1) │    │   = 'POST' (请求2)│                 │
│  │ g.user = UserA    │    │ g.user = UserB    │                 │
│  └───────────────────┘    └───────────────────┘                 │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 7.3 协程场景下的隔离

```python
# 使用 asyncio 的场景（Uvicorn, Hypercorn）
import asyncio
from contextvars import ContextVar

cv = ContextVar("request_id")

async def process_request(request_id):
    token = cv.set(request_id)
    try:
        # 模拟 IO 等待，此时会切换到其他协程
        await asyncio.sleep(0.1)
        # 即使切换过，cv.get() 仍然是本协程的值！
        print(f"Task {asyncio.current_task().get_name()}: request_id = {cv.get()}")
    finally:
        cv.reset(token)

async def main():
    # 并发执行两个协程
    await asyncio.gather(
        process_request(1),
        process_request(2)
    )

asyncio.run(main())

# 输出：
# Task Task-2: request_id = 1
# Task Task-3: request_id = 2
```

### 7.4 内存模型对比

| 存储方式 | 作用域 | 多线程 | 多协程 |
|---------|--------|--------|--------|
| 全局变量 | 整个进程 | ❌ 共享，会串 | ❌ 共享，会串 |
| `threading.local` | 单个线程 | ✅ 隔离 | ❌ 线程内协程共享 |
| `contextvars.ContextVar` | 单个任务（线程/协程） | ✅ 隔离 | ✅ 隔离 |

---

## 8. 总结与架构图

### 8.1 核心组件总结

```
┌────────────────────────────────────────────────────────────────────┐
│                         用户代码层                                   │
│  from flask import request, current_app, g, session                │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  这些看似"全局"的变量，实际上每次访问都在动态查找！            │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────┐
│                      LocalProxy 代理层                              │
│                                                                     │
│  request = LocalProxy(_cv_app, "request")                          │
│  current_app = LocalProxy(_cv_app, "app")                          │
│  g = LocalProxy(_cv_app, "g")                                      │
│  session = LocalProxy(_cv_app, "session")                          │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  工作原理：                                                    │  │
│  │  1. 重载所有魔术方法 (__getattr__, __setattr__, 等)          │  │
│  │  2. 每次访问时调用 _get_current_object()                      │  │
│  │  3. 从 ContextVar 获取当前上下文的属性                        │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────┐
│                    ContextVar 存储层                                │
│                                                                     │
│  _cv_app: ContextVar[AppContext] = ContextVar("flask.app_ctx")   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  核心特性：                                                    │  │
│  │  - 每个线程/协程有独立的副本                                  │  │
│  │  - set() 返回 Token，用于 reset() 恢复                      │  │
│  │  - 栈式管理，支持嵌套                                        │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────┐
│                    AppContext 业务层                                │
│                                                                     │
│  class AppContext:                                                  │
│      app: Flask              # 应用实例                           │
│      g: _AppCtxGlobals      # 请求级全局存储                      │
│      _request: Request | None  # 请求对象（可选）                 │
│      _session: SessionMixin | None  # 会话（可选）                │
│                                                                     │
│      push()   → _cv_app.set(self)                                 │
│      pop()    → _cv_app.reset(token)                              │
│      __enter__/__exit__ → 支持 with 语句                          │
└────────────────────────────────────────────────────────────────────┘
```

### 8.2 关键设计决策

| 决策点 | 设计选择 | 原因 |
|--------|---------|------|
| 上下文存储 | `contextvars.ContextVar` | 线程+协程双安全，Python 3.7+ 标准 |
| 代理方式 | `LocalProxy` 透明代理 | 保持 API 简洁，像全局变量一样使用 |
| 上下文合并 | 单一 `AppContext` | Flask 3.2 合并了 AppContext 和 RequestContext，简化实现 |
| 生命周期 | `push()` / `pop()` + 上下文管理器 | 确保资源正确释放，支持嵌套 |
| 错误处理 | `try...finally` 保证弹出 | 无论请求成功失败，上下文都能正确清理 |

### 8.3 使用建议

1. **理解"看似全局，实则局部"**
   ```python
   # 每次访问都是动态查找
   request.method  # 不是一次赋值，而是每次从当前上下文获取
   ```

2. **需要真实对象时使用 `_get_current_object()`**
   ```python
   # 信号发送时需要真实对象，不是代理
   from flask import current_app
   
   # ❌ 可能有问题：some_signal.send(current_app)
   # ✅ 正确：获取真实对象
   app = current_app._get_current_object()
   some_signal.send(app)
   ```

3. **测试时手动管理上下文**
   ```python
   # 应用上下文
   with app.app_context():
       # 使用 current_app, g
   
   # 请求上下文
   with app.test_request_context('/api', method='POST'):
       # 使用 request, session
   ```

4. **跨线程/协程传递上下文**
   ```python
   from flask import copy_current_request_context
   
   @app.route('/')
   def index():
       @copy_current_request_context
       def background_task():
           # 后台任务中也能访问 request
           print(request.path)
       
       # 在新线程中执行
       import threading
       threading.Thread(target=background_task).start()
       return 'OK'
   ```

---

## 附录：相关文件位置

| 文件 | 路径 | 说明 |
|------|------|------|
| 全局代理定义 | `src/flask/globals.py` | `request`, `current_app`, `g`, `session` |
| 上下文实现 | `src/flask/ctx.py` | `AppContext`, `_AppCtxGlobals` |
| WSGI 处理 | `src/flask/app.py` | `wsgi_app()`, 上下文生命周期管理 |
| 官方文档 | `docs/appcontext.rst` | 应用和请求上下文文档 |
| 测试用例 | `tests/test_reqctx.py` | 上下文功能测试 |

---

*报告生成日期：2026-04-26*
*基于 Flask 源码版本：当前开发版（≥ 3.2，已合并 AppContext 和 RequestContext）*
