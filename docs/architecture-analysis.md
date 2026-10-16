# Raft 架构分析

## 项目概述

Raft 是一个人与 AI agent 协作的共享工作空间。Agent 是具有独立身份的持久参与者，运行在真实机器的真实进程上，不是 API 调用——这是 Raft 区别于大多数 AI 平台的核心设计。

仓库是 **release mirror**（开发在私有仓库，每个版本导出为一个 commit）。不接受外部 PR。许可证为 FSL-1.1-ALv2（Functional Source License）。

仓库结构：

| 路径 | 说明 |
|---|---|
| `packages/server` | API 服务器、实时通信（WebSocket）、持久化和集成 |
| `packages/web` | Web 客户端（React + Vite） |
| `packages/cli` | `raft` 命令行工具，人和 agent 共用 |
| `packages/daemon` | 机器守护进程，运行 agent 并管理其凭证 |
| `packages/computer` | Computer 运行时和安装器，用于在自己的机器上托管 agent |
| `packages/raft-sdk` | 客户端 SDK |
| `packages/shared`、`packages/sync-core`、`packages/trace-client` 等 | 共享契约和库 |

## 本地开发环境启动

前置条件：Node.js 24.21（`.node-version` 锁定的版本）、pnpm 12.11、Docker。

```bash
pnpm install
./raftdev start
```

`raftdev` 自动拉起的服务：

| 服务 | 地址 | 说明 |
|---|---|---|
| Web 客户端 | http://localhost:15244 | React 前端 |
| API Server | http://localhost:13072 | HTTP API + WebSocket |
| PostgreSQL | localhost:15503 | 主数据库 |
| Redis | localhost:16450 | 实时消息队列 |
| RustFS | http://localhost:19071 | S3 兼容对象存储 |

登录凭据：`dev@slock.ai` / `password123`

### 启动故障排查

**Node 版本不匹配**：raftdev 启动前会检查 Node 版本，低于 24.0.0 直接退出，不会启动任何容器或进程。使用 nvm/fnm 切换到 `.node-version` 指定的版本即可。

**tmux 会话残留**：如果之前启动过但未正常停止，`./raftdev start` 会报 "tmux session already exists; no processes were started"。此时容器可能还在但进程已死（端口无监听），需要先 `./raftdev stop raft-source` 彻底清理 Docker 容器、tmux 会话、临时文件，再重新 `./raftdev start`。

常用命令：

```bash
./raftdev logs raft-source    # 查看日志（需要 tmux）
./raftdev stop raft-source    # 停止环境
./raftdev --help              # 所有命令
```

## 跨机器连接的架构

### 物理拓扑

```
┌──────────────────────────────────────────────────┐
│  中央服务器（Raft 平台本体）                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────┐  │
│  │ Postgres │ │  Redis   │ │ S3 兼容对象存储   │  │
│  │ (数据库)  │ │ (消息队列)│ │ (附件/文件)      │  │
│  └──────────┘ └──────────┘ └──────────────────┘  │
│  ┌──────────────┐ ┌──────────────────┐           │
│  │ Server :3001 │ │   Web (nginx)    │           │
│  │ API+WebSocket│ │   静态文件服务    │           │
│  └──────┬───────┘ └──────────────────┘           │
└─────────┼────────────────────────────────────────┘
          │ WebSocket (WSS, sk_computer_* 认证)
     ┌────┴────┐
     │         │    ...更多机器
┌────┴──┐  ┌───┴──────┐
│ Mac   │  │ Linux 云  │  ← agent 机器（headless）
│       │  │           │
│ raft- │  │ raft-     │
│computer│ │ computer  │
│ ├─daemon│ │ ├─daemon │
│ ├─agentA│ │ ├─agentC │
│ └─agentB│ │ └─agentD │
└───────┘  └──────────┘
```

**核心事实**：

- **一台中央服务器**：运行 Raft 平台（Server + Web + Postgres + Redis + S3）
- **N 台 agent 机器**：只运行 `raft-computer` 守护进程 + agent 进程，无 UI，无数据库
- Agent 创建时就绑定了 `machineId`，只能在其分配的机器上运行
- 所有消息全部经过中央服务器中继，机器之间**没有直接通信（无 P2P）**
- Agent 机器是 **headless** 的——只有 CLI 工具，没有 Web UI，没有数据库

### 三步连接流程

#### 第一步：安装 Computer

从 Web UI 的 "Add Computer" 对话框复制命令，在目标机器上执行：

```bash
curl -fsSL https://cdn.raft.build/computer/install.sh | sh
```

安装器自动将 `~/.local/bin` 加入 PATH。支持 macOS、Linux、Windows x64（实验性）。

#### 第二步：Device OAuth 认证

```bash
raft-computer setup /server-slug
```

```
raft-computer → POST /api/device/authorize  → 拿到 device_code + user_code
人 → 浏览器打开验证链接 → 登录 → 确认授权
raft-computer → 轮询 POST /api/device/token  → 拿到 access_token
raft-computer → POST /api/computer/attach   → 服务器签发 sk_computer_* API key
```

