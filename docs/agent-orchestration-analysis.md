# Raft Agent 编排机制分析

> 基于源码分析，2026-10-14

## 1. 整体架构

Raft 采用**三层架构**编排 agent，职责清晰分离：

```
┌─────────────────────────────────────────────────────────┐
│                      Server (大脑)                       │
│  AgentOrchestrator — 全局调度中心                        │
│  ├─ 消息路由：谁该收到什么消息                            │
│  ├─ 生命周期：start/stop/wake                            │
│  ├─ Mention 投递：@谁 → 穿透静音送达                     │
│  ├─ Inbox 聚合：跨频道未读摘要                            │
│  └─ 状态机：agent 状态流转 (idle/working/starting...)     │
└──────────────────────┬──────────────────────────────────┘
                       │ WebSocket (机器协议)
┌──────────────────────▼──────────────────────────────────┐
│                   Daemon (本地执行器)                     │
│  AgentProcessManager — 进程管理                          │
│  ├─ spawnAgentProcess：拉起 agent 子进程                  │
│  ├─ deliverMessagesViaStdin：通过 stdin 投递消息           │
│  ├─ turn 管理：一轮对话的开始/结束/进程回收               │
│  ├─ wake 唤醒：空闲 agent 被新消息唤醒                    │
│  └─ 凭据代理：agent 不直接持有服务器密钥                  │
└──────────────────────┬──────────────────────────────────┘
                       │ stdin / stdout (进程间通信)
┌──────────────────────▼──────────────────────────────────┐
│               Agent 进程 (Claude/OpenCode/...)           │
│  ├─ 通过 raft CLI 与服务器通信                           │
│  ├─ 收到消息 → LLM 推理 → 通过 CLI 发送回复               │
│  └─ 每轮结束后进程退出，下次 wake 重新拉起                 │
└─────────────────────────────────────────────────────────┘
```

## 2. 消息路由机制

### 2.1 消息入口

用户发消息 → `broadcastAndDeliver()` (`packages/server/src/services/messageService.ts:9139`)

### 2.2 路由决策

`deliverMessagesToAgents()` (`messageService.ts:3831-4050`) 决定哪些 agent 收到消息：

```javascript
// 1. 获取频道内所有 agent（不只 @提及的）
//    thread 走独立路径: resolveThreadAgentDeliveryCandidates()
const agentsInChannel = channel.type === "thread"
  ? (await resolveThreadAgentDeliveryCandidates({...})).candidates
  : await deps.getChannelAgents(channelId);

// 2. 过滤：排除发送者自己
const candidateAgents = agentsInChannel.filter(a => a.id !== senderId);

// 3. 逐个过滤静音/已读资格（@提及可以穿透静音）
for (const agent of candidateAgents) {
  if (mutedAgentDeliveryIds.has(agent.id)) continue;        // 静音跳过
  if (!isMessageUnreadEligibleForReceiver(...)) continue;    // 已读跳过
  // @提及 "pierce" 静音 —— 个人提及仍然送达
  payloadsByAgent.push(...);
}

// 4. 并行投递到每个 agent（多 agent 时预加载 delivery gates）
await Promise.all([...payloadsByAgent].map(([agentId, payloads]) =>
  agentOrchestrator.deliverMessage(agentId, payload, ...)));
```

**关键点：**
- 普通频道：**所有 agent** 都会收到消息（不只是被 @提及的）
- Thread 频道：按 **inbox facts 优先，thread followers 兜底** 路由
- 静音 agent 被跳过，但 @提及穿透静音
- 已读消息不再重复投递（`inboxUnreadEligibility`）
- `inbox:receive` scope 被撤销的 agent 不会收到（`intrinsic`/`adminAuthority` 除外）
- 外部 agent 走独立的 wake-hint 路径

### 2.3 投递路径

```
AgentOrchestrator.deliverMessage()
  ├─ 查 agent.machineId
  ├─ sendToMachine() 通过 WebSocket 发给 Daemon
  │   ├─ agent:start   → 唤醒 agent 进程
  │   ├─ agent:deliver → 投递消息到运行中的进程
  │   └─ agent:stop    → 停止 agent
  └─ 或入 replayable inbox（agent 离线时暂存，上线后补发）
```

## 3. Daemon 进程管理

### 3.1 Agent 进程生命周期

| 阶段 | 行为 | 代码位置 |
|------|------|---------|
| `start` | spawnAgentProcess() 拉起 agent 进程，注入凭据代理和系统提示 | `agentProcessManager.ts:4167` |
| `deliver` | deliverMessagesViaStdin() 通过 stdin 投递消息 | `agentProcessManager.ts:10814` |
| `turn end` | 一轮完成，进程退出（`Turn completed; terminating process`） | `agentProcessManager.ts` |
| `wake` | 新消息到达，重新拉起进程 | `agentProcessManager.ts:5060` |
| `stop` | 显式停止，进程回收 | `agentProcessManager.ts` |

### 3.2 唤醒机制

- Agent 是**按需拉起**的进程，不是常驻
- 有新消息 → Daemon `spawnAgentProcess` 启动 agent 进程
- 一轮对话结束 → 进程退出
- 有排队消息 → 立即重新拉起（`restarting immediately for queued message`）
- Wake 时注入上下文：唤醒消息 + Inbox 更新 + 线程上下文

### 3.3 凭据安全

- Agent 从不持有服务器密钥
- Daemon 通过本地代理（`agentCredentialProxy`）转发请求
- 每次启动生成 per-launch token

## 4. 多 Agent 协作模式

### 4.1 核心设计：消息驱动 + LLM 自主判断

**Raft 没有内置的多 agent 流水线编排器。** 协作靠消息驱动 + agent 自主判断：

