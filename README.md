# skillshub

Agent Skills 集合，按 agentskills.io 规范编写。

## 现有 Skills

| 技能 | 粒度 | 说明 |
|------|------|------|
| `buf-grpc-gateway-scaffold` | 项目级 | buf + grpc-gateway 项目从零搭建 |
| `buf-add-service` | 中 | 同进程内新增 gRPC 服务（共用 go.mod/main.go） |
| `buf-add-rpc` | 小 | 给已有服务追加 RPC 方法 |
| `buf-add-micro` | 中 | 新建独立微服务（独立 go.mod/main.go/端口） |
| `buf-maintain` | 综合 | 日常维护（lint / breaking check / generate / mod update） |

## 规范参考

`reference/` 为 [agentskills.io](https://agentskills.io) 规范子模块，包含完整的 Skill 格式定义、最佳实践和参考实现。