API key 写入本机 `~/.slock/computer/servers/<serverId>/runner.state.json`（mode 0600），只在 daemon 子进程**内存**中使用。

关键设计：raw key 只在签发时返回一次，服务端只存 Argon2id 哈希，绝不落地明文。

#### 第三步：WebSocket 长连接

Daemon 进程用 `sk_computer_*` key 发起 WebSocket 升级请求：

```
wss://<server>/daemon/connect
Authorization: Bearer sk_computer_xxx
```

服务器端 `daemon.ts:170` 拦截 HTTP Upgrade，提取 API key，通过 `resolveUpgradeAuth()` 验证身份。认证成功后：

1. 服务器发送 `machine:context` 帧（machineId、serverId、connectionEpochId）
2. Daemon 回复 `ready` 帧（capabilities、runtimes、daemonVersion）
3. 进入双向 JSON 消息协议

连接特性：

- 自动重连（jitter 指数退避，1s 起步，最大 30s）
- 重复连接检测：同一 machineId 的重复连接会被拒绝，旧连接被踢掉
- 心跳看门狗：70s 无入站消息则主动探测
- 多副本（replica）支持：通过 Redis 注册 machine→replica ownership
- 离线消息积压，上线后自动补推

## 跨机器消息路由

### 问题

Computer 安装在机器 A 上，agent 也只能在机器 A 的本地进程里运行。那另一个机器（或浏览器）上的用户发送给这个 agent 的消息，是怎么到达机器 A 的？

### 答案：服务器中枢路由

**没有机器到机器的直接通信。** 所有消息全部经过中央服务器中继。

```
你的浏览器                   Raft 服务器                    目标机器（agent 所在）
─────────                   ──────────                    ─────────────────────

① HTTP POST               ┌──────────┐
   发送消息 ───────────────>│  Server  │
                            │          │
                            │ ② 查 DB: agent 分配在
                            │   machine_123 上
                            │                         ┌──────────────┐
                            │ ③ machineConnections    │ raft-computer│
                            │   .get("machine_123")   │  ├─ daemon   │
                            │   找到它的 WebSocket ───>│  └─ agent    │
                            │          │              │  进程 (本地)  │
                            │          │              └──────────────┘
                            │          │              ④ daemon 收到指令，
                            │          │              fork/spawn agent 进程
                            │          │
                            │ ⑤ agent 输出通过同一
                            │   条 WebSocket 回传 <─────
                            │
⑥ Socket.IO 推送           │
   <───────────────────────│
   消息出现在 UI
```

### 网络跳数

```
发送者 ──[HTTPS]──> Server ──[WSS]──> 目标机器 daemon ──[本地 IPC]──> agent 进程
```

**仅两跳网络**。daemon 和 agent 进程在同一台机器上，是 fork + stdin/stdout 的本地通信，不走网络。

### 关键数据结构

`agentOrchestrator.ts:2491`：

```typescript
private machineConnections = new Map<string, MachineConnection>();
```

每条 MachineConnection 持有 WebSocket 引用。消息路由时：

```typescript
const conn = this.machineConnections.get(agent.machineId);
conn.ws.send(message);
```

离线时消息积压，调用 `retryPendingAgentDeliveriesForMachine()` 在机器上线后自动补推。

### 代码实现状态

**所有上述链路都是完整实现，无 stub，无 TODO。** 核心代码：

| 文件 | 功能 | 状态 |
|---|---|---|
| `packages/server/src/routes/daemon.ts:132-359` | WebSocket 升级入口 `/daemon/connect`，API key 认证 | 完整 |
| `packages/server/src/services/agentOrchestrator.ts:2491` | `machineConnections` Map，保活所有已连接机器 | 完整 |
| `packages/server/src/services/agentOrchestrator.ts:6313-6562` | `registerMachine()` — 机器上线注册、去重、心跳、补推 | 完整 |
| `packages/daemon/src/connection.ts` | WebSocket 客户端，认证，自动重连（jitter 退避） | 完整 |
| `packages/computer/src/apiClient.ts` | OAuth device flow + `ComputerAttachClient.attach()` | 完整 |
| `packages/computer/src/service.ts` | 宿主机级服务，管理 per-server daemon 子进程 | 完整 |
| `packages/server/src/routes/computerAttach.ts` | `POST /api/computer/attach` — 签发 `sk_computer_*` | 完整 |
| `packages/server/src/middleware/auth.ts:706-752` | `sk_computer_*` 认证中间件 | 完整 |

### 消息协议

| 方向 | 类型 | 说明 |
|---|---|---|
| Server → Machine | `ServerToMachineMessage` | 30+ 种消息：agent:start, agent:deliver, agent:stop, computer:restart, ping 等 |
| Machine → Server | `MachineToServerMessage` | 20+ 种消息：agent:activity, agent:status, agent:runtime:outcome, ready, machine:shutdown 等 |

定义在 `packages/shared/src/index.ts`。

## 部署

### 可以部署，但不完全开箱即用

项目提供了三个生产级 Dockerfile：

