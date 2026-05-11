---
name: buf-add-micro
description: >
  Create a new independent Go microservice project using buf + grpc-gateway
  as a subdirectory within an existing monorepo. Generates its own go.mod,
  buf config, proto files, server implementation, and main.go. Use this
  skill when the user wants to create a standalone deployable service, add
  a new microservice to a monorepo, or explicitly says "独立服务" / "新建微服务"
  / "单独部署" in a Go gRPC context. Not for adding a service to an existing
  process — use buf-add-service for that.
compatibility: Requires Go 1.21+, buf CLI, existing project root with buf-grpc-gateway-scaffold skill available for reference.
metadata:
  author: skillshub
  version: "1.0"
---

# 新建独立微服务

在现有 monorepo 中创建一个完全独立的 gRPC 微服务项目（独立 go.mod、main.go、部署单元）。

与 `buf-add-service` 的区别：

| | buf-add-service | buf-add-micro |
|---|---|---|
| go.mod | 共用 | 独立 |
| main.go | 追加注册 | 全新文件 |
| 部署 | 一个二进制多 service | 一个 service 一个二进制 |
| 接口 | 走同一个端口 | 独立端口 |
| 目录 | 放在现有 api/server/ 下 | 独立子目录 |

## 检查清单

- [ ] Step 1: 创建服务目录骨架
- [ ] Step 2: 编写 buf 配置
- [ ] Step 3: 编写 proto 文件
- [ ] Step 4: 生成代码
- [ ] Step 5: 实现服务
- [ ] Step 6: 编写 main.go
- [ ] Step 7: 编写 Makefile
- [ ] Step 8: 编译验证
- [ ] Step 9: 启动测试

---

## Step 1: 创建目录骨架

在项目根目录下创建服务目录。用户需提供服务名（kebab-case，如 `order-service`）。

```
root/
├── order-service/           # 新建
│   ├── api/v1/              # proto 文件
│   ├── gen/                 # 生成代码（.gitignore）
│   ├── server/
│   │   └── grpc/
│   ├── cmd/
│   │   └── server/
│   └── config/
├── user-service/            # 已有服务
│   └── ...
└── ...
```

命令：

```bash
mkdir -p <service>/api/v1 \
         <service>/gen \
         <service>/server/grpc \
         <service>/cmd/server \
         <service>/config
echo "gen/" > <service>/.gitignore
```

---

## Step 2: buf 配置

`<service>/buf.yaml`：

```yaml
version: v2
modules:
  - path: api
lint:
  use:
    - DEFAULT
  except:
    - PACKAGE_VERSION_SUFFIX
breaking:
  use:
    - FILE
deps:
  - buf.build/googleapis/googleapis
```

`<service>/buf.gen.yaml`：

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

## Step 3: Proto 文件

根据用户提供的 RPC 定义编写 `<service>/api/v1/<name>.proto`。

**命名规则：**
- 目录名：kebab-case（如 `order`）
- proto 文件名：snake_case（如 `order.proto`）
- go_package：`<service>/gen/v1;v1`

示例 `<service>/api/v1/order.proto`：

```protobuf
syntax = "proto3";

package api.v1;

option go_package = "order-service/gen/v1;v1";

import "google/api/annotations.proto";

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
}
```

---

## Step 4: 生成代码

```bash
cd <service>
go mod init <service>
buf mod update
buf generate
```

验证：

```bash
ls gen/v1/<name>.pb.go
ls gen/v1/<name>_grpc.pb.go
ls gen/v1/<name>.pb.gw.go
```

---

## Step 5: 实现服务

`<service>/server/grpc/<name>.go`：

```go
package grpc

import (
    "context"
    "fmt"

    pb "<service>/gen/v1"
)

type <ServiceName>Server struct {
    pb.Unimplemented<ServiceName>Server
}

func (s *<ServiceName>Server) <MethodName>(ctx context.Context, req *pb.<RequestType>) (*pb.<ResponseType>, error) {
    return &pb.<ResponseType>{
        // TODO: 实现业务逻辑
    }, nil
}
```

