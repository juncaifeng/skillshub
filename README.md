# skillshub

Agent Skills 集合，按 agentskills.io 规范编写。

## 架构模式

**API Gateway 模式**：一个 Gateway 统一对外，后端服务纯 gRPC。

```
HTTP 请求 → Gateway (:8080) ──gRPC── ▶ auth-service (:9091)
                            ──gRPC── ▶ order-service (:9092)
                            ──gRPC── ▶ payment-service (:9093)
```

## 现有 Skills

| 技能 | 场景 | 说明 |
|------|------|------|
| `buf-grpc-gateway-scaffold` | 项目初始化 | 创建统一 API Gateway（唯一 HTTP 入口），支持代理外部 gRPC 服务 |
| `buf-add-grpc-svc` | 新建后端服务 | 纯 gRPC 后端服务（无 HTTP，挂载在 Gateway 后面）⭐ 推荐 |
| `buf-add-service` | monolith 扩展 | 同进程内新增 gRPC 服务（共用 go.mod/main.go/端口） |
| `buf-add-rpc` | 追加接口 | 给已有服务追加 RPC 方法 |
| `buf-add-micro` | BFF 场景 | 独立微服务（自带 HTTP Gateway，仅用于独立对外场景） |
| `buf-maintain` | 日常维护 | lint / breaking check / generate / mod update / build |

### 通用工具

| 技能 | 说明 |
|------|------|
| `git-submodule-add` | 添加 GitHub 仓库为子模块，统一放到 `ref/` 下 |

### 选型指南

```
需要 HTTP 对外吗？
├── 是 → 已有 Gateway？
│   ├── 是 → 纯后端服务用 buf-add-grpc-svc
│   └── 否 → 创建 gateway 用 buf-grpc-gateway-scaffold
│            → 后端服务用 buf-add-grpc-svc
└── 否 → 同进程扩展用 buf-add-service

特殊场景：服务需要独立 HTTP 入口（BFF/第三方回调）→ buf-add-micro
```

## 规范参考

`reference/` 为 [agentskills.io](https://agentskills.io) 规范子模块，包含完整的 Skill 格式定义、最佳实践和参考实现。
