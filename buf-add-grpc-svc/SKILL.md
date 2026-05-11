---
name: buf-add-grpc-svc
description: >
  Create a pure gRPC backend service (no HTTP gateway) that runs behind a
  unified API Gateway. Generates independent go.mod, proto files without
  HTTP annotations, gRPC server only — no grpc-gateway dependencies. Use
  this skill when the user wants to add a backend microservice behind a
  gateway, create a service that only needs gRPC, or mentions "纯 gRPC" /
  "后端服务" / "不暴露 HTTP" in a monorepo with a gateway project. NOT for
  services that need HTTP exposure — use buf-add-micro for that.
compatibility: Requires Go 1.21+, buf CLI, existing gateway project created with buf-grpc-gateway-scaffold.
metadata:
  author: skillshub
  version: "1.0"
---

# 新建纯 gRPC 后端服务

创建挂载在 API Gateway 后面的纯 gRPC 后端服务（无 HTTP 端口，不依赖 grpc-gateway）。

## 与相关技能的区别

| | buf-add-service | buf-add-micro | buf-add-grpc-svc |
|---|---|---|---|
| HTTP Gateway | 注册到已有 gateway | 自建一套 | ❌ 无 |
| HTTP 端口 | 共用 | :808N | ❌ 无 |
| go.mod | 共用 | 独立 | 独立 |
| gRPC 端口 | 共用 | :909N | :909N |
| 认证 | 网关统一 | 需自配 | 网关统一 |
| 部署 | 一个二进制 | 一个二进制 | 一个二进制 |
| 依赖 grpc-gateway | ✅ | ✅ | ❌ |
| proto 含 HTTP 注解 | ✅ | ✅ | ❌ |

---

## 检查清单

- [ ] Step 1: 创建服务目录骨架
- [ ] Step 2: 编写 proto（无 HTTP 注解）
- [ ] Step 3: 配置 buf 并生成代码
- [ ] Step 4: 实现 gRPC 服务
- [ ] Step 5: 编写 main.go（仅 gRPC server）
- [ ] Step 6: 编写 Makefile
- [ ] Step 7: 编译验证
- [ ] Step 8: 接入 Gateway（参考 buf-grpc-gateway-scaffold）

---

## Step 1: 创建目录骨架

在 gateway 项目同级创建：

```bash
mkdir -p <service>/api/v1 \
         <service>/gen \
         <service>/server \
         <service>/cmd/server \
         <service>/config
echo "gen/" > <service>/.gitignore
```

```
root/
├── gateway/          # API Gateway（已有）
├── auth-service/     # ← 新建
│   ├── api/v1/
│   ├── gen/
│   ├── server/
│   ├── cmd/server/
│   └── config/
├── order-service/    # 后续新建
└── ...
```

---

## Step 2: Proto 文件

**关键区别**：不需要 `import "google/api/annotations.proto"`，不需要 HTTP 注解。

`<service>/api/v1/auth.proto`：

```protobuf
syntax = "proto3";

package api.v1;

option go_package = "auth-service/gen/v1;v1";

service Auth {
  rpc Login (LoginRequest) returns (LoginResponse);
  rpc ValidateToken (ValidateTokenRequest) returns (ValidateTokenResponse);
  rpc RefreshToken (RefreshTokenRequest) returns (RefreshTokenResponse);
}

message LoginRequest {
  string username = 1;
  string password = 2;
}

message LoginResponse {
  string access_token = 1;
  string refresh_token = 2;
  int64 expires_at = 3;
}

message ValidateTokenRequest {
  string access_token = 1;
}

message ValidateTokenResponse {
  bool valid = 1;
  string user_id = 2;
  string username = 3;
}

message RefreshTokenRequest {
  string refresh_token = 1;
}

message RefreshTokenResponse {
  string access_token = 1;
  int64 expires_at = 2;
}
```

**命名规则：**
- 目录名：kebab-case（`auth-service`）
- proto 文件名：snake_case（`auth.proto`）
- go_package：`<service>/gen/v1;v1`
- service 名：PascalCase（`Auth`）

---

## Step 3: buf 配置 + 代码生成

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
# 不需要 deps 了，因为没有 google/api/annotations.proto
```

> 纯 gRPC 服务不需要依赖 `buf.build/googleapis/googleapis`。

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
  # 不需要 grpc-gateway 插件
```

生成代码：

```bash
cd <service>
go mod init <service>
buf mod update
buf generate
go mod tidy
```

验证：

```bash
ls gen/v1/auth.pb.go          # protobuf 消息
ls gen/v1/auth_grpc.pb.go     # gRPC 接口
# 无 auth.pb.gw.go — 不生成 gateway 代码
```

---

## Step 4: 实现 gRPC 服务

`<service>/server/auth.go`：