为每个 RPC 方法添加对应的实现函数。

---

## Step 6: 编写 main.go

`<service>/cmd/server/main.go`：

```go
package main

import (
    "context"
    "fmt"
    "net"
    "net/http"
    "os"
    "os/signal"
    "syscall"

    "github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"

    pb "<service>/gen/v1"
    "<service>/server/grpc"
)

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    grpcAddr := envOrDefault("GRPC_ADDR", ":9090")
    httpAddr := envOrDefault("HTTP_ADDR", ":8080")

    // gRPC server
    lis, err := net.Listen("tcp", grpcAddr)
    if err != nil {
        panic(fmt.Errorf("failed to listen on %s: %w", grpcAddr, err))
    }

    grpcServer := grpc.NewServer()
    pb.Register<ServiceName>Server(grpcServer, &grpc.<ServiceName>Server{})

    go func() {
        fmt.Printf("gRPC server listening on %s\n", grpcAddr)
        if err := grpcServer.Serve(lis); err != nil {
            panic(err)
        }
    }()

    // HTTP gateway
    conn, err := grpc.DialContext(ctx, grpcAddr,
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithBlock(),
    )
    if err != nil {
        panic(fmt.Errorf("failed to dial gRPC: %w", err))
    }

    mux := runtime.NewServeMux()
    if err := pb.Register<ServiceName>Handler(ctx, mux, conn); err != nil {
        panic(err)
    }

    fmt.Printf("HTTP gateway listening on %s\n", httpAddr)

    go func() {
        if err := http.ListenAndServe(httpAddr, mux); err != nil && err != http.ErrServerClosed {
            panic(err)
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    fmt.Println("Shutting down...")
    grpcServer.GracefulStop()
    cancel()
}

func envOrDefault(key, defaultVal string) string {
    if v := os.Getenv(key); v != "" {
        return v
    }
    return defaultVal
}
```

**占位符替换：**
- `<service>` — go module 路径，与 `go mod init` 一致
- `<ServiceName>` — 首字母大写服务名（如 `Order`）

---

## Step 7: Makefile

`<service>/Makefile`：

```makefile
.PHONY: init generate clean build run dev

init:
	go mod tidy
	buf mod update

generate:
	buf generate

clean:
	rm -rf gen/

build: generate
	go build -o bin/server ./cmd/server

run:
	go run ./cmd/server

dev: clean generate build run
```

---

## Step 8: 编译验证

```bash
cd <service>
go build ./cmd/server
```

常见错误：
- `go.mod` 未 init → 回到 Step 4
- 依赖未下载 → `go mod tidy`
- import 路径不一致 → 检查所有文件中的 `<service>` 替换

---

## Step 9: 启动测试

```bash
cd <service>
GRPC_ADDR=:9091 HTTP_ADDR=:8081 make run
```

> 注意使用不同于已有服务的端口，避免冲突。

```bash
curl -X POST -d '{"product_id":"p1","quantity":2}' http://localhost:8081/v1/orders
```

---

## 多服务端口规划建议

| 服务 | gRPC | HTTP |
|------|------|------|
| user-service | :9091 | :8081 |
| order-service | :9092 | :8082 |
| payment-service | :9093 | :8083 |
| ... | :909N | :808N |

或通过环境变量注入，在 docker-compose/k8s 中管理。

---

## Gotchas

- 每个微服务有独立的 `go.mod`，依赖版本独立管理
- 跨服务调用需走 gRPC client，不是直接 import 另一个服务的 gen 包
- `go_package` 路径以 service 目录名为前缀（`order-service/gen/v1;v1`），与 monolith 不同
- 端口规划要提前约定，避免冲突。推荐用环境变量而非硬编码
- 如需共享 message 类型（如 PageRequest），可在顶层建 `shared/` proto 包或 copy 定义（前者更优但增加耦合）
