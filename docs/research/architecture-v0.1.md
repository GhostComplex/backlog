# Project Isotope — Technical Architecture (Draft v0.1)

> Multi-agent orchestration platform for developers
> MVP-driven, bottom-up layered design

## Overview

6 layers, each independently shippable:

```
┌─────────────────────────────────────┐
│  L6: Web Dashboard (展示层)          │
├─────────────────────────────────────┤
│  L5: Orchestrator (多 agent 编排)    │
├─────────────────────────────────────┤
│  L4: Async Messaging Protocol       │
├─────────────────────────────────────┤
│  L3: Agent Iteration & Persistence  │
├─────────────────────────────────────┤
│  L2: Agent Runtime (容器化/非容器化)  │
├─────────────────────────────────────┤
│  L1: Agent Core (单 agent 生命周期)  │
└─────────────────────────────────────┘
```

---

## L1: Agent Core

**职责：** 单个 agent 的完整生命周期管理

**参考：** pi-agent-core 的事件流架构

### 核心概念

```typescript
interface AgentCore {
  id: string;
  systemPrompt: string;
  model: ModelConfig;
  tools: Tool[];
  state: AgentState; // idle | thinking | tool_calling | error
}

interface AgentEvent {
  type: 'agent_start' | 'turn_start' | 'message_start' | 'message_update' 
      | 'message_end' | 'tool_call_start' | 'tool_result' | 'turn_end' | 'agent_end';
  timestamp: number;
  agentId: string;
  sessionId: string;
  payload: unknown;
}
```

### Model Provider Support (MVP)
- OpenAI (GPT-4o, o1, o3)
- Anthropic (Claude Sonnet/Opus)
- GitHub Copilot API (参考 OpenClaw 的 `pi-ai` GitHub Copilot OAuth 实现)

### MVP 交付物
- `@isotope/agent-core` package
- 统一 LLM API（可直接用 `@mariozechner/pi-ai` 或 fork 精简版）
- Event emitter 架构，所有状态变化通过事件流出
- Tool calling 框架

---

## L2: Agent Runtime

**职责：** 在容器化或非容器化环境中运行 agent

**参考：** OpenClaw sandbox（`src/agents/sandbox/`）

### 两种运行模式

#### 非容器化（Local）
- 直接在宿主进程中运行 agent
- 适合开发调试
- 类似 OpenClaw 的 host mode

#### 容器化（Docker）
- 每个 agent 是一个 Docker container
- Agent image = base runtime + workspace（含 prompts）
- 支持 replica：同一个 image 可 spawn 多个实例

### Agent Image 规范

```dockerfile
FROM isotope/agent-runtime:latest

# Agent 的核心 prompt 和配置
COPY workspace/ /agent/workspace/
COPY agent.config.yaml /agent/config.yaml

# Workspace 包含：
# - SYSTEM_PROMPT.md (核心 system prompt)
# - TOOLS.md (工具配置)
# - AGENTS.md (行为规范)
# - skills/ (技能包)
```

```yaml
# agent.config.yaml
name: my-agent
version: 1.0.0
model:
  provider: anthropic
  model: claude-sonnet-4-20250514
tools:
  - read
  - write
  - exec
sandbox:
  network: restricted  # none | restricted | full
  memory: 512m
  cpu: 1
```

### MVP 交付物
- `@isotope/agent-runtime` package
- Docker image builder CLI: `isotope build`
- Container lifecycle: create / start / stop / destroy
- 非容器化 fallback for dev

---

## L3: Agent Iteration & Persistence

**职责：** Agent 的 prompt 迭代和状态持久化

### 核心问题
容器是无状态的，但 agent 需要：
1. **Prompt 迭代** — agent 在运行中可能修改自己的 prompt/workspace
2. **状态持久化** — 容器销毁后，迭代结果不丢失
3. **版本化发布** — 迭代后的 prompt 可以打包成新版本镜像

### 设计

