---
name: fastapi-lifespan
description: >
  Use this skill when building FastAPI applications that need startup/shutdown lifecycle
  management — loading ML models, initializing database connection pools, setting up shared
  resources before requests begin, or cleaning up resources on shutdown. Covers the lifespan
  parameter (recommended, using async context manager with yield) and the deprecated
  on_event("startup") / on_event("shutdown") approach. Trigger when the user mentions:
  FastAPI lifespan, FastAPI startup, FastAPI shutdown, application lifecycle, 应用生命周期,
  启动事件, 关闭事件, 共享资源初始化, ML model loading, database pool setup, or when
  they write @app.on_event.
license: MIT
metadata:
  author: user
  version: "1.0"
---

# FastAPI Lifespan — 应用生命周期事件管理

## When to Use This Skill

Use this skill when you need to:

- **在 FastAPI 应用启动前执行初始化逻辑** — 加载 ML 模型、建立数据库连接池、预热缓存
- **在 FastAPI 应用关闭时执行清理逻辑** — 释放 GPU 资源、关闭连接、写入日志
- **管理跨请求共享的资源** — 避免每个请求都重复加载昂贵资源
- **将资源加载从模块级别移到应用生命周期** — 避免测试时也加载模型

## 推荐方案：lifespan 参数

使用 `lifespan` 参数 + 异步上下文管理器（`@asynccontextmanager`）。

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI


def fake_answer_to_everything_ml_model(x: float):
    return x * 42


ml_models = {}


@asynccontextmanager
async def lifespan(app: FastAPI):
    # yield 之前 → 启动时执行（请求处理之前）
    ml_models["answer_to_everything"] = fake_answer_to_everything_ml_model
    yield
    # yield 之后 → 关闭时执行（请求处理完毕之后）
    ml_models.clear()


app = FastAPI(lifespan=lifespan)


@app.get("/predict")
async def predict(x: float):
    result = ml_models["answer_to_everything"](x)
    return {"result": result}
```

### 执行时序

```
应用启动
    │
    ├─ yield 之前的代码执行（初始化）
    │
    ├─ 应用开始接收请求...
    │
    ├─ 应用处理完所有请求
    │
    └─ yield 之后的代码执行（清理）
```

### 核心概念：异步上下文管理器

`@asynccontextmanager` 将带有 `yield` 的异步函数转换为异步上下文管理器，类似于 Python 的 `with` 语句：

```python
# Python 上下文管理器（同步）
with open("file.txt") as file:
    file.read()

# 异步上下文管理器
async with lifespan(app):
    await do_stuff()
```

FastAPI 的 `lifespan` 参数接收这个上下文管理器，自动管理进入/退出逻辑。

### 典型使用场景

| 场景 | 启动时（yield 前） | 关闭时（yield 后） |
|------|-------------------|-------------------|
| ML 模型 | 从磁盘加载模型 | 释放 GPU 显存 |
| 数据库连接池 | 建立连接池 | 关闭所有连接 |
| 缓存预热 | 加载热数据到 Redis | 清理缓存 |
| 定时任务 | 启动后台调度器 | 优雅停止调度器 |
| 文件写入 | 打开日志文件句柄 | 刷新并关闭文件 |

---

## 弃用方案：on_event("startup") / on_event("shutdown")

**不推荐使用**，但了解它有助于维护旧代码。

### startup 事件

```python
from fastapi import FastAPI

app = FastAPI()
items = {}


@app.on_event("startup")
async def startup_event():
    items["foo"] = {"name": "Fighters"}
    items["bar"] = {"name": "Tenders"}


@app.get("/items/{item_id}")
async def read_items(item_id: str):
    return items[item_id]
```

### shutdown 事件

```python
from fastapi import FastAPI

app = FastAPI()


@app.on_event("shutdown")
def shutdown_event():
    with open("log.txt", mode="a") as log:
        log.write("Application shutdown\n")


@app.get("/items/")
async def read_items():
    return [{"name": "Foo"}]
```

### 同时使用

```python
@app.on_event("startup")
async def startup_event():
    items["foo"] = {"name": "Fighters"}

@app.on_event("shutdown")
def shutdown_event():
    with open("log.txt", mode="a") as log:
        log.write("Application shutdown\n")
```

---

## Gotchas

- **不能混用 lifespan 和 on_event** — 如果你提供了 `lifespan` 参数，`startup`/`shutdown` 事件处理器不会被调用。要么全用 lifespan，要么全用事件。
- **shutdown 事件处理器可以用 `def`（同步）** — 如果只是写文件等 I/O 操作，不需要 `async def`。
- **模块级别加载 ≠ 生命周期加载** — 把资源放在模块顶层会在 `import` 时就加载，包括测试时。使用 `lifespan` 可以让测试跳过加载。
- **yield 的位置很重要** — yield 之前是启动，yield 之后是关闭。不要把启动逻辑放在 yield 之后。
- **底层是 ASGI Lifespan 协议** — 定义了 `startup` 和 `shutdown` 两个事件。

---

## 对比总结

| | lifespan (推荐) | on_event (弃用) |
|---|---|---|
| 语法 | `@asynccontextmanager` + `yield` | `@app.on_event("startup")` / `@app.on_event("shutdown")` |
| 启动/关闭逻辑关联 | 同一个函数，自然关联 | 分散在不同函数，需全局变量 |
| 资源管理 | 上下文管理器自动清理 | 需手动在 shutdown 中清理 |
| 是否推荐 | ✅ 推荐 | ❌ 弃用 |
| 共存 | 不能与 on_event 共存 | 不能与 lifespan 共存 |
