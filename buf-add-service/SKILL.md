---
name: buf-add-service
description: >
  Add a complete new gRPC service to an existing buf + grpc-gateway project.
  Covers proto file creation, code generation, gRPC server implementation,
  HTTP gateway registration, and main.go wiring. Use this skill when the user
  wants to add a new service, create a new API resource, add a module like
  "订单服务" or "用户服务", or mentions "新增服务" in a buf gRPC project.
compatibility: Requires an existing buf + grpc-gateway project scaffolded with buf-grpc-gateway-scaffold.
metadata:
  author: skillshub
  version: "1.0"
---

# 新增 gRPC 服务

在已有的 buf + grpc-gateway 项目中新增一个完整的 gRPC 服务。

## 检查清单

- [ ] Step 1: 创建 Proto 文件 `api/v<N>/<service>.proto`
- [ ] Step 2: `buf generate` 生成代码
- [ ] Step 3: 实现服务 `server/grpc/<service>.go`
- [ ] Step 4: 注册到 `cmd/server/main.go`（gRPC + HTTP 两处）
- [ ] Step 5: 编译验证 `go build ./cmd/server`
- [ ] Step 6: 测试 `curl` 验证 HTTP endpoint

---

## Step 1: 创建 Proto 文件

用户需提供：服务名（如 `order`）、版本号（如 `v1`）、RPC 方法列表。

Proto 模板（`api/v<N>/<service>.proto`）：

```protobuf
syntax = "proto3";

package api.<version>;

option go_package = "<module>/gen/<version>;<version>";

import "google/api/annotations.proto";

service <ServiceName> {
  // (在此添加 RPC 方法)
}

// (在此添加 message 定义)
```

**占位符替换规则：**
- `<module>` — 从 `go.mod` 第一行 `module xxx` 提取
- `<version>` — 用户指定的版本号（如 `v1`）
- `<ServiceName>` — 首字母大写的 PascalCase 服务名（如 `Order`）
- 文件名 — 小写下划线（如 `order.proto`）

### 常用 import 参考

```protobuf
import "google/api/annotations.proto";   // HTTP 注解（gateway 必须）
import "google/api/field_behavior.proto"; // 字段约束（可选）
import "api/v1/common.proto";             // 项目内共享类型（可选）
```

### 完整示例（订单服务）

```protobuf
syntax = "proto3";

package api.v1;

option go_package = "my-project/gen/v1;v1";

import "google/api/annotations.proto";
import "api/v1/common.proto";

service Order {
  rpc CreateOrder (CreateOrderRequest) returns (CreateOrderResponse) {
    option (google.api.http) = {
      post: "/v1/orders"
      body: "*"
    };
  }

  rpc GetOrder (GetOrderRequest) returns (GetOrderResponse) {
    option (google.api.http) = {
      get: "/v1/orders/{id}"
    };
  }

  rpc ListOrders (PageRequest) returns (ListOrdersResponse) {
    option (google.api.http) = {
      get: "/v1/orders"
    };
  }
}

message CreateOrderRequest {
  string product_id = 1;
  int32 quantity = 2;
}

message CreateOrderResponse {
  string order_id = 1;
}

message GetOrderRequest {
  string id = 1;
}

message GetOrderResponse {
  string order_id = 1;
  string product_id = 2;
  int32 quantity = 3;
  string status = 4;
  int64 created_at = 5;
}

message ListOrdersResponse {
  repeated GetOrderResponse orders = 1;
}
```

Uses `PageRequest` from `common.proto`. HTTP method 约定：POST 创建、GET 单个/列表、PUT/PATCH 更新、DELETE 删除。

不要重复定义已在 `common.proto` 中定义的 PageRequest/PageResponse。

---

## Step 2: 生成代码

```bash
buf generate
```

验证生成结果：

```bash
ls gen/<version>/<service>.pb.go          # protobuf 消息
ls gen/<version>/<service>_grpc.pb.go     # gRPC 服务端/客户端
ls gen/<version>/<service>.pb.gw.go       # grpc-gateway
```

> 如果 `generate_unbound_methods=true` 已启用，即使 proto 中暂时没有写 HTTP 注解也会生成 gw.go。

---

## Step 3: 实现 gRPC 服务

创建 `server/grpc/<service>.go`（文件名用下划线小写）：

```go
package grpc

import (
    "context"

    pb "<module>/gen/<version>"
)

type <ServiceName>Server struct {
    pb.Unimplemented<ServiceName>Server
}

func (s *<ServiceName>Server) <MethodName>(ctx context.Context, req *pb.<RequestType>) (*pb.<ResponseType>, error) {
    // TODO: 实现业务逻辑
    return &pb.<ResponseType>{}, nil
}
```

**占位符：**
- `<ServiceName>` — 如 `Order`
- `<MethodName>` — 如 `CreateOrder`
- `<RequestType>` / `<ResponseType>` — 对应 message 名

每个 RPC 方法都需要实现。未实现的方法由 `UnimplementedXxxServer` 返回 `codes.Unimplemented`。

---

## Step 4: 注册到 main.go

在 `cmd/server/main.go` 中**两处**注册：

### gRPC 端（注册 server 实现）

```go
pb.Register<ServiceName>Server(grpcServer, &grpc.<ServiceName>Server{})
```

### HTTP Gateway 端（注册 handler）

```go
if err := pb.Register<ServiceName>Handler(ctx, mux, conn); err != nil {
    panic(err)
}
```

两处放在各自已有关闭逻辑之后。

---

## Step 5: 编译验证

```bash
go build ./cmd/server
```

编译通过才算完成。常见错误：
- 遗漏 import：新生成的 pb 包需要在 `main.go` 中 import
- 方法未实现：`server/grpc/<service>.go` 中缺少某个 RPC 方法的实现
- 重复注册：服务名与已有服务冲突

---

## Step 6: 测试验证

启动服务后，用 curl 测试：

```bash
curl -X POST -d '{"product_id":"p1","quantity":2}' http://localhost:8080/v1/orders
```

---

## Gotchas

- `buf generate` 生成的 pb 包 import path 是 `<module>/gen/<version>`，务必与 `go.mod` 的 module 一致
- 新增服务后**不需要**在 `main.go` 中新增 import（`gen/v1` 和 `server/grpc` 包已由 Greeter 导入），只需添加注册行
- gRPC 和 HTTP 两处注册缺一不可：漏 gRPC → 504 超时，漏 HTTP → 404
- 方法实现文件必须放在 `server/grpc/` 包中，与 `GreeterServer` 同级
- `UnimplementedXxxServer` 是 protoc-gen-go-grpc 生成的兜底实现，务必嵌入 struct