```
┌──────────────────────────────────┐
│  Docker Container                │
│  ┌────────────────────────────┐  │
│  │  /agent/workspace (rw)     │  │  ← volume mount
│  │  - SYSTEM_PROMPT.md        │  │
│  │  - AGENTS.md               │  │
│  │  - memory/                 │  │
│  └────────────────────────────┘  │
│  ┌────────────────────────────┐  │
│  │  /agent/immutable (ro)     │  │  ← baked into image
│  │  - base config, tools      │  │
│  └────────────────────────────┘  │
└──────────────────────────────────┘
         │
         ▼ volume mount
┌──────────────────────────────────┐
│  Host: /data/agents/{id}/workspace │
│  Git-tracked, auto-commit        │
└──────────────────────────────────┘
```

**迭代流程：**
1. Agent 运行中修改 `/agent/workspace/` 下的文件
2. 修改实时写入 host volume（持久化）
3. Host 侧用 git 自动 track 变更
4. 用户可以 `isotope snapshot {agent-id}` 将当前 workspace 打包为新版本镜像
5. 新镜像可推送到 registry，分发给其他节点

```bash
# 迭代后打包新版本
isotope snapshot my-agent --tag v1.1.0

# 推送到 registry
isotope push my-agent:v1.1.0

# 其他节点拉取并升级
isotope pull my-agent:v1.1.0
isotope upgrade my-agent --to v1.1.0
```

### MVP 交付物
- Volume mount 方案实现
- `isotope snapshot` CLI
- Git-based workspace versioning

---

## L4: Async Messaging Protocol

**职责：** Agent 之间的异步、非阻塞通信

**参考：** OpenClaw 的 session-based 消息路由

### 设计原则
- 消息是异步的，像 IM 一样 fire-and-forget
- 每条消息有 sender、recipient、session context
- 支持 1:1 和 group（多 agent 在同一 session）

### 消息模型

```typescript
interface Message {
  id: string;
  sessionId: string;
  senderId: string;      // agent ID
  recipientId?: string;  // null = broadcast to session
  content: string;
  attachments?: Attachment[];
  metadata: {
    timestamp: number;
    replyTo?: string;     // 引用消息 ID
    threadId?: string;
  };
}

interface Session {
  id: string;
  name: string;
  participants: string[];  // agent IDs
  createdAt: number;
  metadata: Record<string, unknown>;
}
```

### 消息流转

```
Agent A                    Message Bus                  Agent B
   │                           │                           │
   │── send(msg) ─────────────►│                           │
   │                           │── deliver(msg) ──────────►│
   │                           │                           │── process ──►
   │                           │                           │
   │                           │◄── send(reply) ───────────│
   │◄── deliver(reply) ────────│                           │
```

### Message Bus 实现（MVP）
- **进程内：** 简单的 EventEmitter / pub-sub（单节点）
- **跨节点（后续）：** Redis Streams 或 NATS

### 关键参考（OpenClaw）
OpenClaw 的消息路由核心：
- `src/routing/session-key.ts` — session key 构建规则：`agent:{agentId}:{channel}:{peerKind}:{peerId}`
- `src/agents/command/delivery.ts` — 消息投递到 channel
- `src/agents/tools/sessions-send-tool.ts` — agent 间消息发送
- `src/agents/subagent-registry.ts` — 子 agent 注册和生命周期追踪

OpenClaw 的精华在于 **session key 是消息路由的唯一标识**，agent 通过 session key 找到对方，不关心对方在哪个容器/进程里。我们沿用这个设计。

### MVP 交付物
- `@isotope/messaging` package
- Session CRUD API
- 进程内 message bus
- Agent → Session → Agent 的消息路由

---

## L5: Multi-Agent Orchestrator

**职责：** 多 agent 的编排、调度、生命周期管理

**参考：** OpenClaw 的 subagent system + session routing

### Session 模型

```
Session "code-review-123"
├── Agent: reviewer (claude-opus)
├── Agent: coder (claude-sonnet)  
└── Agent: tester (gpt-4o)

所有 agent 共享同一个 session 的消息流
每个 agent 独立运行，异步响应
```

### 编排方式

#### Declarative（声明式）
```yaml
# orchestration.yaml
name: code-review-pipeline
session:
  agents:
    - id: reviewer
      image: isotope/code-reviewer:latest
      model: anthropic/claude-opus
      role: "Review code changes for quality and security"
    - id: coder
      image: isotope/coder:latest
      model: anthropic/claude-sonnet
      role: "Implement code based on review feedback"
    - id: tester
      image: isotope/tester:latest
      model: openai/gpt-4o
      role: "Write and run tests for implemented code"
  
  # 谁能看到谁的消息
  visibility: full  # full | directed | custom
```

