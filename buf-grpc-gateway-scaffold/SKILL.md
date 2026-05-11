---
name: buf-grpc-gateway-scaffold
description: >
  Scaffold a standalone API Gateway project using buf + grpc-gateway. This
  gateway is the SINGLE HTTP entry point — all backend microservices are
  pure gRPC (no HTTP) and connected via go.mod replace + gRPC client proxy.
  Use this skill when creating a new Gateway project, setting up the unified
  API entry point, or bootstrapping a project that will proxy to multiple
  backend gRPC services. For backend services, use buf-add-grpc-svc instead.
compatibility: Requires Go 1.21+, buf CLI, git. Network access for buf.build.
metadata:
  author: skillshub
  version: "2.0"
---

# buf + grpc-gateway 项目脚手架

## 目录结构

Gateway 项目作为顶层入口，后端 gRPC 服务作为同级子目录（monorepo）：

```
root/
├── gateway/              # ⭐ API Gateway（本 skill 创建的唯一网关）
│   ├── buf.yaml
│   ├── buf.gen.yaml
│   ├── go.mod
│   ├── go.sum
│   ├── .gitignore
│   │
│   ├── api/              # 网关自己的 proto（含 HTTP 注解）
│   │   └── v1/
│   │       ├── greeter.proto
│   │       └── common.proto
│   │
│   ├── gen/              # 生成代码
│   │   └── v1/
│   │
│   ├── server/
│   │   └── grpc/
│   │       └── greeter.go
│   │
│   ├── cmd/
│   │   └── server/main.go   # gRPC + HTTP 网关 + 外部服务代理
│   │
│   ├── config/
│   │   └── config.yaml
│   │
│   └── Makefile
│
├── auth-service/         # 纯 gRPC 后端服务（无 HTTP，由 buf-add-grpc-svc 创建）
│   └── ...
│
├── order-service/        # 纯 gRPC 后端服务
│   └── ...
│
└── payment-service/      # 纯 gRPC 后端服务
    └── ...
```

> gateway 是**唯一**的 HTTP 入口，所有外部请求经它路由到后端 gRPC 服务。
> 后端服务不暴露 HTTP，认证/限流/鉴权统一在 gateway 处理。

## buf.yaml 模板

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

## buf.gen.yaml 模板

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

## Proto 模板（api/v1/greeter.proto）

```protobuf
syntax = "proto3";

package api.v1;

option go_package = "<module>/gen/v1;v1";

import "google/api/annotations.proto";

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply) {
    option (google.api.http) = {
      post: "/v1/hello"
      body: "*"
    };
  }
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}
```

> 替换 `<module>` 为 go.mod 中的 module 名。

## 共享类型模板（api/v1/common.proto）

```protobuf
syntax = "proto3";

package api.v1;

option go_package = "<module>/gen/v1;v1";

message PageRequest {
  int32 page = 1;
  int32 page_size = 2;
}

message PageResponse {
  int32 total = 1;
  int32 page = 2;
  int32 page_size = 3;
}
```

## cmd/server/main.go 模板

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

    pb "<module>/gen/v1"
    "<module>/server/grpc"
)

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    grpcAddr := envOrDefault("GRPC_ADDR", ":9090")
    httpAddr := envOrDefault("HTTP_ADDR", ":8080")

    // 1. gRPC server
    lis, err := net.Listen("tcp", grpcAddr)
    if err != nil {
        panic(fmt.Errorf("failed to listen on %s: %w", grpcAddr, err))
    }

    grpcServer := grpc.NewServer()
    pb.RegisterGreeterServer(grpcServer, &grpc.GreeterServer{})

    go func() {
        fmt.Printf("gRPC server listening on %s\n", grpcAddr)
        if err := grpcServer.Serve(lis); err != nil {
            panic(err)
        }
    }()

    // 2. HTTP Gateway
    conn, err := grpc.DialContext(ctx, grpcAddr,
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithBlock(),
    )
    if err != nil {
        panic(fmt.Errorf("failed to dial gRPC: %w", err))
    }

    mux := runtime.NewServeMux()
    if err := pb.RegisterGreeterHandler(ctx, mux, conn); err != nil {
        panic(err)
    }

    fmt.Printf("HTTP gateway listening on %s\n", httpAddr)

    go func() {
        if err := http.ListenAndServe(httpAddr, mux); err != nil && err != http.ErrServerClosed {
            panic(err)
        }
    }()

    // 3. Graceful shutdown
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