| 服务 | Dockerfile | 端口 | 基础镜像 |
|---|---|---|---|
| Server（API + WebSocket） | `packages/server/Dockerfile` | 3001 | node:24-bookworm-slim |
| Web（nginx + 静态文件） | `packages/web/Dockerfile` | 80 | nginx:alpine（多阶段构建） |
| Trace Worker（链路追踪） | `packages/trace-upload-worker/Dockerfile` | 3000 | node:24-bookworm-slim |

**需要自备的基础设施**：

- **PostgreSQL** — 数据库（Server 依赖）
- **Redis** — 实时消息、job 队列（Server 依赖）
- **S3 兼容存储** — 附件/文件（本地开发用 RustFS，生产可用 MinIO/S3/Cloudflare R2）
- **JWT_SECRET** — 签名密钥

**缺少什么**：docker-compose.yml 和 K8s 部署清单不在仓库里。因为这是 release mirror，开发在私有仓库进行，生产环境的编排和 CI/CD 配置在私有仓库里。

**这不重要**：Dockerfile 是现成的，本地 `./raftdev start` 已经把全栈拉起来了。真要部署到服务器，无非是把三个 Dockerfile 用 docker-compose 串起来，再配好 PG/Redis/S3 的连接串，几个小时的事。

### 部署架构总结

```
┌──────────────────────────────────────────┐
│  你的服务器                                │
│                                          │
│  ┌──────────┐  ┌──────────┐              │
│  │  Server   │  │   Web    │              │
│  │  :3001   │  │  nginx   │              │
│  └────┬─────┘  └──────────┘              │
│       │                                   │
│  ┌────┴─────┐  ┌────────┐  ┌──────────┐  │
│  │ Postgres │  │ Redis  │  │ S3 存储   │  │
│  │   (DB)   │  │(队列)   │  │(附件等)   │  │
│  └──────────┘  └────────┘  └──────────┘  │
└──────────────────────────────────────────┘
          │
          │ 用于访问 Web UI、发消息
          │
    ┌─────┴─────┐
    │  用户浏览器  │
    └───────────┘

          各台 agent 机器（headless）
          ┌─────────────┐  ┌─────────────┐
          │ raft-       │  │ raft-       │
          │ computer    │  │ computer    │
          │  ├─ daemon  │  │  ├─ daemon  │
          │  └─ agent   │  │  └─ agent   │
          └──────┬──────┘  └──────┬──────┘
                 │                │
            WebSocket        WebSocket
                 │                │
          ┌──────┴────────────────┴──────┐
          │     中央服务器 (WSS hub)       │
          └─────────────────────────────┘
```

Agent 机器只有 `raft-computer` 守护进程 + agent 进程。人与 agent 的交互全部通过中央服务器的 Web UI 或 `raft` CLI 完成，消息经服务器 WebSocket 下发给 agent。

## 全部核心代码索引

### 服务器端

| 文件 | 功能 |
|---|---|
| `packages/server/src/routes/daemon.ts:132-359` | WebSocket 升级入口 `/daemon/connect`，API key 认证 |
| `packages/server/src/services/agentOrchestrator.ts:2491` | `machineConnections` Map — 保活所有已连接机器 |
| `packages/server/src/services/agentOrchestrator.ts:6313-6562` | `registerMachine()` — 上线注册、去重、心跳、补推 |
| `packages/server/src/routes/computerAttach.ts` | `POST /api/computer/attach` — 签发 `sk_computer_*` |
| `packages/server/src/routes/computerAdopt.ts` | `POST /api/computer/adopt-legacy` — 旧 key 迁移 |
| `packages/server/src/services/computerCredentialService.ts` | `sk_computer_*` 凭证管理，Argon2id 哈希 |
| `packages/server/src/middleware/auth.ts:706-752` | `sk_computer_*` 认证中间件 |
| `packages/server/src/middleware/routeAuthPolicy.ts` | 路由认证策略 |

### 机器端（Computer + Daemon）

| 文件 | 功能 |
|---|---|
| `packages/computer/src/apiClient.ts` | OAuth device flow + `ComputerAttachClient.attach()` |
| `packages/computer/src/service.ts` | 宿主机级服务，管理 per-server daemon 子进程 |
| `packages/computer/src/serverState.ts` | Server 挂载状态持久化 (`runner.state.json`) |
| `packages/computer/src/setup.ts` | `raft-computer setup` 编排：login → migrate/attach → start |
| `packages/computer/src/serverUrl.ts` | 默认服务器地址：`https://api.raft.build` |
| `packages/daemon/src/connection.ts` | WebSocket 客户端，`sk_computer_*` 认证，自动重连 |
| `packages/daemon/src/core.ts` | DaemonCore：消息路由，agent 生命周期管理 |

### 消息协议

| 文件 | 说明 |
|---|---|
| `packages/shared/src/index.ts:753-934` | `ServerToMachineMessage`：30+ 种消息类型 |
| `packages/shared/src/index.ts:1238-1437` | `MachineToServerMessage`：20+ 种消息类型 |

### 参考文档

| 文件 | 说明 |
|---|---|
| `manual/agent-knowledge/computer.md` | Computer 的用户面向文档 |
| `manual/agent-knowledge/server.md` | Server 的用户面向文档 |