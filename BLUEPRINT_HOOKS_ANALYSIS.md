# Flask Blueprint 钩子延迟注册与合并机制深度分析

## 目录

1. [概述](#1-概述)
2. [三种注册/合并机制总览](#2-三种注册合并机制总览)
3. [机制 A：直接存储 + _merge_blueprint_funcs 合并](#3-机制-a直接存储--_merge_blueprint_funcs-合并)
4. [机制 B：record_once + deferred_functions](#4-机制-brecord_once--deferred_functions)
5. [机制 C：路由的 record + deferred_functions](#5-机制-c路由的-record--deferred_functions)
6. [请求时钩子的调用机制（作用域原理）](#6-请求时钩子的调用机制作用域原理)
7. [三种机制对比总结](#7-三种机制对比总结)
8. [完整链路流程图](#8-完整链路流程图)
9. [附录：关键源码位置速查](#9-附录关键源码位置速查)

---

## 1. 概述

在上一篇报告中，我们分析了 Blueprint 路由通过 `deferred_functions` 延迟注册的机制。但 Blueprint 上的**钩子函数**（如 `before_request`、`after_request`、`errorhandler` 等）实际上有**两种不同的注册路径**：

1. **Blueprint 级别钩子**（如 `@bp.before_request`）：只对该 Blueprint 的路由生效
2. **全局钩子**（如 `@bp.before_app_request`）：对所有请求生效

这两种钩子的注册和合并机制**完全不同**：
- Blueprint 级别钩子：**不使用 `deferred_functions`**，直接存储在 Blueprint 的字典中，通过 `_merge_blueprint_funcs` 合并
- 全局钩子：**使用 `record_once` + `deferred_functions`**，注册时直接操作 App 的字典
- 路由：**使用 `record` + `deferred_functions`**，通过 `BlueprintSetupState.add_url_rule` 中转

本文将深入分析这三种机制的实现细节、差异以及作用域原理。

---

## 2. 三种注册/合并机制总览

| 机制 | 用途 | 存储位置 | 合并方式 | 重复注册行为 |
|------|------|----------|----------|-------------|
| **A: 直接存储 + merge** | Blueprint 级别钩子 | Blueprint 实例字典 | `_merge_blueprint_funcs` | 首次注册合并 |
| **B: record_once + deferred** | 全局钩子 | `deferred_functions` | 直接操作 App 字典 | 只执行一次 |
| **C: record + deferred** | 路由 | `deferred_functions` | `state.add_url_rule` 中转 | 每次都执行 |

### 2.1 涉及的钩子类型

| 钩子类型 | Blueprint 级别（机制 A） | 全局（机制 B） |
|----------|-------------------------|-----------------|
| 请求前 | `@bp.before_request` | `@bp.before_app_request` |
| 请求后 | `@bp.after_request` | `@bp.after_app_request` |
| 请求清理 | `@bp.teardown_request` | `@bp.teardown_app_request` |
| 模板上下文 | `@bp.context_processor` | `@bp.app_context_processor` |
| URL 值预处理 | `@bp.url_value_preprocessor` | `@bp.app_url_value_preprocessor` |
| URL 默认值 | `@bp.url_defaults` | `@bp.app_url_defaults` |
| 错误处理 | `@bp.errorhandler` | `@bp.app_errorhandler` |

### 2.2 数据结构总览

```python
class Blueprint(Scaffold):
    def __init__(self, ...):
        # ========== 机制 A 的存储：直接存储 ==========
        self.view_functions: dict[str, RouteCallable] = {}
        self.error_handler_spec: dict[
            AppOrBlueprintKey,  # None 或 blueprint 名称
            dict[int | None, dict[type[Exception], ErrorHandlerCallable]],
        ] = defaultdict(lambda: defaultdict(dict))
        self.before_request_funcs: dict[
            AppOrBlueprintKey, list[BeforeRequestCallable]
        ] = defaultdict(list)
        self.after_request_funcs: dict[
            AppOrBlueprintKey, list[AfterRequestCallable]
        ] = defaultdict(list)
        self.teardown_request_funcs: dict[
            AppOrBlueprintKey, list[TeardownCallable]
        ] = defaultdict(list)
        self.template_context_processors: dict[
            AppOrBlueprintKey, list[TemplateContextProcessorCallable]
        ] = defaultdict(list, {None: [_default_template_ctx_processor]})
        self.url_value_preprocessors: dict[
            AppOrBlueprintKey, list[URLValuePreprocessorCallable],
        ] = defaultdict(list)
        self.url_default_functions: dict[
            AppOrBlueprintKey, list[URLDefaultCallable]
        ] = defaultdict(list)
        
        # ========== 机制 B 和 C 的存储：延迟函数列表 ==========
        self.deferred_functions: list[DeferredSetupFunction] = []
        # DeferredSetupFunction = Callable[[BlueprintSetupState], None]
```

**关键点**：
- 机制 A 的数据直接存储在 Blueprint 实例的各种字典中，key 为 `None`
- 机制 B 和 C 的数据存储在 `deferred_functions` 列表中，是可调用对象

---

## 3. 机制 A：直接存储 + `_merge_blueprint_funcs` 合并

### 3.1 装饰器实现（Scaffold 中定义）

Blueprint 级别的钩子装饰器在 `Scaffold` 基类中定义，**Blueprint 不重写**这些方法。

**源码位置**: `src/flask/sansio/scaffold.py:459-698`

#### before_request

```python
@setupmethod
def before_request(self, f: T_before_request) -> T_before_request:
    """Register a function to run before each request.
    
    When used on a blueprint, this executes before
    every request that the blueprint handles.
    """
    # 直接存储到 Blueprint 的字典，key 为 None
    self.before_request_funcs.setdefault(None, []).append(f)
    return f
```

#### after_request

```python
@setupmethod
def after_request(self, f: T_after_request) -> T_after_request:
    """Register a function to run after each request to this object.
    
    When used on a blueprint, this executes after
    every request that the blueprint handles.
    """
    self.after_request_funcs.setdefault(None, []).append(f)
    return f
```

#### errorhandler

```python
@setupmethod
def errorhandler(
    self, code_or_exception: type[Exception] | int
) -> t.Callable[[T_error_handler], T_error_handler]:
    """Register a function to handle errors by code or exception class.
    
    When used on a blueprint, this can handle
    errors from requests that the blueprint handles.
    """

    def decorator(f: T_error_handler) -> T_error_handler:
        self.register_error_handler(code_or_exception, f)
        return f

    return decorator

@setupmethod
def register_error_handler(
    self,
    code_or_exception: type[Exception] | int,
    f: ft.ErrorHandlerCallable,
) -> None:
    exc_class, code = self._get_exc_class_and_code(code_or_exception)
    # 存储到 error_handler_spec[None][code][exc_class]
    self.error_handler_spec[None][code][exc_class] = f
```

### 3.2 关键特点

1. **不使用 `deferred_functions`**：直接存储到 Blueprint 实例的字典
2. **key 为 `None`**：表示"当前对象的默认作用域"
3. **延迟合并**：存储在 Blueprint 中，直到 `register_blueprint` 时才合并到 App

### 3.3 `_merge_blueprint_funcs` 合并机制

**源码位置**: `src/flask/sansio/blueprints.py:379-410`

```python
def _merge_blueprint_funcs(self, app: App, name: str) -> None:
    """Merge blueprint functions into the app.
    
    :param name: The blueprint name (used as key prefix)
    """
    
    def extend(
        bp_dict: dict[ft.AppOrBlueprintKey, list[t.Any]],
        parent_dict: dict[ft.AppOrBlueprintKey, list[t.Any]],
    ) -> None:
        """辅助函数：合并字典
        
        关键逻辑：将 key 从 None 转换为 blueprint 名称
        """
        for key, values in bp_dict.items():
            # ========== 核心：作用域转换 ==========
            # 如果 key 是 None（Blueprint 内部的默认作用域），
            # 则转换为 blueprint 名称作为 App 中的 key
            # 这样可以实现：只有匹配该 blueprint 的请求才执行这些钩子
            key = name if key is None else f"{name}.{key}"
            # ====================================
            
            parent_dict[key].extend(values)

    # ========== 1. 合并错误处理器（特殊处理） ==========
    for key, value in self.error_handler_spec.items():
        # key 转换：None → name
        key = name if key is None else f"{name}.{key}"
        value = defaultdict(
            dict,
            {
                code: {exc_class: func for exc_class, func in code_values.items()}
                for code, code_values in value.items()
            },
        )
        app.error_handler_spec[key] = value

    # ========== 2. 合并视图函数 ==========
    # 注意：view_functions 的 key 已经是带前缀的 endpoint
    for endpoint, func in self.view_functions.items():
        app.view_functions[endpoint] = func

    # ========== 3. 合并各种钩子函数 ==========
    extend(self.before_request_funcs, app.before_request_funcs)
    extend(self.after_request_funcs, app.after_request_funcs)
    extend(self.teardown_request_funcs, app.teardown_request_funcs)
    extend(self.url_default_functions, app.url_default_functions)
    extend(self.url_value_preprocessors, app.url_value_preprocessors)
    extend(self.template_context_processors, app.template_context_processors)
```

### 3.4 合并过程示例

假设有一个名为 `"auth"` 的 Blueprint：

```python
auth_bp = Blueprint("auth", __name__)

@auth_bp.before_request
def before_auth():
    print("before auth request")

@auth_bp.after_request
def after_auth(response):
    print("after auth request")
    return response
```

**合并前（Blueprint 内部）**:
```python
auth_bp.before_request_funcs = {
    None: [before_auth]  # key 是 None
}
auth_bp.after_request_funcs = {
    None: [after_auth]  # key 是 None
}
```

**合并后（App 中）**:
```python
app.before_request_funcs = {
    None: [...],           # 全局钩子
    "auth": [before_auth]  # key 变为 "auth"
}
app.after_request_funcs = {
    None: [...],           # 全局钩子
    "auth": [after_auth]   # key 变为 "auth"
}
```

### 3.5 合并时机

**源码位置**: `src/flask/sansio/blueprints.py:331-332`

```python
def register(self, app: App, options: dict[str, t.Any]) -> None:
    # ...
    
    # 只有首次注册或新名称注册时才合并
    if first_bp_registration or first_name_registration:
        self._merge_blueprint_funcs(app, name)
    
    # ...
```

**条件说明**：
- `first_bp_registration`: 该 Blueprint 实例是否首次注册到 App
- `first_name_registration`: 该名称是否首次注册

这意味着：
- 同一个 Blueprint 实例注册多次（不同 `url_prefix`），**钩子只合并一次**
- 不同 Blueprint 实例（即使相同名称），**各自合并**

---

## 4. 机制 B：`record_once` + `deferred_functions`

### 4.1 装饰器实现（Blueprint 中重写）

全局钩子装饰器在 `Blueprint` 类中定义，使用 `record_once` 将回调存储到 `deferred_functions`。

**源码位置**: `src/flask/sansio/blueprints.py:613-691`

#### before_app_request

```python
@setupmethod
def before_app_request(self, f: T_before_request) -> T_before_request:
    """Like :meth:`before_request`, but before every request, 
    not only those handled by the blueprint.
    
    Equivalent to :meth:`.Flask.before_request`.
    """
    # 使用 record_once 存储到 deferred_functions
    self.record_once(
        # 回调直接操作 App 的字典，key 为 None（全局）
        lambda s: s.app.before_request_funcs.setdefault(None, []).append(f)
    )
    return f
```

#### after_app_request

```python
@setupmethod
def after_app_request(self, f: T_after_request) -> T_after_request:
    """Like :meth:`after_request`, but after every request, 
    not only those handled by the blueprint.
    """
    self.record_once(
        lambda s: s.app.after_request_funcs.setdefault(None, []).append(f)
    )
    return f
```

#### app_errorhandler

```python
@setupmethod
def app_errorhandler(
    self, code: type[Exception] | int
) -> t.Callable[[T_error_handler], T_error_handler]:
    """Like :meth:`errorhandler`, but for every request, 
    not only those handled by the blueprint.
    """

    def decorator(f: T_error_handler) -> T_error_handler:
        def from_blueprint(state: BlueprintSetupState) -> None:
            # 直接调用 App 的 errorhandler 装饰器
            state.app.errorhandler(code)(f)

        self.record_once(from_blueprint)
        return f

    return decorator
```

### 4.2 `record_once` 实现

**源码位置**: `src/flask/sansio/blueprints.py:232-244`

```python
@setupmethod
def record_once(self, func: DeferredSetupFunction) -> None:
    """Works like :meth:`record` but wraps the function in another
    function that will ensure the function is only called once. If the
    blueprint is registered a second time on the application, the
    function passed is not called.
    """

    def wrapper(state: BlueprintSetupState) -> None:
        # 只有首次注册时才执行
        if state.first_registration:
            func(state)

    self.record(update_wrapper(wrapper, func))
```

### 4.3 执行时机

在 `Blueprint.register` 中：

**源码位置**: `src/flask/sansio/blueprints.py:334-335`

```python
def register(self, app: App, options: dict[str, t.Any]) -> None:
    # ...
    
    # 遍历所有延迟函数并执行
    for deferred in self.deferred_functions:
        deferred(state)  # 调用 wrapper(state)
    
    # ...
```

### 4.4 执行过程示例

假设有一个名为 `"auth"` 的 Blueprint：

```python
auth_bp = Blueprint("auth", __name__)

@auth_bp.before_app_request
def before_all():
    print("before every request")
```

**定义阶段（存储到 deferred_functions）**:
```python
auth_bp.deferred_functions = [
    # wrapper 函数，检查 state.first_registration
    lambda state: if state.first_registration: 
                       s.app.before_request_funcs[None].append(before_all)
]
```

**注册阶段（执行）**:
```python
# 首次注册时 state.first_registration = True
# 执行：app.before_request_funcs[None].append(before_all)

# 结果：
app.before_request_funcs = {
    None: [..., before_all]  # 添加到全局作用域
}
```

**再次注册时**：
- `state.first_registration = False`
- `wrapper` 函数内部判断后**不执行**
- 钩子不会重复添加

---

## 5. 机制 C：路由的 `record` + `deferred_functions`

### 5.1 与机制 B 的区别

| 特性 | 全局钩子（机制 B） | 路由（机制 C） |
|------|-------------------|----------------|
| 注册函数 | `record_once()` | `record()` |
| 执行条件 | `state.first_registration` | 无条件执行 |
| 重复注册 | 只执行一次 | 每次都执行 |
| 执行方式 | 直接操作 `app.xxx[None]` | 通过 `state.add_url_rule` 中转 |

### 5.2 Blueprint.add_url_rule 实现

**源码位置**: `src/flask/sansio/blueprints.py:412-441`

```python
@setupmethod
def add_url_rule(
    self,
    rule: str,
    endpoint: str | None = None,
    view_func: ft.RouteCallable | None = None,
    provide_automatic_options: bool | None = None,
    **options: t.Any,
) -> None:
    """Register a URL rule with the blueprint."""
    if endpoint and "." in endpoint:
        raise ValueError("'endpoint' may not contain a dot '.' character.")

    if view_func and hasattr(view_func, "__name__") and "." in view_func.__name__:
        raise ValueError("'view_func' name may not contain a dot '.' character.")

    # 使用 record（不是 record_once），每次注册都执行
    self.record(
        lambda s: s.add_url_rule(
            rule,
            endpoint,
            view_func,
            provide_automatic_options=provide_automatic_options,
            **options,
        )
    )
```

### 5.3 BlueprintSetupState.add_url_rule

**源码位置**: `src/flask/sansio/blueprints.py:87-116`

```python
def add_url_rule(
    self,
    rule: str,
    endpoint: str | None = None,
    view_func: ft.RouteCallable | None = None,
    **options: t.Any,
) -> None:
    """A helper method to register a rule to the application.
    The endpoint is automatically prefixed with the blueprint's name.
    """
    # ========== 1. 拼接 URL 前缀 ==========
    if self.url_prefix is not None:
        if rule:
            rule = "/".join((self.url_prefix.rstrip("/"), rule.lstrip("/")))
        else:
            rule = self.url_prefix
    
    # ========== 2. 设置子域名 ==========
    options.setdefault("subdomain", self.subdomain)
    
    # ========== 3. 确定端点名称 ==========
    if endpoint is None:
        endpoint = _endpoint_from_view_func(view_func)  # 默认为函数名
    
    # ========== 4. 处理 URL 默认值 ==========
    defaults = self.url_defaults
    if "defaults" in options:
        defaults = dict(defaults, **options.pop("defaults"))

    # ========== 5. 调用 App.add_url_rule ==========
    self.app.add_url_rule(
        rule,
        # 端点格式：name_prefix.name.endpoint（去掉前导点）
        f"{self.name_prefix}.{self.name}.{endpoint}".lstrip("."),
        view_func,
        defaults=defaults,
        **options,
    )
```

### 5.4 路由注册示例

```python
auth_bp = Blueprint("auth", __name__, url_prefix="/auth")

@auth_bp.route("/login")
def login():
    return "login page"
```

**定义阶段**:
```python
auth_bp.deferred_functions = [
    lambda s: s.add_url_rule("/login", None, login, ...)
]
```

**注册阶段** (`app.register_blueprint(auth_bp, url_prefix="/api")`):
```python
# 执行 deferred(state)
# state.url_prefix = "/api" (注册时指定) + "/auth" (Blueprint 默认) = "/api/auth"

# state.add_url_rule("/login", None, login, ...)
#   → 拼接 URL: "/api/auth" + "/" + "/login" = "/api/auth/login"
#   → 拼接端点: "" + "." + "auth" + "." + "login" = "auth.login"
#   → 调用 app.add_url_rule("/api/auth/login", "auth.login", login, ...)

# 结果：
# app.url_map 中添加 Rule("/api/auth/login", endpoint="auth.login")
# app.view_functions["auth.login"] = login
```

---

## 6. 请求时钩子的调用机制（作用域原理）

### 6.1 核心问题

合并后，App 的字典中存储了各种 key 的钩子：

```python
app.before_request_funcs = {
    None: [global_hook1, global_hook2],      # 全局钩子
    "auth": [auth_hook1, auth_hook2],          # auth blueprint 的钩子
    "api": [api_hook1],                         # api blueprint 的钩子
    "admin.users": [nested_hook],               # 嵌套 blueprint 的钩子
}
```

**问题**：请求时如何知道哪些钩子应该执行？

### 6.2 preprocess_request 实现

**源码位置**: `src/flask/app.py:1366-1392`

```python
def preprocess_request(self, ctx: AppContext) -> ft.ResponseReturnValue | None:
    """Called before the request is dispatched.
    
    Calls url_value_preprocessors and before_request_funcs
    registered with the app and the current blueprint.
    """
    req = ctx.request
    
    # ========== 核心：确定作用域列表 ==========
    # names = (None, *reversed(req.blueprints))
    # 例如：
    # - 全局请求 (不匹配任何 blueprint): names = (None,)
    # - 匹配 auth blueprint: names = (None, "auth")
    # - 匹配嵌套的 admin.users: names = (None, "users", "admin")
    names = (None, *reversed(req.blueprints))

    # ========== 1. 先执行 url_value_preprocessors ==========
    for name in names:
        if name in self.url_value_preprocessors:
            for url_func in self.url_value_preprocessors[name]:
                url_func(req.endpoint, req.view_args)

    # ========== 2. 再执行 before_request ==========
    for name in names:
        if name in self.before_request_funcs:
            for before_func in self.before_request_funcs[name]:
                rv = self.ensure_sync(before_func)()

                if rv is not None:
                    return rv  # 提前返回，中断请求处理

    return None
```

### 6.3 process_response 实现

**源码位置**: `src/flask/app.py:1394-1418`

```python
def process_response(self, ctx: AppContext, response: Response) -> Response:
    """Called after the request is dispatched to modify the response."""
    
    # 先执行请求特定的 after_request
    for func in ctx._after_request_functions:
        response = self.ensure_sync(func)(response)

    # ========== 核心：作用域列表（顺序不同）==========
    # 注意：这里是 chain(ctx.request.blueprints, (None,))
    # 顺序：先 blueprint 级，后全局
    # 同时使用 reversed 遍历（因为 after_request 是反序执行的设计）
    for name in chain(ctx.request.blueprints, (None,)):
        if name in self.after_request_funcs:
            # after_request 是反序执行的（后注册的先执行）
            for func in reversed(self.after_request_funcs[name]):
                response = self.ensure_sync(func)(response)

    # ... 保存 session
    return response
```

### 6.4 `req.blueprints` 的来源

`request.blueprints` 是从哪里来的？

**答案**：从匹配的 `Rule.endpoint` 解析而来。

端点格式示例：
- `"auth.login"` → 解析为 `["auth"]`
- `"admin.users.profile"` → 解析为 `["admin", "users"]`
- `"index"`（直接注册到 app）→ 解析为 `[]`

**执行顺序**：
- `names = (None, *reversed(req.blueprints))`
- 对于 `endpoint = "admin.users.profile"`：
  - `req.blueprints = ["admin", "users"]`
  - `reversed(req.blueprints) = ["users", "admin"]`
  - `names = (None, "users", "admin")`
  - 执行顺序：**全局 → users → admin**

### 6.5 作用域机制图解

```
请求: GET /api/auth/login
匹配的路由端点: "api.auth.login"

                      ┌─────────────────────────────────────────┐
                      │     req.blueprints = ["api", "auth"]   │
                      └─────────────────────────────────────────┘
                                          │
                                          ▼
                    ┌─────────────────────────────────────────┐
                    │  names = (None, *reversed(["api", "auth"])) │
                    │        = (None, "auth", "api")           │
                    └─────────────────────────────────────────┘
                                          │
            ┌─────────────────────────────┼─────────────────────────────┐
            ▼                             ▼                             ▼
    ┌───────────────┐           ┌───────────────┐           ┌───────────────┐
    │  name = None  │           │ name = "auth" │           │ name = "api"  │
    │  (全局作用域)  │           │(auth  blueprint)│           │(api  blueprint) │
    └───────────────┘           └───────────────┘           └───────────────┘
            │                             │                             │
            ▼                             ▼                             ▼
    执行全局钩子                   执行 auth 钩子                   执行 api 钩子
    before_request_funcs[None]    before_request_funcs["auth"]    before_request_funcs["api"]
```

### 6.6 errorhandler 的作用域

错误处理器的查找逻辑稍微不同：

**源码位置**: `src/flask/sansio/app.py:865-888`

```python
def _find_error_handler(
    self, e: Exception, blueprints: list[str]
) -> ft.ErrorHandlerCallable | None:
    """Return a registered error handler for an exception.
    
    Order: blueprint handler for code → app handler for code
         → blueprint handler for class → app handler for class
    """
    exc_class, code = self._get_exc_class_and_code(type(e))
    
    # 名称列表：先 blueprint，后全局 (None)
    names = (*blueprints, None)

    # 先按状态码查找
    for c in (code, None) if code is not None else (None,):
        for name in names:
            handler_map = self.error_handler_spec[name][c]

            if not handler_map:
                continue

            # 按异常类的 MRO 查找
            for cls in exc_class.__mro__:
                handler = handler_map.get(cls)

                if handler is not None:
                    return handler
    return None
```

**查找顺序**：
1. 按状态码查找：先 blueprint 级，后全局
2. 按异常类查找：先 blueprint 级，后全局
3. 同一级别内，按 MRO 顺序（子类优先）

---

## 7. 三种机制对比总结

### 7.1 综合对比表

| 维度 | 机制 A: Blueprint 级别钩子 | 机制 B: 全局钩子 | 机制 C: 路由 |
|------|---------------------------|-----------------|--------------|
| **装饰器示例** | `@bp.before_request` | `@bp.before_app_request` | `@bp.route` |
| **存储位置** | Blueprint 实例字典 | `deferred_functions` | `deferred_functions` |
| **注册函数** | 直接存储 | `record_once()` | `record()` |
| **合并函数** | `_merge_blueprint_funcs` | 遍历 `deferred_functions` | 遍历 `deferred_functions` |
| **key 转换** | `None` → `"blueprint_name"` | 保持 `None` | 端点加前缀 |
| **作用域 key** | `"blueprint_name"` | `None` (全局) | 端点格式 |
| **重复注册** | 首次合并 | 只执行一次 | 每次都执行 |
| **请求时查找** | 按 `req.blueprints` 匹配 | 所有请求都执行 | 按端点匹配 |

### 7.2 数据流向对比

#### 机制 A: Blueprint 级别钩子

```
定义阶段                          注册阶段                          请求阶段
─────────                          ─────────                          ────────

@bp.before_request
    │
    ▼
bp.before_request_funcs[None]
    │
    ▼ (register_blueprint)
_merge_blueprint_funcs()
    │
    ▼ (key 转换)
app.before_request_funcs["bp_name"]
    │
    ▼ (请求时)
preprocess_request()
    │
    ▼
检查 req.blueprints
    │
    ├───► 包含 "bp_name"? ──► 执行钩子
    └───► 不包含? ──► 跳过
```

#### 机制 B: 全局钩子

```
定义阶段                          注册阶段                          请求阶段
─────────                          ─────────                          ────────

@bp.before_app_request
    │
    ▼
record_once(lambda s: ...)
    │
    ▼
bp.deferred_functions.append(wrapper)
    │
    ▼ (register_blueprint)
遍历 deferred_functions
    │
    ▼ (检查 first_registration)
首次注册? ──► 是 ──► app.before_request_funcs[None].append(f)
              │
              └──► 否 ──► 跳过
    │
    ▼ (请求时)
preprocess_request()
    │
    ▼
names = (None, ...)  # 总是包含 None
    │
    ▼
执行 app.before_request_funcs[None] 中的所有钩子
```

#### 机制 C: 路由

```
定义阶段                          注册阶段                          请求阶段
─────────                          ─────────                          ────────

@bp.route("/users")
    │
    ▼
bp.add_url_rule()
    │
    ▼
record(lambda s: s.add_url_rule(...))
    │
    ▼
bp.deferred_functions.append(lambda s: ...)
    │
    ▼ (register_blueprint)
遍历 deferred_functions
    │
    ▼ (每次都执行)
state.add_url_rule(rule, endpoint, view_func, ...)
    │
    ├───► 拼接 URL 前缀: /prefix + /users
    ├───► 拼接端点前缀: bp_name.endpoint
    └───► 调用 app.add_url_rule()
    │
    ▼
app.url_map.add(Rule(...))
app.view_functions["bp_name.endpoint"] = view_func
    │
    ▼ (请求时)
url_adapter.match()
    │
    ▼
匹配 Rule，获取 endpoint = "bp_name.endpoint"
    │
    ▼
dispatch_request()
    │
    ▼
view_functions["bp_name.endpoint"](**view_args)
```

---

## 8. 完整链路流程图

### 8.1 注册阶段总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Blueprint 注册阶段总览                                  │
└─────────────────────────────────────────────────────────────────────────────┘

app.register_blueprint(bp, url_prefix="/api")
    │
    ▼
bp.register(app, options)
    │
    ├───► 步骤 1：创建 BlueprintSetupState
    │         state.app = app
    │         state.url_prefix = "/api"
    │         state.name = bp.name
    │         state.first_registration = True/False
    │
    ├───► 步骤 2：注册静态文件路由
    │         state.add_url_rule(...)
    │
    ├───► 步骤 3：合并机制 A 的钩子（_merge_blueprint_funcs）
    │         │
    │         ├───► 合并 error_handler_spec
    │         │         key: None → "bp_name"
    │         │
    │         ├───► 合并 view_functions
    │         │
    │         └───► 合并其他钩子（通过 extend 函数）
    │               before_request_funcs[None] → app.before_request_funcs["bp_name"]
    │               after_request_funcs[None] → app.after_request_funcs["bp_name"]
    │               ...
    │
    ├───► 步骤 4：执行 deferred_functions（机制 B 和 C）
    │         │
    │         ├───► 遍历 bp.deferred_functions
    │         │
    │         ├───► 机制 B（全局钩子）:
    │         │     wrapper(state)
    │         │         │
    │         │         └───► 检查 state.first_registration
    │         │               ├───► True: 执行 lambda s: s.app.xxx[None].append(f)
    │         │               └───► False: 跳过
    │         │
    │         └───► 机制 C（路由）:
    │               lambda s: s.add_url_rule(...)
    │                   │
    │                   └───► state.add_url_rule(rule, endpoint, view_func, ...)
    │                         │
    │                         ├───► 拼接 URL 前缀
    │                         ├───► 拼接端点前缀
    │                         └───► app.add_url_rule(...)
    │
    ├───► 步骤 5：处理 CLI 命令
    │
    └───► 步骤 6：递归注册嵌套 Blueprint
              传递 name_prefix、url_prefix、subdomain
```

### 8.2 请求时钩子调用流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        请求时钩子调用流程                                      │
└─────────────────────────────────────────────────────────────────────────────┘

HTTP 请求到达
    │
    ▼
ctx.push() → match_request()
    │
    ▼
匹配 Rule，获取 endpoint（如 "api.auth.login"）
    │
    ▼
解析 blueprints: ["api", "auth"]
    │
    ▼
full_dispatch_request()
    │
    ├───► preprocess_request()
    │         │
    │         ├───► names = (None, *reversed(["api", "auth"]))
    │         │         = (None, "auth", "api")
    │         │
    │         ├───► 执行 url_value_preprocessors
    │         │     for name in names:
    │         │         if name in url_value_preprocessors:
    │         │             for func in url_value_preprocessors[name]:
    │         │                 func(...)
    │         │
    │         └───► 执行 before_request
    │               for name in names:
    │                   if name in before_request_funcs:
    │                       for func in before_request_funcs[name]:
    │                           rv = func()
    │                           if rv is not None: return rv
    │
    ├───► dispatch_request()
    │         │
    │         └───► view_functions[endpoint](**view_args)
    │
    └───► finalize_request()
              │
              └───► process_response()
                    │
                    ├───► 执行 ctx._after_request_functions
                    │
                    └───► 执行 after_request
                          for name in chain(["api", "auth"], (None,)):
                              if name in after_request_funcs:
                                  for func in reversed(after_request_funcs[name]):
                                      response = func(response)
                    │
                    └───► 保存 session
    │
    ▼
ctx.pop()
    │
    └───► do_teardown_request()
          for name in chain(["api", "auth"], (None,)):
              if name in teardown_request_funcs:
                  for func in reversed(teardown_request_funcs[name]):
                      func(exc)
```

---

## 9. 附录：关键源码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| `Scaffold.before_request` | `sansio/scaffold.py` | 459-484 |
| `Scaffold.after_request` | `sansio/scaffold.py` | 486-505 |
| `Scaffold.errorhandler` | `sansio/scaffold.py` | 597-639 |
| `Blueprint.record` | `sansio/blueprints.py` | 223-230 |
| `Blueprint.record_once` | `sansio/blueprints.py` | 232-244 |
| `Blueprint.register` | `sansio/blueprints.py` | 273-377 |
| `Blueprint._merge_blueprint_funcs` | `sansio/blueprints.py` | 379-410 |
| `Blueprint.add_url_rule` | `sansio/blueprints.py` | 412-441 |
| `Blueprint.before_app_request` | `sansio/blueprints.py` | 613-621 |
| `Blueprint.after_app_request` | `sansio/blueprints.py` | 623-631 |
| `Blueprint.app_errorhandler` | `sansio/blueprints.py` | 655-670 |
| `BlueprintSetupState.add_url_rule` | `sansio/blueprints.py` | 87-116 |
| `Flask.preprocess_request` | `app.py` | 1366-1392 |
| `Flask.process_response` | `app.py` | 1394-1418 |
| `Flask.do_teardown_request` | `app.py` | 1420-1451 |
| `App._find_error_handler` | `sansio/app.py` | 865-888 |

---

## 核心设计思想总结

### 1. 延迟注册的统一与分化

Flask 使用 `deferred_functions` 作为统一的延迟注册机制，但根据需求分化出两种变体：

- **`record()`**：每次注册都执行，用于路由（同一 Blueprint 可注册到多个 URL 前缀）
- **`record_once()` + `first_registration`**：只执行一次，用于全局钩子（避免重复注册）

### 2. 作用域控制的巧妙设计

通过**key 转换**实现作用域控制：

- Blueprint 级别钩子：合并时 `key = None` → `key = "blueprint_name"`
- 全局钩子：保持 `key = None`
- 请求时：通过 `req.blueprints` 动态确定执行哪些钩子

### 3. 两种数据存储策略

| 策略 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| **直接存储 + 合并** | Blueprint 级别钩子 | 合并时可灵活转换 key | 需要专门的合并逻辑 |
| **deferred_functions** | 全局钩子、路由 | 统一的执行机制，灵活 | lambda 闭包可能增加理解难度 |

### 4. 设计哲学

1. **模块化优先**：Blueprint 级别钩子是默认行为，全局钩子需要显式使用 `_app_` 后缀
2. **灵活性**：同一 Blueprint 可注册多次，路由每次都注册，钩子只注册一次
3. **一致性**：全局钩子的行为与直接在 App 上注册的钩子完全一致
4. **可预测性**：通过 `first_registration` 标志确保全局钩子不会重复执行