> 替换 `<module>` 为 go.mod 中的 module 名。此版本含优雅退出，优于原始版本。

## server/grpc/greeter.go 模板

```go
package grpc

import (
    "context"
    "fmt"

    pb "<module>/gen/v1"
)

type GreeterServer struct {
    pb.UnimplementedGreeterServer
}

func (s *GreeterServer) SayHello(ctx context.Context, req *pb.HelloRequest) (*pb.HelloReply, error) {
    return &pb.HelloReply{
        Message: fmt.Sprintf("Hello, %s", req.Name),
    }, nil
}
```

## Makefile

```makefile
.PHONY: init generate clean build run dev test lint deps

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

test:
	go test ./...

lint:
	buf lint
	buf breaking --against '.git#branch=main'

# 同步外部服务依赖（需后端服务先 go mod tidy）
deps:
	go mod tidy
```

## 接入外部 gRPC 服务

Gateway 作为唯一入口，需要代理后端纯 gRPC 服务的 HTTP 请求。

### Step 1: go.mod 添加 replace 指令

后端服务在 monorepo 同级目录下，用 `replace` 指向本地路径：

```
module gateway

go 1.21

require (
    github.com/grpc-ecosystem/grpc-gateway/v2 v2.25.0
    google.golang.org/grpc v1.68.0
    auth-service v0.0.0
    order-service v0.0.0
)

replace (
    auth-service => ../auth-service
    order-service => ../order-service
)
```

`require` 中的 `auth-service v0.0.0` 是占位，实际版本由 `replace` 指向的本地路径决定。

### Step 2: main.go 注册外部服务代理

```go
package main

import (
    // ...

    pb        "gateway/gen/v1"
    "gateway/server/grpc"

    // === 外部 gRPC 服务 ===
    authpb    "auth-service/gen/v1"
    orderpb   "order-service/gen/v1"
)

func main() {
    // ... gRPC server 启动 ...

    // 1. 本地服务（同进程注册）
    pb.RegisterGreeterServer(grpcServer, &grpc.GreeterServer{})

    // 2. HTTP Gateway：本地服务 handler
    mux := runtime.NewServeMux()
    pb.RegisterGreeterHandler(ctx, mux, conn)              // 本地 gRPC

    // === 3. 外部 gRPC 服务代理 ===

    // auth-service
    authConn, err := grpc.DialContext(ctx, "localhost:9091",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithBlock(),
    )
    if err != nil {
        panic(fmt.Errorf("failed to dial auth-service: %w", err))
    }
    defer authConn.Close()
    if err := authpb.RegisterAuthHandler(ctx, mux, authConn); err != nil {
        panic(err)
    }

    // order-service
    orderConn, err := grpc.DialContext(ctx, "localhost:9092",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithBlock(),
    )
    if err != nil {
        panic(fmt.Errorf("failed to dial order-service: %w", err))
    }
    defer orderConn.Close()
    if err := orderpb.RegisterOrderHandler(ctx, mux, orderConn); err != nil {
        panic(err)
    }

    // HTTP server 启动...
}
```

**关键点：**
- 每个外部服务一个独立的 `grpc.DialContext` 连接（加 `defer close`）
- `Register<Service>Handler` 将 HTTP 请求转为 gRPC 调用并转发到目标服务
- 连接通过 `replace` 指令找到本地 proto 包，实际网络调用走 `localhost:<port>`

### Step 3: 端口规划

| 服务 | gRPC 端口 | HTTP 端口 | 说明 |
|------|-----------|-----------|------|
| gateway | :9090 | :8080 | 唯一 HTTP 入口 |
| auth-service | :9091 | — | 纯 gRPC |
| order-service | :9092 | — | 纯 gRPC |
| ... | :909N | — | — |

### Step 4: 认证中间件（示例）

