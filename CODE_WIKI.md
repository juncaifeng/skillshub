# Skillshub 项目 Code Wiki

## 目录
1. [项目概览](#项目概览)
2. [系统架构](#系统架构)
3. [技能模块详解](#技能模块详解)
4. [选型指南](#选型指南)
5. [使用流程](#使用流程)
6. [最佳实践](#最佳实践)

---

## 项目概览

Skillshub 是一个 Agent Skills 集合仓库，按照 [agentskills.io](https://agentskills.io) 规范编写，用于快速搭建基于 gRPC 和 API Gateway 的微服务架构。

### 核心思想
- **统一 API Gateway 模式**：单一入口对外暴露 HTTP 接口
- **后端纯 gRPC 服务**：所有微服务仅提供 gRPC 接口
- **分层架构**：HTTP 请求 → Gateway → 内部 gRPC 调用

### 项目结构
```
skillshub/
├── buf-add-grpc-svc/       # 新建纯 gRPC 后端服务
├── buf-add-micro/          # 新建独立微服务（BFF）
├── buf-add-rpc/            # 给已有服务追加 RPC 方法
├── buf-add-service/        # 同进程内新增 gRPC 服务
├── buf-grpc-gateway-scaffold/  # API Gateway 脚手架
├── buf-maintain/           # buf 项目日常维护
├── fastapi-generate-sdk/   # FastAPI SDK 生成
├── fastapi-lifespan/       # FastAPI 生命周期管理
├── git-submodule-add/      # Git 子模块管理
├── .gitmodules
├── .gitignore
└── README.md
```

---

## 系统架构

### API Gateway 模式架构图

```
                    ┌─────────────────────┐
                    │ 外部 HTTP 请求      │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   API Gateway       │
                    │  (:8080 HTTP)       │
                    │  (:9090 gRPC)       │
                    └─────┬──────┬────────┘
                          │      │
              ┌───────────┴──┬───┴───────────┐
              │              │               │
┌─────────────▼──────┐ ┌────▼────────┐ ┌───▼──────────┐
│  auth-service     │ │order-service│ │payment-service│
│  (:9091 gRPC)     │ │(:9092 gRPC) │ │(:9093 gRPC)  │
└───────────────────┘ └─────────────┘ └──────────────┘
```

### 架构特点
1. **统一入口**：Gateway 是唯一的 HTTP 对外接口
2. **服务发现**：Gateway 内部维护对后端服务的 gRPC 连接
3. **认证授权**：统一在 Gateway 层处理
4. **限流熔断**：可在 Gateway 层统一实现
5. **后端解耦**：后端服务仅需实现 gRPC，无需关心 HTTP 转码

---

## 技能模块详解

### 1. buf-grpc-gateway-scaffold - API Gateway 脚手架

**用途**：创建 API Gateway 项目，作为整个系统的唯一 HTTP 入口。

**生成的目录结构**：
```
gateway/
├── api/                    # proto 文件（含 HTTP 注解）
│   └── v1/
│       ├── greeter.proto
│       └── common.proto
├── gen/                    # 生成代码（.gitignore）
├── server/grpc/            # gRPC 服务实现
├── cmd/server/             # 主程序入口
├── config/                 # 配置文件
├── buf.yaml                # buf 配置
├── buf.gen.yaml            # 代码生成配置
├── go.mod
├── go.sum
└── Makefile
```

**核心文件说明**：

- `buf.yaml`：定义 proto 模块、lint 规则、breaking change 检测
- `buf.gen.yaml`：配置代码生成插件（protobuf、grpc、grpc-gateway）
- `cmd/server/main.go`：启动 gRPC server 和 HTTP Gateway，注册外部服务代理
- `server/grpc/greeter.go`：gRPC 服务实现示例

**buf.gen.yaml 配置**：
```yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen
    opt:
      - paths=source_relative
  - remote: buf.build/grpc/go
    out: gen
    opt:
      - paths=source_relative
  - remote: buf.build/grpc-ecosystem/gateway:v2.25.0
    out: gen
    opt:
      - paths=source_relative
      - generate_unbound_methods=true
```

---

### 2. buf-add-grpc-svc - 新建纯 gRPC 后端服务

**用途**：创建挂载在 Gateway 后面的纯 gRPC 后端服务（无 HTTP 端口，不依赖 grpc-gateway）。

**与其他技能的区别**：

| 特性 | buf-add-service | buf-add-micro | buf-add-grpc-svc |
|------|-----------------|---------------|------------------|
| HTTP Gateway | 注册到已有 | 自建一套 | ❌ 无 |
| HTTP 端口 | 共用 | :808N | ❌ 无 |
| go.mod | 共用 | 独立 | 独立 |
| gRPC 端口 | 共用 | :909N | :909N |
| 认证 | 网关统一 | 需自配 | 网关统一 |
| 依赖 grpc-gateway | ✅ | ✅ | ❌ |
| proto 含 HTTP 注解 | ✅ | ✅ | ❌ |

**生成的目录结构**：
```
auth-service/
├── api/v1/
│   └── auth.proto          # 无 HTTP 注解
├── gen/                    # 生成代码（无 .pb.gw.go）
├── server/                 # gRPC 服务实现
├── cmd/server/             # 仅启动 gRPC server
├── config/
├── buf.yaml                # 无需 googleapis 依赖
├── buf.gen.yaml            # 无需 gateway 插件
├── go.mod
└── Makefile
```

**接入 Gateway 的步骤**：
1. 在 gateway 的 `go.mod` 中添加 `require` + `replace`
2. 在 gateway 的 `main.go` 中建立 gRPC 连接并注册 Handler
3. 配置环境变量指向后端服务地址

---

### 3. buf-add-service - 同进程内新增 gRPC 服务

**用途**：在已有的 monolith 项目（单一 go.mod、单一 main.go）中新增 gRPC 服务。

**检查清单**：
- [x] 创建 Proto 文件 `api/v<N>/<service>.proto`
- [x] `buf generate` 生成代码
- [x] 实现服务 `server/grpc/<service>.go`
- [x] 注册到 `cmd/server/main.go`（gRPC + HTTP 两处）
- [x] 编译验证 `go build ./cmd/server`
- [x] 测试 `curl` 验证 HTTP endpoint

**关键要点**：
- 不需要修改 `go.mod`（与现有服务共用）
- 在 `main.go` 中**两重注册**：gRPC server 和 HTTP mux
- proto 文件需包含 HTTP 注解

---

### 4. buf-add-rpc - 给已有服务追加 RPC 方法

**用途**：在已有的 gRPC 服务中添加新的 RPC 方法和对应的 HTTP 接口。

**检查清单**：
- [x] 在 `api/v<N>/<service>.proto` 中新增 RPC 定义 + HTTP 注解
- [x] `buf generate` 重新生成
- [x] 在 `server/grpc/<service>.go` 中实现新方法
- [x] 编译验证 `go build ./cmd/server`

**HTTP 方法约定**：
| HTTP 方法 | 典型场景 |
|-----------|----------|
| GET | 查询单个/列表 |
| POST | 创建、自定义操作 |
| PUT | 全量更新 |
| PATCH | 部分更新 |
| DELETE | 删除 |

---

### 5. buf-add-micro - 新建独立微服务（BFF 模式）

**用途**：创建带自有 HTTP Gateway 的独立微服务（独立 go.mod、独立部署）。

**⚠️ 注意**：大多数场景请使用 `buf-add-grpc-svc`，本技能仅适用于：
- BFF (Backend For Frontend) 场景
- 独立对外暴露的服务
- 不需要统一 Gateway 的场景

**端口规划建议**：
| 服务 | gRPC | HTTP |
|------|------|------|
| user-service | :9091 | :8081 |
| order-service | :9092 | :8082 |
| payment-service | :9093 | :8083 |
| ... | :909N | :808N |

---

### 6. buf-maintain - buf 项目日常维护

**用途**：lint、breaking change 检测、代码生成、依赖更新等日常操作。

**操作清单**：

| # | 操作 | 命令 | 何时使用 |
|---|------|------|----------|
| 1 | Lint | `buf lint` | proto 修改后、PR 前 |
| 2 | Breaking Check | `buf breaking --against '.git#branch=main'` | API 变更后、发版前 |
| 3 | Regenerate | `buf generate` | proto 修改后 |
| 4 | Dep Update | `buf mod update` | 依赖过期时 |
| 5 | Go Build | `go build ./cmd/server` | 生成后验证 |
| 6 | Clean | `rm -rf gen/` | 疑难杂症、重新生成 |

**常见 Breaking Change**：
| 变更 | Breaking? |
|------|-----------|
| 新增 RPC 方法 | ❌ 否 |
| 删除 RPC 方法 | ✅ 是 |
| 改名 RPC 方法 | ✅ 是 |
| 新增 message 字段 | ❌ 否 |
| 删除 message 字段 | ✅ 是 |
| 改名 message 字段 | ✅ 是 |
| 改变字段类型 | ✅ 是 |
| 改变字段编号 | ✅ 是 |

---

### 7. git-submodule-add - Git 子模块管理

**用途**：添加外部 Git 仓库作为子模块，统一放在 `ref/` 目录下。

**目录约定**：
```
root/
├── ref/                    # 所有子模块统一入口
│   ├── project-a/          # git submodule
│   ├── project-b/          # git submodule
│   └── ...
```

**常用命令**：
```bash
# 添加子模块
git submodule add https://github.com/<owner>/<repo> ref/<repo>

# 克隆含子模块的项目
git clone --recurse-submodules <project-url>

# 初始化已克隆项目的子模块
git submodule update --init --recursive

# 更新子模块到最新
git submodule update --remote

# 移除子模块
git submodule deinit -f ref/<repo>
rm -rf ref/<repo>
rm -rf .git/modules/ref/<repo>
```

---

### 8. fastapi-lifespan - FastAPI 生命周期管理

**用途**：管理 FastAPI 应用的启动/关闭事件，初始化和清理共享资源。

**推荐方案：lifespan 参数**
```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

ml_models = {}

@asynccontextmanager
async def lifespan(app: FastAPI):
    # yield 之前 → 启动时执行
    ml_models["answer_to_everything"] = load_ml_model()
    yield
    # yield 之后 → 关闭时执行
    ml_models.clear()

app = FastAPI(lifespan=lifespan)
```

**弃用方案：on_event（但仍需了解）**
```python
@app.on_event("startup")
async def startup_event():
    pass

@app.on_event("shutdown")
def shutdown_event():
    pass
```

**⚠️ 注意**：不能混用 `lifespan` 和 `on_event`。

---

### 9. fastapi-generate-sdk - FastAPI SDK 生成

**用途**：从 FastAPI 的 OpenAPI 规范自动生成类型安全的客户端 SDK（TypeScript、Python、Go 等）。

**推荐工具：Hey API (TypeScript)**
```bash
# 从运行中的 FastAPI 应用生成
npx @hey-api/openapi-ts -i http://localhost:8000/openapi.json -o src/client

# 或从本地文件生成
npx @hey-api/openapi-ts -i ./openapi.json -o src/client
```

**自定义操作 ID 使方法名更简洁**：
```python
from fastapi import FastAPI
from fastapi.routing import APIRoute

def custom_generate_unique_id(route: APIRoute):
    return f"{route.tags[0]}-{route.name}"

app = FastAPI(generate_unique_id_function=custom_generate_unique_id)
```

---

## 选型指南

### 流程图决策
```
需要 HTTP 对外?
├─ 是 → 已有 Gateway?
│   ├─ 是 → 纯后端服务 → buf-add-grpc-svc
│   └─ 否 → 创建 gateway → buf-grpc-gateway-scaffold
│          → 后端服务 → buf-add-grpc-svc
└─ 否 → 同进程扩展 → buf-add-service
        或特殊场景（BFF/独立）→ buf-add-micro
```

### 详细对比
| 场景 | 推荐技能 | 理由 |
|------|----------|------|
| 从零开始搭建整套系统 | buf-grpc-gateway-scaffold + buf-add-grpc-svc | 统一 Gateway + 纯后端微服务 |
| 给已有 Gateway 加新的后端服务 | buf-add-grpc-svc | 简单、解耦、认证在 Gateway |
| 给现有 monolith 加功能 | buf-add-service | 共用 go.mod、无需额外部署 |
| 给已有服务加接口 | buf-add-rpc | 最小改动，无需动 main.go |
| BFF 层或需要独立 HTTP 入口 | buf-add-micro | 自包含，适合特定场景 |
| 日常维护 | buf-maintain | 标准化的 lint/breaking check/generate |

---

## 使用流程

### 完整项目搭建流程

#### 1. 初始化 Gateway
```bash
# 创建项目结构
mkdir -p gateway/api/v1 gateway/cmd/server gateway/server/grpc gateway/gen gateway/config
cd gateway

# 初始化 go module
go mod init gateway

# 编写 buf.yaml、buf.gen.yaml、proto 文件、main.go、greeter.go
# ...（参考 buf-grpc-gateway-scaffold 技能）

# 初始化项目
make init
make generate
make build
```

#### 2. 新建后端服务
```bash
# 在 gateway 同级创建
cd ..
mkdir -p auth-service/api/v1 auth-service/gen auth-service/server auth-service/cmd/server auth-service/config
cd auth-service

go mod init auth-service

# 编写 proto（无 HTTP 注解）、buf 配置、服务实现、main.go
# ...（参考 buf-add-grpc-svc 技能）

make init
make generate
make build
```

#### 3. 接入 Gateway
```bash
cd ../gateway

# 修改 go.mod，添加 replace
go mod edit -require=auth-service@v0.0.0
go mod edit -replace=auth-service=../auth-service
go mod tidy

# 修改 main.go，添加 auth-service 的连接和注册
# ...（参考 buf-grpc-gateway-scaffold 接入外部服务部分）

make build
```

#### 4. 启动测试
```bash
# 终端 1：启动 auth-service
cd ../auth-service
GRPC_ADDR=:9091 make run

# 终端 2：启动 gateway
cd ../gateway
AUTH_SVC_ADDR=localhost:9091 make run

# 测试
curl http://localhost:8080/v1/hello  # 测试 gateway 自有接口
curl http://localhost:8080/v1/auth/login  # 测试代理到 auth-service 的接口
```

---

## 最佳实践

### 1. Gateway 作为唯一入口
- 不要在后端服务上挂 HTTP gateway
- 认证、限流、鉴权统一在 Gateway 层处理
- 后端服务只关注业务逻辑

### 2. 版本管理
- proto 文件按版本组织在 `api/v1/`、`api/v2/` 下
- 多个版本可共存，实现平滑升级

### 3. 环境变量配置
- 端口、服务地址等通过环境变量注入，不要硬编码
- 示例：`GRPC_ADDR`、`HTTP_ADDR`、`AUTH_SVC_ADDR`

### 4. CI/CD 集成
- PR 时运行 `buf lint` + `buf breaking` 检查
- 自动 `buf generate` + `go build` 验证
- 使用 GitHub Actions 等自动化工具

### 5. Git 管理
- `gen/` 目录加入 `.gitignore`，不要提交生成的代码
- 子模块 URL 使用 HTTPS 而非 SSH，提高兼容性
- `buf.lock` 提交到仓库，锁定依赖版本

### 6. proto 规范
- 包名 `package api.v1`
- `go_package` 格式 `<module>/gen/v1;v1`
- 使用 `common.proto` 共享通用类型（如 `PageRequest`、`PageResponse`）
- 每个 RPC 定义独立的 Request/Response 类型

### 7. 端口规划
预先约定端口范围，避免冲突：
| 用途 | 端口范围 |
|------|----------|
| Gateway HTTP | :8080 |
| Gateway gRPC | :9090 |
| 后端服务 gRPC | :9091 - :9099 |
| 独立微服务 HTTP | :8081 - :8089 |

---

## 依赖关系

### 核心依赖
- **Go 1.21+**
- **buf CLI** - proto 管理工具
- **Git** - 版本控制
- **gRPC** - RPC 框架
- **grpc-gateway** - HTTP → gRPC 转码

### buf 远程插件
```yaml
plugins:
  - buf.build/protocolbuffers/go
  - buf.build/grpc/go
  - buf.build/grpc-ecosystem/gateway:v2.25.0
```

### Go 依赖（Gateway）
```go
require (
    github.com/grpc-ecosystem/grpc-gateway/v2 v2.25.0
    google.golang.org/grpc v1.68.0
)
```

### Go 依赖（纯 gRPC 后端）
```go
require (
    google.golang.org/grpc v1.68.0
)
```

---

## Gotchas（避坑指南）

### buf 相关
- `buf generate` 前必须先 `go mod init`
- `go_package` 必须与 `go.mod` 的 module 路径一致
- `gen/` 目录必须加入 `.gitignore`
- `buf.build` 远程插件需要网络，离线环境需改用本地插件

### Gateway 相关
- 新增外部服务后，gateway 的 `go.mod` 需要同时有 `require` 和 `replace`
- 每个外部服务独立建立 `grpc.DialContext` 连接
- 生产环境不要用 `insecure.NewCredentials()`，改用 TLS
- 注意 HTTP 路径冲突，合理规划路径前缀

### 后端服务相关
- 纯 gRPC 服务的 proto 不要 import `google/api/annotations.proto`
- 多个后端服务端口不能冲突
- 跨服务调用直接用 gRPC client，不要绕 Gateway

### FastAPI 相关
- 不能混用 `lifespan` 和 `on_event`
- 模块级别的资源加载会在 import 时执行（包括测试），用 `lifespan` 避免
- FastAPI 生成 OpenAPI 3.1 规范，确保 SDK 生成工具支持

### Git 子模块相关
- URL 使用 HTTPS 而非 SSH
- `.gitmodules` 必须提交到仓库
- 主仓库只记录子模块的 commit hash，不记录分支
- 移除子模块时，不要忘记清理 `.git/modules/` 下的缓存

---

## 参考资源

- [Buf 官方文档](https://buf.build/docs)
- [gRPC 官方文档](https://grpc.io/docs/)
- [grpc-gateway 官方文档](https://grpc-ecosystem.github.io/grpc-gateway/)
- [FastAPI 官方文档](https://fastapi.tiangolo.com/)
- [agentskills.io 规范](https://agentskills.io)
