---
name: buf-add-rpc
description: >
  Add one or more RPC methods to an existing gRPC service in a buf +
  grpc-gateway project. Handles proto updates, code regeneration, and
  server implementation. Use this skill when the user wants to add a new
  endpoint, extend an existing service with new methods, add RPCs, or
  mentions "加一个接口" / "新增方法" on an existing service in a
  buf gRPC project.
compatibility: Requires an existing buf + grpc-gateway project with at least one service already registered.
metadata:
  author: skillshub
  version: "1.0"
---

# 给已有服务新增 RPC 方法

在已有 gRPC 服务上追加新方法。

## 检查清单

- [ ] Step 1: 在 `api/v<N>/<service>.proto` 中新增 RPC 定义 + HTTP 注解
- [ ] Step 2: `buf generate` 重新生成
- [ ] Step 3: 在 `server/grpc/<service>.go` 中实现新方法
- [ ] Step 4: 编译验证 `go build ./cmd/server`

> 与 `buf-add-service` 的区别：不需要改 `main.go`，因为该 service 已经注册过了。

---

## Step 1: 追加 RPC 定义

在已有 proto 的 service 块中新增方法。先读取现有文件，找到 `service <Name>` 块，在最后一个 RPC 之后添加。

**示例 — 给 Order 服务加退款接口：**

```protobuf
service Order {
  // ... 已有方法 ...

  rpc RefundOrder (RefundOrderRequest) returns (RefundOrderResponse) {
    option (google.api.http) = {
      post: "/v1/orders/{id}/refund"
      body: "*"
    };
  }
}

message RefundOrderRequest {
  string id = 1;
  string reason = 2;
}

message RefundOrderResponse {
  string refund_id = 1;
  string status = 2;
}
```

### RPC 定义速查

| HTTP 方法 | proto 注解 | 典型场景 |
|-----------|-----------|---------|
| GET | `get: "/v1/orders/{id}"` | 查询单个 |
| GET | `get: "/v1/orders"` | 列表查询 |
| POST | `post: "/v1/orders"` + `body: "*"` | 创建（整个 body 绑定）|
| POST | `post: "/v1/orders/{id}/activate"` | 自定义操作 |
| PUT | `put: "/v1/orders/{id}"` + `body: "*"` | 全量更新 |
| PATCH | `patch: "/v1/orders/{id}"` + `body: "*"` | 部分更新 |
| DELETE | `delete: "/v1/orders/{id}"` | 删除 |

### 请求体绑定

- `body: "*"` — 整个 HTTP body 映射到 request message（最常用）
- `body: "field_name"` — 只映射指定字段
- 不写 body — 仅从 URL path / query string 取值

### URL 路径变量

`{id}` 自动从 request message 的同名字段（如 `string id = 1`）取值，类型自动转换。

---

## Step 2: 重新生成

```bash
buf generate
```

只会更新已改变的 proto 对应的 pb 文件，已有方法不受影响。

---

## Step 3: 实现新方法

在已有 `server/grpc/<service>.go` 文件中追加方法实现：

```go
func (s *<ServiceName>Server) RefundOrder(ctx context.Context, req *pb.RefundOrderRequest) (*pb.RefundOrderResponse, error) {
    // TODO: 实现退款逻辑
    return &pb.RefundOrderResponse{
        RefundId: "refund-" + req.Id,
        Status:   "pending",
    }, nil
}
```

**注意：** 追加到现有 struct 的实现中，不要创建新的 struct。

---

## Step 4: 编译验证

```bash
go build ./cmd/server
```

常见编译错误及原因：
- `method <MethodName> already declared` → 该方法已存在
- `missing method <MethodName>` → proto 中定义了但没实现（需要全部实现）
- `undefined: pb.<Type>` → `buf generate` 未执行或失败

---

## Gotchas

- 只需改 proto + server 实现 + 重新生成，不需要动 `main.go`（service 已注册）
- 新增 message 定义可以放在 proto 文件底部，也可新建独立文件通过 import 引入
- 如果 message 体积大或会被多个 proto 共享，考虑放在 `common.proto` 或独立文件中
- `buf generate` 是幂等的：重复执行不影响已有代码
- 方法签名中的 `req` 类型必须与 proto 中定义的 Request message 一致，拼写大小写也要完全匹配