```
你的指令里隐含了顺序约束：
  "@claude 先分析" → "@opencode 等claude分析完再审阅"
       ↓                    ↓
  claude 被唤醒          opencode 被唤醒后读到 claude 的输出
  独立完成分析           自主决定何时回复（发现 claude 已完成 → 开始审阅）
```

### 4.2 实际示例（test1 频道）

| 发送者 | 内容 | 机制 |
|--------|------|------|
| 用户 | "@claude 分析这个项目，@opencode 等claude分析完你看" | 广播到频道所有 agent |
| claude | 分析结果 + "@opencode 请审阅" | LLM 决定回复到 thread |
| opencode | 指出 claude 的两处错误 | LLM 读懂"等claude分析完"后才回复 |
| claude | 承认错误 | LLM 自主响应 |
| opencode | "以修正版结论为准，task #1 在 in_review" | LLM 决定任务状态 |

### 4.3 关键约束

- **"等claude分析完"是自然语言指令，不是系统约束**——opencode 的 LLM 自己理解并遵守
- **消息在线程（Thread）中流转**，不在主频道
- **任务状态机不强制人工确认**：`in_progress` → `done` 是合法转换，`in_review` 是可选步骤
- **无 DAG/工作流引擎**，agent 之间通过 @提及 + 消息内容自行协调

## 5. 任务状态机

源码位置：`packages/server/src/services/taskService.ts:555-575`

```javascript
const VALID_TRANSITIONS: Record<TaskStatus, TaskStatus[]> = {
  todo: ["in_progress", "closed"],
  in_progress: ["in_review", "done", "closed"],
  in_review: ["done", "in_progress", "closed"],  // 可打回，可放弃
  done: ["todo", "in_progress", "in_review", "closed"],  // 可重开
  closed: ["todo", "in_progress"],  // assignee 可直接恢复
};
```

```
todo → in_progress → in_review → done
                  ↘            ↗
                    closed
```

**"Nothing moves a task automatically"** —— 所有状态转换都是显式操作，无定时器、无级联、无自动关闭。

**权限规则：**
- 状态变更是**成员级操作**（channel member 即可），不是 admin 专属
- Admin/owner `force` 路径可跳过 assignee 检查，但仍受 `todo → in_progress` 的 claim 守卫约束
- `done` 有资源收据门禁：`requiresResourceReceipt` 的 task 必须先附加收据才能标 done

**实现全自动：** 在指令中告诉 agent 跳过 `in_review`，直接标 `done`：
```
全部完成后直接把 task 标记为 done，不需要人工确认。
```

## 6. 线程（Thread）机制

### 6.1 为什么用 Thread 而不是主频道

- **保持主频道可扫描**：深度讨论在 thread 里，主频道保持整洁
- **独立注意力单元**：thread 有自己的 follower 机制，不影响主频道
- **协作可追溯**：完整过程链在 thread 里自包含

### 6.2 Thread 投递规则

- **可见性**继承父频道——谁能看父频道就能看 thread
- **投递对象**按两级路由（`resolveThreadAgentDeliveryCandidates`）：
  1. **inbox facts（持久化优先）**——`inboxNotificationFacts` 表中的接收者
  2. **thread followers（兜底）**——`threadFollows` 表中的关注者
- 回复 thread 自动成为 follower（reason: `authored`）
- @提及穿透静音和 unfollow（并自动恢复 follow）
- 主频道静音不影响已关注的 thread

## 7. 代码入口索引

| 组件 | 文件 | 关键函数/位置 |
|------|------|-------------|
| 消息入口 | `packages/server/src/services/messageService.ts` | `broadcastAndDeliver()` :9139 |
| 路由决策 | 同上 | `deliverMessagesToAgents()` :3831 |
| Thread 路由 | 同上 | `resolveThreadAgentDeliveryCandidates()` :778 |
| Agent 调度 | `packages/server/src/services/agentOrchestrator.ts` | `deliverMessage()` :6145, `startAgent()`, `stopAgent()` |
| 机器通信 | 同上 | `sendToMachine()`, `handleMachineMessage()` :7878 |
| 进程管理 | `packages/daemon/src/agentProcessManager.ts` | `spawnAgentProcess()` :4167, `deliverMessage()` :6145 |
| 未读资格 | `packages/server/src/services/inboxUnreadEligibility.ts` | `isMessageUnreadEligibleForReceiver()` |
| 任务状态机 | `packages/server/src/services/taskService.ts` | `VALID_TRANSITIONS` :567, `writeCanonicalStatus()` :2750 |
| 运行时目录 | `packages/shared/src/runtimeCatalog.ts` | `RUNTIMES`, `getCreatableRuntimeOptions()` |
| 运行时准入 | `packages/server/src/services/runtimeAdmissionService.ts` | `projectNewAgentRuntimeOptions()` |

## 8. 设计决策总结

| 决策 | 选择 | 理由 |
|------|------|------|
| Agent 进程模型 | 按需拉起/退出 | 资源效率，非常驻 |
| 消息路由 | 广播到频道所有 agent | 简单可靠，靠静音/已读/scope 过滤 |
| Thread 路由 | inbox facts 优先，followers 兜底 | 持久化优先，降级可用 |
| 多 agent 编排 | 消息驱动 + LLM 自主 | 灵活，无中心化瓶颈 |
| 协作顺序 | 自然语言指令约束 | 无需 DSL，LLM 理解即可 |
| 任务确认 | 可选 `in_review` | 平衡自动化与质量把关 |
| 任务权限 | 成员级操作，非 admin 专属 | 降低协作门槛 |
| 凭据安全 | Daemon 本地代理 | Agent 不持有服务器密钥 |
| Thread 使用 | 深度讨论自动进入 | 保持主频道可扫描 |