在 gateway 的 HTTP mux 上包装认证：

```go
authInterceptor := func(h http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, `{"error":"unauthorized"}`, http.StatusUnauthorized)
            return
        }
        // 验证 token，注入 context...
        h.ServeHTTP(w, r)
    })
}

// 对特定路径加认证保护
protectedMux := http.NewServeMux()
protectedMux.Handle("/v1/orders/", authInterceptor(http.HandlerFunc(mux.ServeHTTP)))
protectedMux.Handle("/v1/hello", mux)  // 公开接口

http.ListenAndServe(httpAddr, protectedMux)
```

---

## Makefile 模板

网关 Makefile，多了连接外部服务的依赖管理：

```makefile
.PHONY: init generate clean build run dev test lint

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

test:
	go test ./...

lint:
	buf lint
	buf breaking --against '.git#branch=main'

# 新增 proto 后的完整流程
all: lint generate build test
```

## 使用流程

按顺序执行：

1. **创建目录结构** — `mkdir -p gateway/api/v1 gateway/cmd/server gateway/server/grpc gateway/gen gateway/config`
2. **编写配置文件** — 复制 `gateway/buf.yaml`、`gateway/buf.gen.yaml`
3. **编写 proto** — 复制模板到 `gateway/api/v1/`，替换 `<module>` 占位符
4. **编写 Go 代码** — 复制 `main.go`、`greeter.go`，替换 `<module>`
5. **init** — `cd gateway && make init`
6. **generate** — `make generate`
7. **run** — `make run`
8. **验证** — `curl http://localhost:8080/v1/hello`

后续接入外部服务：
9. **创建后端服务** — 使用 `buf-add-grpc-svc` 创建 `auth-service`、`order-service` 等
10. **go.mod 添加 replace** — 在 gateway 的 go.mod 中 `replace auth-service => ../auth-service`
11. **main.go 添加代理** — 参考上方"接入外部 gRPC 服务"章节
12. **重建验证** — `make deps && make build && make run`

## Gotchas

- `go_package` 选项格式为 `<module>/gen/v1;v1`，分号前是导入路径，分号后是包名。两者通常同名（都是 `v1`）。
- `buf generate` 前必须 `go mod init` 完成，否则代码生成会失败。
- 生成的代码在 `gen/` 目录，务必加入 `.gitignore`：`echo "gen/" >> .gitignore`
- BSR (buf.build) 远程插件需要网络，离线环境需改为本地插件（`local:` 类型）。
- `generate_unbound_methods=true` 会为没有 `google.api.http` 注解的方法也生成 gateway handler。
- gRPC 和 Gateway 绑定在同一个端口也是可行的（通过 `cmux`），但默认分开端口更简单可靠。

### 外部服务代理相关

- `go.mod` 的 `replace` 指令中，`require` 行必须存在才能 `replace` 成功。使用 `go get auth-service@latest` 或手动写入 `require auth-service v0.0.0`
- 外部服务的 proto 中定义的 HTTP 路径会注册到 gateway 的 mux 上，注意路径冲突
- 新增外部服务后需要 `go mod tidy` 更新 go.sum
- 每个外部 gRPC 连接独立管理，不要共享一个 conn 给多个 Handler
- `grpc.WithBlock()` 确保启动时连接失败会 panic 而非静默降级
- 生产环境用 TLS 替代 `insecure.NewCredentials()`

## 最佳实践

- **Gatewy 为唯一入口**：不要在后端服务上挂 HTTP gateway，认证/限流/鉴权统一在 gateway 层处理
- **版本管理**：`api/` 下按版本分目录（v1, v2...），多版本共存
- **独立仓库**：如果 API 要对外暴露，考虑将 `gateway/` 作为独立 Git 仓库
- **CI/CD**：PR 中运行 `buf lint` + `buf breaking` 检查规范
- **环境变量**：`GRPC_ADDR` 和 `HTTP_ADDR` 通过环境变量配置，不要硬编码
- **后端服务地址**：也通过环境变量注入（`AUTH_SVC_ADDR`），不要硬编码 `localhost:9091`
- **依赖更新**：定期 `buf mod update` 保持 proto 依赖最新
