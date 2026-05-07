---
name: buf-grpc-gateway-scaffold
description: >
  Scaffold a production-ready Go gRPC project with grpc-gateway HTTP/JSON
  transcoding, using buf for proto management and code generation. Use this
  skill when the user wants to create a new gRPC service, set up
  gRPC-gateway HTTP endpoints, scaffold a Go microservice with buf, or
  mentions buf, grpc-gateway, protobuf, gRPC project setup, or dual
  gRPC+HTTP API. Supports multi-version APIs in api/v1, api/v2, etc.
compatibility: Requires Go 1.21+, buf CLI, git. Network access for buf.build.
metadata:
  author: skillshub
  version: "1.0"
---

# buf + grpc-gateway 项目脚手架

## 目录结构

```
<project>/
├── buf.yaml              # buf 模块配置（依赖、lint）
├── buf.gen.yaml          # buf 代码生成配置
├── go.mod
├── go.sum
├── .gitignore            # 含 gen/ 忽略规则
│
├── api/                  # Proto 文件
│   └── v1/               # 按版本分目录
│       ├── greeter.proto
│       └── common.proto  # 共享 message（分页、错误码等）
│
├── gen/                  # 生成代码（不提交）
│   └── v1/
│
├── server/
│   ├── grpc/             # gRPC 服务实现
│   │   └── greeter.go
│   └── gateway/          # HTTP 网关（可选）
│       └── handler.go
│
├── cmd/
│   ├── server/main.go    # 同时启动 gRPC + HTTP
│   └── client/main.go    # 测试客户端（可选）
│
├── config/
│   └── config.yaml
│
└── Makefile
```

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

## Makefile 模板

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

1. **创建目录结构** — `mkdir -p api/v1 cmd/server server/grpc gen config`
2. **编写配置文件** — 从本 skill 复制 `buf.yaml`、`buf.gen.yaml`
3. **编写 proto** — 复制模板到 `api/v1/`，替换 `<module>` 占位符
4. **编写 Go 代码** — 复制 `main.go`、`greeter.go`，替换 `<module>`
5. **init** — `make init`
6. **generate** — `make generate`
7. **run** — `make run`
8. **验证** — `curl -X POST -d '{"name":"World"}' http://localhost:8080/v1/hello`

## Gotchas

- `go_package` 选项格式为 `<module>/gen/v1;v1`，分号前是导入路径，分号后是包名。两者通常同名（都是 `v1`）。
- `buf generate` 前必须 `go mod init` 完成，否则代码生成会失败。
- 生成的代码在 `gen/` 目录，务必加入 `.gitignore`：`echo "gen/" >> .gitignore`
- BSR (buf.build) 远程插件需要网络，离线环境需改为本地插件（`local:` 类型）。
- `generate_unbound_methods=true` 会为没有 `google.api.http` 注解的方法也生成 gateway handler。
- gRPC 和 Gateway 绑定在同一个端口也是可行的（通过 `cmux`），但默认分开端口更简单可靠。

## 最佳实践

- **版本管理**：`api/` 下按版本分目录（v1, v2...），多版本共存
- **独立仓库**：如果 API 要对外暴露，考虑将 `api/` 作为独立 Git 仓库或 submodule
- **CI/CD**：PR 中运行 `buf lint` + `buf breaking` 检查规范
- **环境变量**：`GRPC_ADDR` 和 `HTTP_ADDR` 通过环境变量配置，不要硬编码
- **依赖更新**：定期 `buf mod update` 保持 proto 依赖最新
