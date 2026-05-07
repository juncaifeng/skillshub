---
name: buf-maintain
description: >
  Perform routine buf + grpc-gateway maintenance: lint proto files, check
  for breaking changes, regenerate generated code, update buf dependencies,
  and run Go builds. Use this skill when the user says "check the protos",
  "run lint", "breaking change check", "regenerate", "update deps", "buf
  mod update", or any maintenance / CI-related request in a buf gRPC
  project.
compatibility: Requires an existing buf + grpc-gateway project.
metadata:
  author: skillshub
  version: "1.0"
---

# buf 日常维护

封装 buf 项目的日常维护操作，按需选择执行步骤。

## 操作清单

| # | 操作 | 命令 | 何时使用 |
|---|------|------|---------|
| 1 | Lint | `buf lint` | proto 修改后、PR 前 |
| 2 | Breaking Check | `buf breaking --against '.git#branch=main'` | API 变更后、发版前 |
| 3 | Regenerate | `buf generate` | proto 修改后 |
| 4 | Dep Update | `buf mod update` | 依赖过期时 |
| 5 | Go Build | `go build ./cmd/server` | 生成后验证 |
| 6 | Clean | `rm -rf gen/` | 疑难杂症、重新生成 |

除非用户明确要求，否则**不要**主动 clean gen/ 目录。

---

## 1. Lint

```bash
buf lint
```

检查 proto 文件的命名、格式、最佳实践。常见报错：

| 错误 | 原因 | 修复 |
|------|------|------|
| `PACKAGE_VERSION_SUFFIX` | 包名不以 `.v1` 结尾 | 已在 `buf.yaml` 中 `except` 掉 |
| `RPC_REQUEST_RESPONSE_UNIQUE` | 多个 RPC 共用同一个 message | 每个 RPC 定义独立的 Request/Response |
| `RPC_REQUEST_STANDARD_NAME` | Request 命名不是 `<RpcName>Request` | 按规范重命名 |
| `FIELD_LOWER_SNAKE_CASE` | 字段用了驼峰命名 | 改用 lower_snake_case |
| `ENUM_PASCAL_CASE` | 枚举值不是全大写+下划线 | 改为 `ENUM_VALUE_NAME` |
| `DIRECTORY_SAME_PACKAGE` | 同目录下不同 proto 用了不同 package | 统一 package 声明 |

如需放宽规则，在 `buf.yaml` 的 `lint.except` 中添加。

---

## 2. Breaking Change Check

```bash
buf breaking --against '.git#branch=main'
```

检测当前修改是否破坏了向后兼容性。CI 中通常配置为失败即阻断。

常见 breaking change：

| 变更 | Breaking? | 说明 |
|------|-----------|------|
| 新增 RPC 方法 | 否 | 新增不会破坏现有客户端 |
| 删除 RPC 方法 | 是 | 现有客户端调用会失败 |
| 改名 RPC 方法 | 是 | 旧名不可用 |
| 新增 message 字段 | 否 | 只要不是 required |
| 删除 message 字段 | 是 | 现有客户端发送/接收会丢数据 |
| 改名 message 字段 | 是 | 等同于删除+新增 |
| 改变字段类型 | 是 | 序列化不兼容 |
| 改变字段编号 | 是 | 二进制不兼容 |
| 新增 enum 值 | 否* | 但需注意接收侧处理未知值 |
| 删除 enum 值 | 是 | 已序列化数据无法解析 |

> 为确保安全，在 `buf.yaml` 的 `breaking.use` 上至少使用 `FILE` 级别。

---

## 3. Regenerate

```bash
buf generate
```

重新生成所有 gen/ 下的代码。执行前确保 `buf.gen.yaml` 存在且 go 模块已初始化。

生成成功的标志：每个 proto 对应三个文件:
- `<name>.pb.go` — 消息序列化
- `<name>_grpc.pb.go` — gRPC 接口
- `<name>.pb.gw.go` — HTTP Gateway handler

---

## 4. 更新依赖

```bash
buf mod update
```

从 BSR (buf.build) 拉取最新的 proto 依赖，更新 `buf.lock`。

Go 侧依赖同步更新：

```bash
go get -u github.com/grpc-ecosystem/grpc-gateway/v2
go get -u google.golang.org/grpc
go mod tidy
```

---

## 5. Go Build

```bash
go build ./cmd/server
```

兜底校验，确保 Go 侧代码与生成的 pb 包一致。

---

## 6. Clean（慎用）

```bash
rm -rf gen/
buf generate
```

仅在以下情况使用：
- `buf generate` 报错且不好排查
- 切换了 buf 插件版本（如 `gateway:v2.24` → `v2.25`）
- 目录结构有较大调整

---

## CI/CD 模板参考

```yaml
# GitHub Actions 示例
steps:
  - uses: actions/checkout@v4
  - uses: bufbuild/buf-setup-action@v1
  - run: buf lint
  - run: buf breaking --against 'https://github.com/${{ github.repository }}.git#branch=main'
  - run: buf generate
  - run: go build ./cmd/server
```

---

## Gotchas

- `buf breaking` 需要 Git 历史中有比较基准分支（通常是 `main`），所以需要 `fetch-depth: 0` 在 CI 中
- `buf generate` 在 Windows 上可能因路径分隔符报错，确保 `paths=source_relative`
- 远程插件需要网络，离线环境改用 `local:` 指定本地安装的 protoc 插件
- `buf lint` 默认规则较严格，项目初期建议在 `buf.yaml` 中 `except` 掉不适用于团队的规则
