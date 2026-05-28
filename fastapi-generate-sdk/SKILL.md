---
name: fastapi-generate-sdk
description: >
  Use this skill when building FastAPI applications that need auto-generated client SDKs
  (TypeScript, Python, Go, etc.) from OpenAPI specs. Covers Hey API for TypeScript,
  custom operation ID generation, OpenAPI preprocessing for cleaner method names,
  and the full SDK generation workflow. Trigger when the user mentions: FastAPI SDK,
  generate SDK, OpenAPI client generation, TypeScript client, Hey API, openapi-ts,
  生成客户端, 生成 SDK, 操作 ID, operation ID, generate_unique_id_function, or
  when they need to create frontend types/API clients from a FastAPI backend.
license: MIT
metadata:
  author: user
  version: "1.0"
---

# FastAPI Generate SDK — 从 OpenAPI 自动生成客户端

## When to Use This Skill

Use this skill when you need to:

- **为 FastAPI 后端生成 TypeScript SDK** — 前端团队需要类型安全的 API 客户端
- **自动生成多语言客户端** — Python、Go、Java 等语言的 SDK
- **优化操作 ID 使方法名更简洁** — 自定义 `generate_unique_id_function`
- **预处理 OpenAPI JSON 移除标签前缀** — 让生成的客户端方法名更干净
- **保持前后端类型同步** — 后端改模型 → 重新生成 → 前端编译时发现不匹配

---

## 原理

FastAPI 基于 OpenAPI 规范自动生成 API 描述（`/openapi.json`），其中包含：

- 路径操作（endpoints）
- 请求/响应模型（Pydantic models → JSON Schema）
- 操作 ID（每个端点的唯一标识）

SDK 生成器读取这些信息，产出类型安全的客户端代码。

```
FastAPI App  →  /openapi.json  →  SDK Generator  →  Client Code (TS/Py/Go/...)
    ↑                                                      ↓
  Pydantic Models                                    类型安全的 API 调用
```

---

## 工具选型

| 工具 | 适用语言 | 特点 |
|------|----------|------|
| **Hey API** | TypeScript | 专为 TS 优化，npx 一行命令 |
| **OpenAPI Generator** | 50+ 语言 | 通用方案，支持广泛 |
| **Speakeasy** | 多语言 | 商业方案，VC 支持 |
| **Stainless** | 多语言 | 商业方案，企业级 |
| **liblab** | 多语言 | 商业方案 |

> FastAPI 自动生成 OpenAPI **3.1** 规范，确保所用工具支持此版本。

---

## 基础用法：Hey API 生成 TypeScript SDK

### 1. FastAPI 应用（带模型）

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()


class Item(BaseModel):
    name: str
    price: float


class ResponseMessage(BaseModel):
    message: str


@app.post("/items/", response_model=ResponseMessage)
async def create_item(item: Item):
    return {"message": "item received"}


@app.get("/items/", response_model=list[Item])
async def get_items():
    return [
        {"name": "Plumbus", "price": 3},
        {"name": "Portal Gun", "price": 9001},
    ]
```

### 2. 生成 SDK

```bash
npx @hey-api/openapi-ts -i http://localhost:8000/openapi.json -o src/client
```

### 3. 使用生成的客户端

```typescript
// 方法名自动补全
// 请求载荷自动补全（name, price）
// 响应对象自动补全
// 行内错误提示
```

---

## 带标签的应用 + 自定义操作 ID

大型应用中，用 `tags` 分隔不同模块（items、users 等）。

### 问题：默认方法名不简洁

默认生成的方法名如 `createItemItemsPost`（函数名 + 路径 + HTTP 方法），冗余且不直观。

### 解决方案：自定义 `generate_unique_id_function`

```python
from fastapi import FastAPI
from fastapi.routing import APIRoute
from pydantic import BaseModel


def custom_generate_unique_id(route: APIRoute):
    return f"{route.tags[0]}-{route.name}"


