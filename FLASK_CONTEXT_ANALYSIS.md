# Flask 上下文机制深度分析

## 目录

1. [问题背景：为什么需要上下文？](#1-问题背景为什么需要上下文)
2. [核心技术栈概览](#2-核心技术栈概览)
3. [全局代理对象定义](#3-全局代理对象定义)
4. [AppContext 上下文实现](#4-appcontext-上下文实现)
   - [4.5 Flask 3.2 之前：两套独立的上下文](#45-flask-32-之前两套独立的上下文)
   - [4.6 Teardown 回调机制详解](#46-teardown-回调机制详解)
   - [4.7 Token Reset 的异常清理保证机制](#47-token-reset-的异常清理保证机制)
   - [4.8 Blinker 信号系统详解](#48-blinker-信号系统详解)
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

## 4.5 Flask 3.2 之前：两套独立的上下文

### 4.5.1 历史背景：为什么有两套上下文？

在 Flask 3.2 之前，存在两个独立的上下文类：

| 上下文类型 | 类名 | 作用域 | 核心数据 |
|-----------|------|--------|---------|
| **应用上下文** | `AppContext` | 应用级别或请求级别 | `current_app`, `g` |
| **请求上下文** | `RequestContext` | 单次 HTTP 请求 | `request`, `session` |

**设计初衷：**
1. **应用上下文**：可以独立于请求存在，用于 CLI 命令、离线脚本等场景
2. **请求上下文**：必须与 HTTP 请求绑定，包含请求特定的数据

### 4.5.2 合并前的协作逻辑

**关键设计：push RequestContext 会自动带出 AppContext**

```
请求处理流程（Flask < 3.2）：

1. wsgi_app() 被调用
        ↓
2. 创建 RequestContext
        ↓
3. RequestContext.push() 执行
        ↓
   ┌─────────────────────────────────────────┐
   │  检查：是否已有 AppContext 存在？        │
   │         ↓                               │
   │    ┌─────────┐     ┌──────────────┐   │
   │   │  不存在  │    │    存在       │   │
   │    ↓         │     ↓              │   │
   │  创建并      │    检查：当前      │   │
   │  push        │    AppContext 是否 │   │
   │  AppContext  │    属于同一个 app? │   │
   │              │    ┌─────────┐     │   │
   │              │   │  是     │ 否  │   │
   │              │   │  跳过   │ ↓   │   │
   │              │   │         │push │   │
   │              │   │         │新的 │   │
   │              │   │         │AppCtx│   │
   └──────────────┴───┴─────────┴─────┴───┘
        ↓
4. 继续 RequestContext.push 的后续逻辑
   - 加载 session
   - 路由匹配
```

**伪代码示意（Flask < 3.2 的 RequestContext.push）：**

```python
# Flask 3.2 之前的实现逻辑
def push(self):
    # 1. 先处理应用上下文
    app_ctx = _cv_app.get(None)
    
    if app_ctx is None:
        # 没有应用上下文，创建一个新的
        app_ctx = self.app.app_context()
        app_ctx.push()
        self._implicit_app_ctx_stack.append(app_ctx)
    else:
        # 已有应用上下文，检查是否属于同一个 app
        if app_ctx.app is not self.app:
            # 不同的 app，需要 push 新的
            app_ctx = self.app.app_context()
            app_ctx.push()
            self._implicit_app_ctx_stack.append(app_ctx)
    
    # 2. 再推入请求上下文
    self._cv_token = _cv_request.set(self)
    
    # 3. 加载 session、路由匹配等
    # ...
```

### 4.5.3 pop 时的协作逻辑

```
请求结束时的弹出顺序：

RequestContext.pop() 执行
        ↓
   ┌──────────────────────────────┐
   │  1. 执行 teardown_request    │
   │  2. 发送 request_tearing_down │
   │  3. _cv_request.reset(token) │
   └──────────────────────────────┘
        ↓
   ┌──────────────────────────────┐
   │  检查：_implicit_app_ctx_stack │
   │  是否有"隐式"推入的 AppContext？│
   │         ↓                     │
   │    ┌──────────┐  ┌────────┐ │
   │   │   有      │ │  无    │ │
   │   │   ↓       │ │  跳过  │ │
   │   │ 逐个 pop  │ │        │ │
   │   │ AppContext │ │        │ │
   │   └──────────┘ │ └────────┘ │
   └──────────────────────────────┘
        ↓
请求上下文和关联的应用上下文都已弹出
```

### 4.5.4 为什么要在 3.2 合并？

查看 `CHANGES.rst` 中的变更说明：

```
Version 3.2.0
-------------
- ``RequestContext`` has merged with ``AppContext``. ``RequestContext`` is now
  a deprecated alias. If an app context is already pushed, it is not reused
  when dispatching a request. This greatly simplifies the internal code for tracking
  the active context. :issue:`5639`
```

**合并的优势：**

| 方面 | 合并前 | 合并后 |
|------|--------|--------|
| 复杂度 | 两套栈、两套 push/pop 逻辑、隐式追踪 | 单一 ContextVar，逻辑清晰 |
| 嵌套处理 | 需要 `_implicit_app_ctx_stack` 追踪"隐式"推入的上下文 | 用 `_push_count` 简单计数 |
| 代码量 | 需要维护两个类和它们的协作 | 单一 `AppContext` 类 |
| 理解成本 | 开发者需要理解两套上下文的关系 | 只需理解 `has_request` 属性 |

---

## 4.6 Teardown 回调机制详解

### 4.6.1 回调的注册方式

**两种 teardown 回调：**

| 装饰器 | 触发时机 | 用途 |
|--------|---------|------|
| `@app.teardown_request` | 请求上下文弹出时 | 清理请求相关资源（数据库连接等） |
| `@app.teardown_appcontext` | 应用上下文弹出时 | 清理应用级资源 |

**注册示例：**

```python
from flask import Flask, g
import sqlite3

app = Flask(__name__)

# ========== teardown_request：请求级别清理 ==========
@app.teardown_request
def close_db_connection(exc):
    """请求结束时关闭数据库连接"""
    db = getattr(g, '_database', None)
    if db is not None:
        db.close()

# ========== teardown_appcontext：应用级别清理 ==========
@app.teardown_appcontext
def cleanup_app_resource(exc):
    """应用上下文结束时的清理"""
    # 例如：关闭连接池、释放内存等
    pass
```

### 4.6.2 内部存储结构

查看 `src/flask/sansio/scaffold.py` 和 `src/flask/sansio/app.py`：

```python
# ========== teardown_request 的存储 ==========
# 支持按蓝图分组（None 表示应用级别）
self.teardown_request_funcs: dict[
    str | None, list[ft.TeardownCallable]
] = {None: []}  # None 是应用级别的 key

# 注册时
@app.teardown_request
def my_func(exc):
    pass
# 等价于：
self.teardown_request_funcs.setdefault(None, []).append(my_func)

# 蓝图级别的 teardown_request 会有不同的 key
@bp.teardown_request
def blueprint_teardown(exc):
    pass
# 存储在 teardown_request_funcs[blueprint_name] 中


# ========== teardown_appcontext 的存储 ==========
# 只有应用级别，不区分蓝图
self.teardown_appcontext_funcs: list[ft.TeardownCallable] = []

# 注册时
@app.teardown_appcontext
def my_func(exc):
    pass
# 等价于：
self.teardown_appcontext_funcs.append(my_func)
```

### 4.6.3 触发时机与执行顺序

**在 `AppContext.pop()` 中的触发逻辑：**

```python
# src/flask/ctx.py 的 pop() 方法

def pop(self, exc: BaseException | None = None) -> None:
    # ... 前置检查和 _push_count 处理 ...
    
    collect_errors = _CollectErrors()
    
    # ========== 1. 先执行 teardown_request（如果有请求）==========
    if self._request is not None:
        with collect_errors:
            self.app.do_teardown_request(self, exc)
        with collect_errors:
            self._request.close()
    
    # ========== 2. 再执行 teardown_appcontext ==========
    with collect_errors:
        self.app.do_teardown_appcontext(self, exc)
    
    # ========== 3. 恢复 ContextVar ==========
    _cv_app.reset(self._cv_token)
    self._cv_token = None
    
    # ... 发送信号 ...
```

**`do_teardown_request` 的详细实现：**

```python
# src/flask/app.py

def do_teardown_request(
    self, ctx: AppContext, exc: BaseException | None = None
) -> None:
    collect_errors = _CollectErrors()
    
    # 执行顺序：蓝图级别 → 应用级别
    # ctx.request.blueprints 是蓝图名称列表（按嵌套顺序）
    for name in chain(ctx.request.blueprints, (None,)):
        if name in self.teardown_request_funcs:
            # 反向执行：后注册的先执行（LIFO）
            for func in reversed(self.teardown_request_funcs[name]):
                with collect_errors:
                    self.ensure_sync(func)(exc)  # 传入异常参数
    
    # 发送信号
    with collect_errors:
        request_tearing_down.send(self, _async_wrapper=self.ensure_sync, exc=exc)
    
    collect_errors.raise_any("Errors during request teardown")
```

**`do_teardown_appcontext` 的实现：**

```python
def do_teardown_appcontext(
    self, ctx: AppContext, exc: BaseException | None = None
) -> None:
    collect_errors = _CollectErrors()
    
    # 反向执行：后注册的先执行
    for func in reversed(self.teardown_appcontext_funcs):
        with collect_errors:
            self.ensure_sync(func)(exc)
    
    # 发送信号
    with collect_errors:
        appcontext_tearing_down.send(self, _async_wrapper=self.ensure_sync, exc=exc)
    
    collect_errors.raise_any("Errors during app teardown")
```

### 4.6.4 异常参数 `exc` 的传递规则

| 场景 | `exc` 的值 |
|------|-----------|
| 请求正常完成，无异常 | `None` |
| 请求中抛出异常，但被 `@app.errorhandler` 处理 | `None`（已处理的异常不传递） |
| 请求中抛出未处理的异常 | 异常对象本身 |

**设计意图：**
- 让 teardown 函数知道"是否因异常而结束"
- 可以根据情况执行不同的清理逻辑（如回滚事务）

```python
@app.teardown_request
def cleanup_session(exc):
    if exc is not None:
        # 有异常，回滚事务
        db.session.rollback()
    else:
        # 正常结束，提交事务
        db.session.commit()
    db.session.remove()
```

### 4.6.5 Flask 3.2 的重要改进

查看 `CHANGES.rst`：

```
- All teardown callbacks are called, even if any raise an error. :pr:`5928`
```

**改进对比：**

| 行为 | Flask < 3.2 | Flask ≥ 3.2 |
|------|-------------|-------------|
| 某个 teardown 抛出异常 | 立即停止，后续回调不执行 | 继续执行所有回调，最后汇总抛出 |

**实现方式：`_CollectErrors` 上下文管理器**

```python
# 伪代码示意
class _CollectErrors:
    def __init__(self):
        self.errors = []
    
    def __enter__(self):
        return self
    
    def __exit__(self, exc_type, exc_value, tb):
        if exc_value is not None:
            # 捕获异常，不继续抛出
            self.errors.append(exc_value)
            return True  # 抑制异常
    
    def raise_any(self, message):
        if self.errors:
            # 所有回调执行完后，汇总抛出
            raise ExceptionGroup(message, self.errors)
```

**使用方式：**

```python
collect_errors = _CollectErrors()

# 每个回调都在 collect_errors 上下文中执行
with collect_errors:
    func1(exc)  # 即使抛异常，也会被捕获

with collect_errors:
    func2(exc)  # 继续执行

# 最后检查是否有错误
collect_errors.raise_any("Errors during teardown")
```

---

## 4.7 Token Reset 的异常清理保证机制

### 4.7.1 为什么需要保证？

如果在请求处理过程中发生异常，上下文必须被正确弹出，否则：
- ContextVar 会"泄露"到后续请求
- 资源无法释放
- 后续请求可能读到前一个请求的数据

### 4.7.2 多层保证机制

Flask 通过 **三层防御** 确保上下文正确清理：

```
┌─────────────────────────────────────────────────────────────────┐
│  第一层：try...finally 在 wsgi_app() 中                          │
│                                                                   │
│  def wsgi_app(self, environ, start_response):                    │
│      ctx = self.request_context(environ)                         │
│      error = None                                                 │
│      try:                                                         │
│          try:                                                     │
│              ctx.push()                                            │
│              response = self.full_dispatch_request(ctx)          │
│          except Exception as e:                                   │
│              error = e                                            │
│              response = self.handle_exception(ctx, e)            │
│          return response(environ, start_response)                │
│      finally:                                                     │
│          # 无论成功失败，一定会执行！                             │
│          ctx.pop(error)  ◄────────────────────────────────────┐ │
│      └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  第二层：ContextVar.set() + reset() 的 Token 机制               │
│                                                                   │
│  def push(self):                                                  │
│      # set() 返回一个 Token，记录"设置前的状态"                  │
│      self._cv_token = _cv_app.set(self)                         │
│                      ↓                                           │
│              ┌─────────────────────┐                              │
│              │ Token 包含：         │                              │
│              │ - var: ContextVar   │                              │
│              │ - old_value: 旧值   │                              │
│              └─────────────────────┘                              │
│                                                                   │
│  def pop(self, exc):                                              │
│      # 用 Token 恢复到设置前的状态                                │
│      _cv_app.reset(self._cv_token)  ◄─────────────────────────┐ │
│      self._cv_token = None                                      │ │
│  └──────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  第三层：上下文管理器协议（__enter__ / __exit__）               │
│                                                                   │
│  手动使用时：                                                      │
│                                                                   │
│  with app.app_context():     # 调用 __enter__ → push()         │
│      # 使用 current_app                                          │
│      pass                                                         │
│  # 退出 with 块时，自动调用 __exit__ → pop()                    │
│  # 即使块内抛异常，__exit__ 也会执行！                          │
└─────────────────────────────────────────────────────────────────┘
```

### 4.7.3 Token 机制的工作原理

**ContextVar 的核心 API：**

```python
from contextvars import ContextVar

cv = ContextVar("my_var", default="initial")

# ========== 基础用法 ==========
token = cv.set("new_value")
assert cv.get() == "new_value"

# reset 恢复到 set() 之前的状态
cv.reset(token)
assert cv.get() == "initial"


# ========== 嵌套场景 ==========
token1 = cv.set("value1")
assert cv.get() == "value1"

token2 = cv.set("value2")  # 嵌套设置
assert cv.get() == "value2"

cv.reset(token2)  # 恢复到 value1
assert cv.get() == "value1"

cv.reset(token1)  # 恢复到 initial
assert cv.get() == "initial"


# ========== 异常场景 ==========
token = cv.set("temporary")
try:
    raise RuntimeError("something went wrong")
finally:
    # 即使抛异常，也要 reset！
    cv.reset(token)

# 外部不受影响
assert cv.get() == "initial"
```

**Token 与栈的等价性：**

```
使用 Token 实现的"栈"效果：

操作序列：              ContextVar 值：        Token 链：
────────────────────────────────────────────────────────────
初始状态               "initial"              (no tokens)
                        ↑
cv.set("A")            "A"                    token1 → "initial"
                        ↑
cv.set("B")            "B"                    token2 → "A"
                        ↑
cv.reset(token2)       "A"                    (token2 已使用)
                        ↑
cv.reset(token1)       "initial"              (token1 已使用)
```

### 4.7.4 Flask 中的嵌套上下文处理

**`_push_count` 计数器的作用：**

```python
class AppContext:
    def __init__(self, ...):
        self._push_count: int = 0
        self._cv_token: ... = None
    
    def push(self):
        self._push_count += 1
        
        # 只有第一次 push 才真正设置 ContextVar
        if self._cv_token is not None:
            return  # 已推入，直接返回
        
        self._cv_token = _cv_app.set(self)
        # ... 其他初始化
    
    def pop(self, exc=None):
        # 只有 _push_count 归零时才真正 reset
        self._push_count -= 1
        if self._push_count > 0:
            return  # 还有嵌套，不真正弹出
        
        # ... 执行 teardown
        
        _cv_app.reset(self._cv_token)
        self._cv_token = None
```

**嵌套场景示例：**

```python
ctx = app.app_context()

# ========== 第一次 push ==========
ctx.push()
# _push_count = 1
# _cv_token 被设置，ContextVar 指向 ctx
assert current_app._get_current_object() is app

# ========== 第二次 push（嵌套）==========
ctx.push()
# _push_count = 2
# _cv_token 已存在，直接返回！
# ContextVar 仍然指向同一个 ctx
assert current_app._get_current_object() is app  # 不变

# ========== 第一次 pop ==========
ctx.pop()
# _push_count = 1
# 大于 0，不执行 reset！
assert current_app._get_current_object() is app  # 仍然可用！

# ========== 第二次 pop ==========
ctx.pop()
# _push_count = 0
# 执行 teardown，reset ContextVar
# current_app 现在不可用
```

### 4.7.5 为什么这种设计是安全的？

| 风险点 | 防护机制 |
|--------|---------|
| 视图函数抛异常 | `wsgi_app` 的 `finally` 保证 `ctx.pop()` 执行 |
| `teardown_request` 抛异常 | `_CollectErrors` 捕获，`reset` 仍会执行 |
| `teardown_appcontext` 抛异常 | 同上 |
| 上下文管理器内抛异常 | `__exit__` 保证 `pop()` 执行 |
| 嵌套推入/弹出 | `_push_count` 确保只在最后一次真正 reset |

**极端场景的保证：**

```python
# 即使所有 teardown 都抛异常，reset 仍然执行！

def pop(self, exc=None):
    # ... _push_count 检查 ...
    
    collect_errors = _CollectErrors()
    
    # 即使这些回调抛异常，也被 collect_errors 捕获
    if self._request is not None:
        with collect_errors:
            self.app.do_teardown_request(self, exc)
        with collect_errors:
            self._request.close()
    
    with collect_errors:
        self.app.do_teardown_appcontext(self, exc)
    
    # ========== 关键：reset 不在 collect_errors 中！ ==========
    # 这意味着即使前面都抛异常，reset 一定执行！
    _cv_app.reset(self._cv_token)  # ← 一定会执行！
    self._cv_token = None
    
    with collect_errors:
        appcontext_popped.send(...)
    
    # 最后才抛出收集到的异常
    collect_errors.raise_any(...)
```

**`reset` 放在 `teardown` 之后的原因：**
- teardown 回调可能需要访问 `current_app`、`request` 等
- 必须在 teardown 全部执行完后再 reset
- 但 `reset` 本身不会失败（只要 token 有效），所以放在 `collect_errors` 之外

---

## 4.8 Blinker 信号系统详解

### 4.8.1 什么是 Blinker？

**Blinker** 是一个轻量级的 Python 信号/事件派发库，提供：
- 命名信号注册表
- 发送者/接收者解耦
- 线程安全
- 异步接收器支持

Flask 从 0.6 版本开始集成 Blinker，用于在请求生命周期的各个节点发送通知。

### 4.8.2 Flask 如何接入 Blinker

**`src/flask/signals.py` 的核心实现：**

```python
# src/flask/signals.py

from blinker import Namespace

# 为 Flask 内置信号创建独立的命名空间
# 这确保 Flask 的信号不会与其他库冲突
_signals = Namespace()

# ========== 定义所有内置信号 ==========

# 模板渲染相关
template_rendered = _signals.signal("template-rendered")
before_render_template = _signals.signal("before-render-template")

# 请求生命周期相关
request_started = _signals.signal("request-started")
request_finished = _signals.signal("request-finished")
request_tearing_down = _signals.signal("request-tearing-down")
got_request_exception = _signals.signal("got-request-exception")

# 应用上下文生命周期相关
appcontext_tearing_down = _signals.signal("appcontext-tearing-down")
appcontext_pushed = _signals.signal("appcontext-pushed")
appcontext_popped = _signals.signal("appcontext-popped")

# 消息闪现相关
message_flashed = _signals.signal("message-flashed")
```

**Namespace 的作用：**
```
┌─────────────────────────────────────────────────────────────────┐
│                    Blinker 信号系统架构                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  全局信号（通过 signal('name') 创建）                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  signal('initialized')                                    │   │
│  │  signal('data-changed')                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
│  Flask 命名空间（隔离）                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Namespace()                                              │   │
│  │  ├─► 'template-rendered'   (template_rendered)          │   │
│  │  ├─► 'request-started'     (request_started)            │   │
│  │  ├─► 'appcontext-pushed'   (appcontext_pushed)          │   │
│  │  └─► ...                                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
│  其他扩展的命名空间（互不干扰）                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  SQLAlchemy Namespace                                     │   │
│  │  └─► 'before_flush', 'after_commit', ...                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 4.8.3 信号监听器的注册方式

**方式 1：`connect()` / `disconnect()` 方法**

```python
from flask import request_started, request_finished, current_app

def on_request_started(sender, **extra):
    """请求开始时调用
    sender: Flask 应用实例
    """
    print(f"Request started for {sender.name}")

def on_request_finished(sender, response, **extra):
    """请求结束时调用
    sender: Flask 应用实例
    response: Response 对象
    """
    print(f"Request finished, status: {response.status}")

# ========== 注册 ==========
# 方式 A：监听所有应用的信号（不推荐，除非你知道你在做什么）
request_started.connect(on_request_started)

# 方式 B：只监听特定应用的信号（推荐！）
app = Flask(__name__)
request_started.connect(on_request_started, sender=app)
request_finished.connect(on_request_finished, sender=app)

# ========== 注销 ==========
request_started.disconnect(on_request_started, sender=app)
request_finished.disconnect(on_request_finished, sender=app)
```

**方式 2：`connect_via()` 装饰器（Blinker 1.1+）**

```python
from flask import template_rendered
from flask import Flask

app = Flask(__name__)

# 使用装饰器直接订阅
@template_rendered.connect_via(app)
def on_template_rendered(sender, template, context, **extra):
    """模板渲染后调用
    template: 模板对象 (Jinja2 Template)
    context: 模板上下文字典
    """
    print(f"Template rendered: {template.name}")
```

**方式 3：`connected_to()` 上下文管理器（临时订阅）**

```python
from flask import template_rendered
from contextlib import contextmanager

@contextmanager
def capture_templates(app):
    """临时捕获渲染的模板，用于测试"""
    recorded = []
    
    def record(sender, template, context, **extra):
        recorded.append((template, context))
    
    # 进入 with 块时连接
    template_rendered.connect(record, sender=app)
    try:
        yield recorded
    finally:
        # 退出 with 块时断开
        template_rendered.disconnect(record, sender=app)

# 或者使用 Blinker 内置的 connected_to（更简洁）
def capture_templates_v2(app, recorded):
    def record(sender, template, context, **extra):
        recorded.append((template, context))
    # 返回一个上下文管理器
    return template_rendered.connected_to(record, sender=app)

# 使用方式
with capture_templates(app) as templates:
    client = app.test_client()
    client.get('/')
    print(f"Rendered {len(templates)} templates")
```

**方式 4：匿名信号的类属性方式**

```python
from blinker import Signal

class MyPlugin:
    """自定义信号的示例"""
    
    # 匿名信号，不通过 Namespace
    on_initialized = Signal()
    on_data_changed = Signal(doc="Called when data changes")
    
    def initialize(self):
        self.on_initialized.send(self)
    
    def update_data(self, new_data):
        self._data = new_data
        self.on_data_changed.send(self, data=new_data)

# 使用
plugin = MyPlugin()

@plugin.on_initialized.connect
def on_init(sender):
    print(f"Plugin {sender} initialized!")

plugin.initialize()  # 触发信号
```

### 4.8.4 信号在 push/pop 中的触发时机

**完整的信号触发时序图：**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      请求生命周期完整时序                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  wsgi_app() 入口                                                          │
│       │                                                                   │
│       ▼                                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  ctx.push()                                                       │   │
│  │    │                                                              │   │
│  │    ├─► _cv_app.set(ctx)  ◄── ContextVar 设置                   │   │
│  │    │                                                              │   │
│  │    ├─► appcontext_pushed.send()  ◄─────── 第 1 个信号         │   │
│  │    │         │                                                    │   │
│  │    │         └─► 此时 current_app 已可用                         │   │
│  │    │                                                              │   │
│  │    ├─► (如果是请求上下文)                                         │   │
│  │    │    ├─► session 加载                                          │   │
│  │    │    └─► 路由匹配 (match_request)                              │   │
│  │    │                                                              │   │
│  └────┴─────────────────────────────────────────────────────────────┘   │
│       │                                                                   │
│       ▼                                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  full_dispatch_request()                                          │   │
│  │    │                                                              │   │
│  │    ├─► request_started.send()  ◄────────── 第 2 个信号         │   │
│  │    │         │                                                    │   │
│  │    │         └─► 在 before_request 之前触发                       │   │
│  │    │                                                              │   │
│  │    ├─► preprocess_request()                                       │   │
│  │    │    └─► 执行 @before_request 回调                             │   │
│  │    │                                                              │   │
│  │    ├─► dispatch_request()                                         │   │
│  │    │    └─► 执行视图函数                                          │   │
│  │    │         │                                                    │   │
│  │    │         ├─► (模板渲染时)                                     │   │
│  │    │         │    ├─► before_render_template.send()             │   │
│  │    │         │    └─► template_rendered.send()                  │   │
│  │    │         │                                                    │   │
│  │    │         └─► (flash 消息时)                                  │   │
│  │    │              └─► message_flashed.send()                    │   │
│  │    │                                                              │   │
│  │    ├─► (如果有异常)                                               │   │
│  │    │    ├─► handle_user_exception()                              │   │
│  │    │    └─► got_request_exception.send()  ◄── 第 3 个信号      │   │
│  │    │                                                              │   │
│  │    └─► finalize_request()                                        │   │
│  │         ├─► process_response()                                    │   │
│  │         │    └─► 执行 @after_request 回调                         │   │
│  │         │                                                         │   │
│  │         └─► request_finished.send()  ◄────────── 第 4 个信号    │   │
│  │                                                                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│       │                                                                   │
│       ▼                                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  ctx.pop(exc)                                                     │   │
│  │    │                                                              │   │
│  │    ├─► (如果是请求上下文)                                         │   │
│  │    │    ├─► do_teardown_request()                                │   │
│  │    │    │    ├─► 执行 @teardown_request 回调                     │   │
│  │    │    │    └─► request_tearing_down.send()  ◄── 第 5 个信号  │   │
│  │    │    │                                                         │   │
│  │    │    └─► request.close()                                       │   │
│  │    │                                                              │   │
│  │    ├─► do_teardown_appcontext()                                  │   │
│  │    │    ├─► 执行 @teardown_appcontext 回调                       │   │
│  │    │    └─► appcontext_tearing_down.send()  ◄── 第 6 个信号     │   │
│  │    │                                                              │   │
│  │    ├─► _cv_app.reset()  ◄── ContextVar 恢复                     │   │
│  │    │                                                              │   │
│  │    └─► appcontext_popped.send()  ◄────────── 第 7 个信号        │   │
│  │                                                                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

**信号触发位置（代码引用）：**

| 信号 | 触发位置 | 文件行号 |
|------|---------|---------|
| `appcontext_pushed` | `AppContext.push()` 后 | `ctx.py:434` |
| `request_started` | `full_dispatch_request()` 开始 | `app.py:1013` |
| `got_request_exception` | 异常处理时 | `app.py:926` |
| `request_finished` | `finalize_request()` 结束 | `app.py:1042` |
| `request_tearing_down` | `do_teardown_request()` 后 | `app.py:1449` |
| `appcontext_tearing_down` | `do_teardown_appcontext()` 后 | `app.py:1477` |
| `appcontext_popped` | `AppContext.pop()` 最后 | `ctx.py:502` |

### 4.8.5 信号与 Teardown 回调的区别

这是一个非常关键的问题，让我们从多个维度对比：

| 对比维度 | 信号 (Signals) | Teardown 回调 |
|---------|---------------|---------------|
| **触发时机** | 生命周期的多个节点 | 只在清理阶段 |
| **返回值处理** | 收集但不使用 | 无返回值 |
| **修改数据** | **禁止**（只通知） | 可以（虽然不推荐） |
| **订阅方式** | 全局注册/临时订阅 | 装饰器注册到应用 |
| **发送者过滤** | 支持（只监听特定 app） | 不支持（绑定到注册的 app） |
| **异常处理** | 3.2 前：抛异常停止；3.2 后：收集所有 | 3.2 后：收集所有 |
| **典型用途** | 监控、审计、日志、测试 | 资源清理（DB 连接等） |

**详细对比 1：能否中断流程？**

```python
# ========== Teardown 回调：不能中断，但可以抛异常 ==========
@app.teardown_request
def my_teardown(exc):
    # Teardown 主要用于清理
    # 抛异常会被收集，但不会影响其他 teardown（Flask 3.2+）
    db.session.remove()

# ========== before_request：可以中断流程 ==========
@app.before_request
def check_auth():
    # 可以返回响应，提前结束请求
    if not g.user:
        return "Unauthorized", 401  # 直接返回，视图不会执行

# ========== 信号：绝对不能中断流程 ==========
@request_started.connect_via(app)
def audit_log(sender, **extra):
    # 信号只能做"只读"操作
    # 即使抛异常，也不会中断请求处理
    # (Flask 3.2+ 会收集异常，在最后汇总抛出)
    logger.info(f"Request started to {sender.name}")
```

**详细对比 2：临时订阅能力**

```python
# ========== 信号可以临时订阅（测试场景非常有用）==========
from flask import template_rendered

def test_rendered_templates(app):
    """测试：验证某个请求渲染了哪些模板"""
    templates = []
    
    def record(sender, template, context, **extra):
        templates.append(template.name)
    
    # 只在这个测试中订阅
    with template_rendered.connected_to(record, app):
        client = app.test_client()
        client.get('/dashboard')
        
        assert 'dashboard.html' in templates
        assert 'sidebar.html' in templates
    
    # 退出 with 块后，自动取消订阅
    # 其他测试不受影响

# ========== Teardown 回调：无法临时订阅 ==========
@app.teardown_request
def cleanup(exc):
    # 一旦注册，全局生效
    # 无法在某个测试中临时"取消注册"
    pass
```

**详细对比 3：参数传递**

```python
# ========== 信号：通过关键字参数传递额外信息 ==========

# 发送时
request_finished.send(self, response=response)
message_flashed.send(self, message=message, category=category)

# 接收时
@request_finished.connect_via(app)
def log_response(sender, response, **extra):
    # 可以访问 response 对象
    print(f"Response status: {response.status}")

# 注意：必须用 **extra 接收未知参数！
# 否则 Flask 新增参数时会报错
@request_finished.connect_via(app)
def bad_handler(sender, response):
    # ❌ 危险！如果 Flask 新增参数，会抛 TypeError
    pass

# ========== Teardown 回调：固定参数 ==========

@app.teardown_request
def cleanup(exc):
    # 只有一个 exc 参数
    # - None: 正常结束
    # - 异常对象: 有未处理的异常
    if exc is None:
        db.session.commit()
    else:
        db.session.rollback()
    db.session.remove()
```

**详细对比 4：执行顺序**

```
请求结束时的完整执行顺序：

1. finalize_request() 中的 process_response()
   └─► @after_request 回调（按注册逆序执行）

2. request_finished.send()
   └─► 所有订阅的信号处理器

3. ctx.pop() 开始
   │
   ├─► do_teardown_request()
   │    ├─► @teardown_request 回调（蓝图级 → 应用级）
   │    └─► request_tearing_down.send()
   │
   ├─► do_teardown_appcontext()
   │    ├─► @teardown_appcontext 回调
   │    └─► appcontext_tearing_down.send()
   │
   ├─► _cv_app.reset()
   │
   └─► appcontext_popped.send()
```

### 4.8.6 异步信号支持 (`_async_wrapper`)

你可能注意到了 Flask 发送信号时的特殊参数：

```python
appcontext_pushed.send(
    self.app, 
    _async_wrapper=self.app.ensure_sync  # ← 这是什么？
)
```

**背景：Blinker 1.8+ 支持异步接收器**

```python
# Blinker 原生支持
from blinker import Signal

sig = Signal()

# 同步接收器
def sync_receiver(sender, **extra):
    print("Sync handler")

# 异步接收器
async def async_receiver(sender, **extra):
    await some_async_operation()

# 注册
sig.connect(sync_receiver)
sig.connect(async_receiver)

# 发送给同步接收器：用 send()
sig.send(sender)

# 发送给异步接收器：用 send_async()
await sig.send_async(sender)
```

**问题：Flask 是 WSGI 应用（同步），如何支持异步接收器？**

**解决方案：`_async_wrapper` 参数**

```python
# Flask 的 ensure_sync 方法
def ensure_sync(self, func):
    """将异步函数包装为同步函数"""
    if iscoroutinefunction(func):
        # 如果是 async def，用 async_to_sync 包装
        return self.async_to_sync(func)
    # 同步函数直接返回
    return func

# 信号发送时
request_started.send(
    self, 
    _async_wrapper=self.ensure_sync
)
```

**Blinker 内部如何处理 `_async_wrapper`：**

```python
# Blinker Signal.send() 伪代码
def send(self, sender, *, _async_wrapper=None, **kwargs):
    results = []
    for receiver in self.receivers_for(sender):
        # 如果有 _async_wrapper，用它包装接收器
        if _async_wrapper is not None:
            receiver = _async_wrapper(receiver)
        
        # 调用接收器
        result = receiver(sender, **kwargs)
        results.append((receiver, result))
    return results
```

**实际使用示例：**

```python
from flask import Flask, request_started

app = Flask(__name__)

# 同步接收器（正常使用）
@request_started.connect_via(app)
def sync_handler(sender, **extra):
    print("Request started (sync)")

# 异步接收器（需要 Flask 2.0+ 的 async 支持）
@request_started.connect_via(app)
async def async_handler(sender, **extra):
    # 这里的 await 会被 ensure_sync 包装
    await some_async_logging("Request started")
```

---

### 4.8.6.1 `ensure_sync` 异步适配器深度解析

**问题背景：Flask 是同步 WSGI 应用，但用户想写异步代码**

```
┌─────────────────────────────────────────────────────────────────┐
│                        冲突场景                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Flask 2.0+ 想支持这样的写法：                                     │
│                                                                   │
│  @app.route('/')                                                 │
│  async def index():        ← 用户想用 async def                 │
│      await async_db.query()                                       │
│      return 'Hello'                                               │
│                                                                   │
│  但 WSGI 服务器是同步的：                                         │
│                                                                   │
│  def wsgi_app(self, environ, start_response):  ← 同步入口       │
│      # 这里不能直接 await！                                       │
│      response = view_function()   ← 同步调用                     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

**解决方案：`ensure_sync` + `async_to_sync` 适配器**

#### `ensure_sync` 的核心实现

```python
# src/flask/app.py:1065-1077

def ensure_sync(self, func: t.Callable[..., t.Any]) -> t.Callable[..., t.Any]:
    """确保函数在同步 WSGI 环境中可以调用
    
    - 普通 def 函数：直接返回
    - async def 协程函数：用 async_to_sync 包装
    """
    if iscoroutinefunction(func):
        # 是异步函数，需要包装
        return self.async_to_sync(func)
    
    # 是同步函数，直接返回
    return func
```

#### `async_to_sync` 的实现（依赖 asgiref）

```python
# src/flask/app.py:1079-1100

def async_to_sync(
    self, func: t.Callable[..., t.Coroutine[t.Any, t.Any, t.Any]]
) -> t.Callable[..., t.Any]:
    """将异步协程函数包装为同步可调用函数
    
    依赖 asgiref 库（需要安装 Flask 的 'async' extra）
    """
    try:
        from asgiref.sync import async_to_sync as asgiref_async_to_sync
    except ImportError:
        raise RuntimeError(
            "Install Flask with the 'async' extra in order to use async views."
        ) from None
    
    # asgiref 的 async_to_sync 会：
    # 1. 创建/获取一个事件循环
    # 2. 在循环中运行协程直到完成
    # 3. 返回结果
    return asgiref_async_to_sync(func)
```

#### `asgiref.sync.async_to_sync` 的工作原理

```
┌─────────────────────────────────────────────────────────────────┐
│              async_to_sync 的执行流程                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  同步调用者                                                        │
│       │                                                           │
│       ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  async_to_sync(async_func)(*args, **kwargs)            │   │
│  │                                                           │   │
│  │  1. 检查当前线程是否有事件循环                            │   │
│  │     - 有：使用现有循环（但要小心重入问题）               │   │
│  │     - 无：创建新的事件循环                                │   │
│  │                                                           │   │
│  │  2. 在事件循环中运行协程                                  │   │
│  │     loop.run_until_complete(async_func(*args, **kwargs))│   │
│  │                                                           │   │
│  │  3. 阻塞等待协程完成                                       │   │
│  │                                                           │   │
│  │  4. 返回结果（或抛出异常）                                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│       │                                                           │
│       ▼                                                           │
│  同步调用者继续执行                                                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

**代码示意：**

```python
# 伪代码：async_to_sync 的核心逻辑
def async_to_sync(coro_func):
    def wrapper(*args, **kwargs):
        import asyncio
        
        # 1. 创建协程对象
        coro = coro_func(*args, **kwargs)
        
        # 2. 获取或创建事件循环
        try:
            loop = asyncio.get_running_loop()
        except RuntimeError:
            loop = asyncio.new_event_loop()
            asyncio.set_event_loop(loop)
        
        # 3. 阻塞运行协程
        try:
            return loop.run_until_complete(coro)
        finally:
            # 清理（如果是新创建的循环）
            pass
    
    return wrapper
```

#### `ensure_sync` 在 Flask 中的所有使用场景

从代码搜索结果看，`ensure_sync` 被广泛用于：

| 场景 | 代码位置 | 说明 |
|------|---------|------|
| **视图函数** | `app.py:990` | `self.ensure_sync(self.view_functions[...])(...)` |
| **before_request** | `app.py:1387` | `self.ensure_sync(before_func)()` |
| **after_request** | `app.py:1408, 1413` | `self.ensure_sync(func)(response)` |
| **teardown_request** | `app.py:1446` | `self.ensure_sync(func)(exc)` |
| **teardown_appcontext** | `app.py:1474` | `self.ensure_sync(func)(exc)` |
| **信号发送** | 多处 | `signal.send(..., _async_wrapper=self.ensure_sync)` |
| **error_handler** | `app.py:863, 895, 946` | 异常处理器 |
| **模板上下文** | `app.py:616` | 模板上下文处理器 |
| **MethodView** | `views.py:110, 116, 191` | 视图类的方法 |
| **跨线程上下文** | `ctx.py:204` | `copy_current_request_context` |

#### 在 teardown 回调中的具体使用

```python
# src/flask/app.py:1440-1451 (do_teardown_request)

def do_teardown_request(self, ctx, exc=None):
    collect_errors = _CollectErrors()
    
    # 遍历所有 teardown_request 回调
    for name in chain(ctx.request.blueprints, (None,)):
        if name in self.teardown_request_funcs:
            # 逆序执行（LIFO）
            for func in reversed(self.teardown_request_funcs[name]):
                with collect_errors:
                    # ========== 关键点 ==========
                    # 用 ensure_sync 包装函数
                    # - 如果是同步 def：直接调用
                    # - 如果是 async def：转成同步再调用
                    self.ensure_sync(func)(exc)
                    # ============================
    
    # 然后发送信号
    with collect_errors:
        request_tearing_down.send(
            self, 
            _async_wrapper=self.ensure_sync,  # 信号也用同一个适配器
            exc=exc
        )
```

**这意味着你可以写异步的 teardown 回调：**

```python
@app.teardown_request
async def async_cleanup(exc):
    """异步清理函数"""
    await async_db.close_connection()

# Flask 内部会自动转换：
# wrapped = ensure_sync(async_cleanup)
# wrapped(exc)  # 同步调用，内部阻塞等待
```

#### 在信号发送中的具体使用

```python
# 发送信号时传入 _async_wrapper
request_started.send(
    self, 
    _async_wrapper=self.ensure_sync  # 关键！
)

# Blinker 内部的处理（伪代码）
def send(self, sender, *, _async_wrapper=None, **kwargs):
    for receiver in self.receivers_for(sender):
        # 如果有包装器，用它包装接收器
        if _async_wrapper is not None:
            receiver = _async_wrapper(receiver)
            # 此时：
            # - 同步接收器：不变
            # - 异步接收器：被 async_to_sync 包装
        
        # 调用接收器（现在都是同步的）
        receiver(sender, **kwargs)
```

**这样用户可以写异步信号处理器：**

```python
@request_started.connect_via(app)
async def async_audit_log(sender, **extra):
    """异步记录日志"""
    await async_logger.info("Request started")

# Blinker 内部：
# receiver = ensure_sync(async_audit_log)  # 转成同步
# receiver(sender, **extra)  # 同步调用
```

#### 完整的执行流程示例

```python
# 用户代码
app = Flask(__name__)

# 异步视图
@app.route('/')
async def async_index():
    await asyncio.sleep(0.1)  # 异步操作
    return 'Hello'

# 异步 teardown
@app.teardown_request
async def async_teardown(exc):
    await async_db.close()

# 异步信号处理器
@request_finished.connect_via(app)
async def async_signal_handler(sender, response, **extra):
    await async_metrics.record(response.status_code)


# ========== 请求处理时的执行流程 ==========

# wsgi_app() 中
ctx.push()

# full_dispatch_request()
request_started.send(self, _async_wrapper=self.ensure_sync)
# 内部：如果有异步接收器，ensure_sync 包装后再调用

# 调用视图
view_func = self.view_functions['index']  # async def async_index
wrapped_view = self.ensure_sync(view_func)  # 转成同步
response = wrapped_view()  # 同步调用，内部阻塞等待协程

# finalize_request()
request_finished.send(self, _async_wrapper=self.ensure_sync, response=response)
# 异步信号处理器被包装后调用

# ctx.pop()
# do_teardown_request()
for func in self.teardown_request_funcs[None]:
    wrapped = self.ensure_sync(func)  # async_teardown 被包装
    wrapped(exc)  # 同步调用
```

#### 性能考虑和限制

```
┌─────────────────────────────────────────────────────────────────┐
│                    async_to_sync 的代价                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  优点：                                                           │
│  ✅ 允许用户在同步 WSGI 环境中写异步代码                         │
│  ✅ 统一的编程模型（同步/异步视图写法类似）                      │
│  ✅ 可以使用 asyncio 生态的库（aiohttp, asyncpg 等）            │
│                                                                   │
│  缺点：                                                           │
│  ❌ 每个 async_to_sync 调用都有开销（事件循环操作）              │
│  ❌ 不能真正并行（仍然是阻塞等待）                                │
│  ❌ 与某些异步框架（如 FastAPI/ASGI）的真正并发不同              │
│  ❌ 重入问题（在异步代码中再调用 async_to_sync 可能死锁）        │
│                                                                   │
│  与纯 ASGI 框架的对比：                                           │
│                                                                   │
│  Flask (WSGI + async_to_sync):                                   │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                  │
│  │  请求 1  │ → │ 同步阻塞 │ → │  等待... │                  │
│  └──────────┘    └──────────┘    └──────────┘                  │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                  │
│  │  请求 2  │ → │  排队...  │ → │  等待... │                  │
│  └──────────┘    └──────────┘    └──────────┘                  │
│                                                                   │
│  FastAPI/Quart (原生 ASGI):                                       │
│  ┌─────────────────────────────────────────────────┐            │
│  │              事件循环                            │            │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐       │            │
│  │  │ 请求 1  │  │ 请求 2  │  │ 请求 3  │       │            │
│  │  │ await中 │  │ await中 │  │ await中 │ ← 并行 │            │
│  │  └─────────┘  └─────────┘  └─────────┘       │            │
│  └─────────────────────────────────────────────────┘            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

#### 最佳实践

```python
# ✅ 推荐：简单的异步操作
@app.route('/data')
async def get_data():
    """使用异步数据库客户端"""
    data = await async_db.query("SELECT * FROM table")
    return jsonify(data)

# ✅ 推荐：异步 teardown 清理
@app.teardown_request
async def cleanup(exc):
    await async_db.close()

# ⚠️ 注意：不能在同步代码中直接 await
@app.route('/')
def sync_view():
    # 这会报错！
    # await some_async_func()
    
    # 必须先包装
    from asgiref.sync import async_to_sync
    result = async_to_sync(some_async_func)()
    return result

# ❌ 不推荐：在异步代码中再嵌套 async_to_sync
@app.route('/')
async def nested_async():
    # 这可能导致死锁或性能问题
    from asgiref.sync import async_to_sync
    
    @async_to_sync
    def inner():
        # ...
    
    inner()  # ❌ 不要这样嵌套
```

### 4.8.7 信号使用最佳实践

**✅ 推荐的使用场景：**

```python
# 1. 监控和审计
from flask import request_started, request_finished

@request_started.connect_via(app)
def log_request_start(sender, **extra):
    metrics.increment('requests.total')
    logger.info(f"Request started at {time.time()}")

# 2. 测试断言
def test_user_login(app, client):
    login_events = []
    
    def record_login(sender, **extra):
        if request.endpoint == 'auth.login':
            login_events.append(True)
    
    with user_logged_in.connected_to(record_login, app):
        client.post('/login', data={'username': 'test'})
        assert len(login_events) == 1

# 3. 解耦的扩展
class AnalyticsExtension:
    def __init__(self, app=None):
        if app is not None:
            self.init_app(app)
    
    def init_app(self, app):
        # 不修改应用代码，只通过信号监听
        from flask import request_finished
        request_finished.connect(self.track_request, sender=app)
    
    def track_request(self, sender, response, **extra):
        # 发送数据到分析服务
        self.send_to_analytics(response.status_code)
```

**❌ 不推荐的使用场景：**

```python
# ❌ 1. 在信号中修改请求数据
@request_started.connect_via(app)
def bad_modify(sender, **extra):
    # 信号的目的是通知，不是修改
    # 这会让代码流程变得难以理解
    from flask import request
    request.args = {'modified': 'yes'}  # ❌ 不要这样做

# ❌ 2. 在信号中处理业务逻辑
@got_request_exception.connect_via(app)
def bad_handle_error(sender, exception, **extra):
    # 异常处理应该用 errorhandler
    # 信号只用于记录/通知
    # return error_page()  # ❌ 信号返回值被忽略

# ✅ 正确方式：用 errorhandler
@app.errorhandler(500)
def handle_500(e):
    return render_template('500.html'), 500

# ❌ 3. 依赖信号的执行顺序
@request_started.connect_via(app)
def first(sender, **extra):
    print("First")

@request_started.connect_via(app)
def second(sender, **extra):
    print("Second")

# 信号执行顺序是不确定的！
# 不要假设 first 一定在 second 之前执行
# 如果需要顺序，用 before_request 装饰器
```

**⚠️ 重要注意事项：**

```python
# 1. 始终使用 **extra 接收参数
@template_rendered.connect_via(app)
def good_handler(sender, template, context, **extra):
    # ✅ 安全：Flask 新增参数不会报错
    pass

@template_rendered.connect_via(app)
def bad_handler(sender, template, context):
    # ❌ 危险：如果 Flask 新增参数，会抛 TypeError
    pass

# 2. 发送信号时，sender 必须是真实对象，不是代理
from flask import current_app

# ❌ 错误：current_app 是 LocalProxy
request_started.send(current_app)

# ✅ 正确：获取真实对象
request_started.send(current_app._get_current_object())

# 3. 信号是可选依赖
try:
    from blinker import signal
    has_blinker = True
except ImportError:
    has_blinker = False

# Flask 的做法：如果没有安装 blinker，信号仍然可以"发送"，
# 但没有任何效果（send() 是一个空操作）
```

### 4.8.8 完整信号列表

| 信号名称 | 触发时机 | 传递参数 | 典型用途 |
|---------|---------|---------|---------|
| `request_started` | 请求开始，before_request 之前 | `sender` (app) | 监控、统计 |
| `request_finished` | 请求结束，after_request 之后 | `sender` (app), `response` | 日志、审计 |
| `request_tearing_down` | teardown_request 回调之后 | `sender` (app), `exc` | 清理通知 |
| `got_request_exception` | 视图抛出异常时 | `sender` (app), `exception` | 错误报告 |
| `appcontext_pushed` | 上下文推入后 | `sender` (app) | 初始化 |
| `appcontext_tearing_down` | teardown_appcontext 之后 | `sender` (app), `exc` | 清理通知 |
| `appcontext_popped` | 上下文弹出后 | `sender` (app) | 清理 |
| `before_render_template` | 模板渲染前 | `sender` (app), `template`, `context` | 调试 |
| `template_rendered` | 模板渲染后 | `sender` (app), `template`, `context` | 测试、调试 |
| `message_flashed` | 调用 `flash()` 时 | `sender` (app), `message`, `category` | 前端集成 |

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