#### Programmatic（编程式）
```typescript
import { Orchestrator, Session } from '@isotope/orchestrator';

const orch = new Orchestrator();

const session = await orch.createSession('code-review');
await session.addAgent({ id: 'reviewer', image: 'isotope/code-reviewer:latest' });
await session.addAgent({ id: 'coder', image: 'isotope/coder:latest' });

// 发送消息到 session，所有 agent 都能看到
await session.send({ content: 'Review this PR: #123' });

// 或者定向发送
await session.sendTo('reviewer', { content: 'Focus on security issues' });
```

### Agent 调度
- **单节点 MVP：** 所有 agent 容器在同一台机器上
- **多节点（后续）：** K8s-based，每个 agent 是一个 Pod

### MVP 交付物
- `@isotope/orchestrator` package
- Session-based multi-agent 编排
- YAML 声明式 + TypeScript 编程式 API
- Agent 启动/停止/重启

---

## L6: Web Dashboard

**职责：** 可视化管理界面

**参考：** pi-web-ui 的 chat 组件

### 核心页面

1. **Agent Registry** — 查看/管理所有 agent images
2. **Sessions** — 查看所有活跃 session，进入任意 session 观察对话
3. **Agent Detail** — 单个 agent 的状态、配置、workspace 文件
4. **Conversation View** — 实时对话流，支持多 agent 消息交叉显示
5. **Workspace Editor** — 在线编辑 agent 的 prompt/config，触发 snapshot

### 对话可视化

```
Session: code-review-123
─────────────────────────────────────
[reviewer] 🔵 I've reviewed PR #123. Found 3 issues:
           1. SQL injection in user.ts:45
           2. Missing error handling in api.ts:23
           3. Unused import in utils.ts:1

[coder]    🟢 Got it. Fixing issues 1 and 2 now...
           [tool: edit file user.ts]
           [tool: edit file api.ts]
           Fixed. Ready for re-review.

[tester]   🟡 Running test suite...
           [tool: exec npm test]
           All 47 tests passing. Coverage: 89%

[reviewer] 🔵 LGTM. Approve.
─────────────────────────────────────
```

每个 agent 有独立颜色/标识，tool call 可展开查看详情。

### 技术栈
- **Frontend:** React + Tailwind（或考虑复用 pi-web-ui 的 web components）
- **Backend:** Node.js + WebSocket（实时消息推送）
- **API:** REST + WebSocket

### MVP 交付物
- Session 列表 + 对话查看
- Agent 列表 + 状态展示
- 实时消息流（WebSocket）

---

## 技术栈总结

| 层 | 核心技术 | 复用 |
|---|---|---|
| L1 Agent Core | TypeScript, pi-ai | pi-ai (LLM API) |
| L2 Runtime | Docker API, Node.js | OpenClaw sandbox 思路 |
| L3 Iteration | Git, Docker build | - |
| L4 Messaging | EventEmitter → Redis/NATS | OpenClaw session key 设计 |
| L5 Orchestrator | TypeScript, YAML | OpenClaw subagent 思路 |
| L6 Dashboard | React, WebSocket | pi-web-ui chat 组件 |

## MVP 路线建议

**Phase 1 (M1):** L1 + L2（能跑单个 agent，容器化和非容器化）
**Phase 2 (M2):** L4 + L5（两个 agent 能在一个 session 里异步通信）
**Phase 3 (M3):** L3 + L6（prompt 持久化/迭代 + Web 可视化）

每个 phase 2-3 周，总计 6-9 周到 MVP。

---

## 待讨论

1. **Monorepo 还是多 repo？** — 建议 monorepo（参考 pi-mono），packages 按层拆分
2. **Agent image registry** — 用 Docker Hub/GitHub Container Registry，还是自建？MVP 建议直接用 GHCR
3. **Auth 模型** — API key per agent？还是统一 credential store？
4. **多节点扩展** — 何时引入 K8s？MVP 先不需要
5. **命名** — `isotope` 这个名字 OK？CLI 命令就是 `isotope`