```go
package server

import (
    "context"
    "fmt"

    pb "<service>/gen/v1"
)

type AuthServer struct {
    pb.UnimplementedAuthServer
}

func (s *AuthServer) Login(ctx context.Context, req *pb.LoginRequest) (*pb.LoginResponse, error) {
    // TODO: 验证用户名密码
    return &pb.LoginResponse{
        AccessToken:  fmt.Sprintf("token-%s-%d", req.Username, 3600),
        RefreshToken: fmt.Sprintf("refresh-%s", req.Username),
        ExpiresAt:    3600,
    }, nil
}

func (s *AuthServer) ValidateToken(ctx context.Context, req *pb.ValidateTokenRequest) (*pb.ValidateTokenResponse, error) {
    // TODO: 验证 token
    return &pb.ValidateTokenResponse{
        Valid:    true,
        UserId:   "user-1",
        Username: "admin",
    }, nil
}

func (s *AuthServer) RefreshToken(ctx context.Context, req *pb.RefreshTokenRequest) (*pb.RefreshTokenResponse, error) {
    // TODO: 刷新 token
    return &pb.RefreshTokenResponse{
        AccessToken: "new-token",
        ExpiresAt:   7200,
    }, nil
}
```

---

## Step 5: main.go（仅 gRPC）

`<service>/cmd/server/main.go`：

```go
package main

import (
    "fmt"
    "net"
    "os"
    "os/signal"
    "syscall"

    "google.golang.org/grpc"

    pb "<service>/gen/v1"
    "<service>/server"
)

func main() {
    addr := envOrDefault("GRPC_ADDR", ":9091")

    lis, err := net.Listen("tcp", addr)
    if err != nil {
        panic(fmt.Errorf("failed to listen on %s: %w", addr, err))
    }

    grpcServer := grpc.NewServer()
    pb.RegisterAuthServer(grpcServer, &server.AuthServer{})

    go func() {
        fmt.Printf("gRPC server listening on %s\n", addr)
        if err := grpcServer.Serve(lis); err != nil {
            panic(err)
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    fmt.Println("Shutting down...")
    grpcServer.GracefulStop()
}

func envOrDefault(key, defaultVal string) string {
    if v := os.Getenv(key); v != "" {
        return v
    }
    return defaultVal
}
```

**与 gateway 的 main.go 对比：**
- ❌ 无 `net/http`、`grpc-gateway/v2/runtime` import
- ❌ 无 HTTP server、mux、handler 注册
- ❌ 无 `grpc.DialContext`（不连其他服务）
- ✅ 只有 gRPC server + signal handling

---

## Step 6: Makefile

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

## Step 7: 编译验证

```bash
cd <service>
make build
```

---

## Step 8: 接入 Gateway

回到 gateway 项目：

```bash
cd ../gateway
```

**a) go.mod 添加依赖：**

```bash
go get auth-service@latest
```

或手动写入 `go.mod`：

```
require (
    auth-service v0.0.0
)

replace auth-service => ../auth-service
```

**b) main.go 注册代理：**

```go
import (
    authpb "auth-service/gen/v1"
)

func main() {
    // ... 已有 gRPC server + mux ...

    // 连接到 auth-service
    authConn, err := grpc.DialContext(ctx,
        os.Getenv("AUTH_SVC_ADDR"),  // 如 "localhost:9091"
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithBlock(),
    )
    if err != nil {
        panic(fmt.Errorf("failed to dial auth-service: %w", err))
    }
    defer authConn.Close()

    // 注册到 HTTP mux
    if err := authpb.RegisterAuthHandler(ctx, mux, authConn); err != nil {
        panic(err)
    }
}
```

**c) 重建 gateway：**

```bash
go mod tidy
make build
make run
```

---

## 端口管理

| 服务 | gRPC 端口 | HTTP |
|------|-----------|------|
| gateway | :9090 | :8080 |
| auth-service | :9091 | — |
| order-service | :9092 | — |
| ... | :909N | — |

---

## Gotchas

- 纯 gRPC 服务的 proto **不能** import `google/api/annotations.proto`，也不要有 `option (google.api.http) = {...}`
- 不需要 `buf.yaml` 的 `deps`，也不用 `buf.gen.yaml` 的 gateway 插件
- `go mod init` 的 module 名要与 `go_package` 前缀一致（`auth-service` → `go_package = "auth-service/gen/v1;v1"`）
- gateway 中 `go.mod` 的 `require auth-service v0.0.0` 是占位版本，实际版本由 `replace` 决定
- 后端服务地址通过环境变量注入（`AUTH_SVC_ADDR`），不要硬编码
- 多个后端服务用不同端口启动，不可冲突
- 如果需要跨服务调用（如 order 发请求到 auth），在后端服务内部用 gRPC client 直连，不经过 gateway