app = FastAPI(generate_unique_id_function=custom_generate_unique_id)


class Item(BaseModel):
    name: str
    price: float


class ResponseMessage(BaseModel):
    message: str


class User(BaseModel):
    username: str
    email: str


@app.post("/items/", response_model=ResponseMessage, tags=["items"])
async def create_item(item: Item):
    return {"message": "Item received"}


@app.get("/items/", response_model=list[Item], tags=["items"])
async def get_items():
    return [
        {"name": "Plumbus", "price": 3},
        {"name": "Portal Gun", "price": 9001},
    ]


@app.post("/users/", response_model=ResponseMessage, tags=["users"])
async def create_user(user: User):
    return {"message": "User received"}
```

生成后客户端按标签分组为 `ItemsService` 和 `UsersService`，方法名变为 `items-create_item` 等。

---

## 进一步优化：预处理 OpenAPI 移除标签前缀

方法名 `items-get_items` 中 `items-` 前缀在 `ItemsService` 上下文里是冗余的。

### 步骤 1：下载 OpenAPI JSON

```bash
curl http://localhost:8000/openapi.json -o openapi.json
```

### 步骤 2：移除标签前缀

```python
import json
from pathlib import Path

file_path = Path("./openapi.json")
openapi_content = json.loads(file_path.read_text())

for path_data in openapi_content["paths"].values():
    for operation in path_data.values():
        tag = operation["tags"][0]
        operation_id = operation["operationId"]
        to_remove = f"{tag}-"
        new_operation_id = operation_id[len(to_remove):]
        operation["operationId"] = new_operation_id

file_path.write_text(json.dumps(openapi_content))
```

操作 ID 从 `items-get_items` → `get_items`，从 `items-create_item` → `create_item`。

### 步骤 3：用预处理后的文件生成

```bash
npx @hey-api/openapi-ts -i ./openapi.json -o src/client
```

最终方法名：`ItemsService.createItem(...)` — 简洁直观。

---

## 完整工作流

```yaml
开发:
  1. 编写 FastAPI 应用（Pydantic 模型 + 路径操作 + tags）
  2. 自定义 generate_unique_id_function（标签-函数名 格式）
  3. 启动应用 → /openapi.json 自动生成

生成:
  4. 下载 openapi.json 到本地文件
  5. 运行预处理脚本移除标签前缀（可选）
  6. npx @hey-api/openapi-ts 生成客户端代码

迭代:
  7. 修改后端 → 重新生成客户端
  8. 前端构建 → 编译时报错发现不匹配
  9. 早期发现 bug，避免生产环境暴露
```

---

## Gotchas

- **操作 ID 必须全局唯一** — 自定义 `generate_unique_id_function` 时务必保证唯一性，否则 OpenAPI 校验失败。
- **标签前缀移除是可选的** — 保留标签前缀能确保操作 ID 唯一，移除它需要你确保 `{tag}-{name}` 去前缀后仍不冲突。
- **OpenAPI 版本** — FastAPI 生成 3.1 规范，老旧工具可能只支持 3.0，注意兼容性。
- **重新生成覆盖** — `-o src/client` 会覆盖整个目录，不要在生成目录中手写代码。
- **运行时需要 /openapi.json** — 如果从 URL 生成，确保 FastAPI 应用正在运行；如果从文件生成，确保文件是最新的。

---

## 收益总结

| 收益 | 说明 |
|------|------|
| 方法名自动补全 | IDE 中列出所有可用 API 方法 |
| 请求载荷自动补全 | Pydantic 模型字段 → TypeScript 类型 |
| 响应类型自动补全 | 返回值的字段名和类型全有 |
| 行内错误提示 | 传错字段或类型 → IDE 即时标红 |
| 编译时发现不匹配 | 后端改字段 → 前端构建报错，不等运行时 |
| 按标签分组 | `ItemsService` / `UsersService` 模块化组织 |
