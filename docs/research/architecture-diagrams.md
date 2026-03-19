# Project Isotope — Architecture Diagrams

## 1. System Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Web Dashboard (L6)                           │
│  ┌──────────┐  ┌──────────────┐  ┌────────────┐  ┌──────────────┐ │
│  │  Agent    │  │  Session     │  │ Conversation│  │  Workspace   │ │
│  │  Registry │  │  Manager     │  │  Viewer     │  │  Editor      │ │
│  └────┬─────┘  └──────┬───────┘  └──────┬─────┘  └──────┬───────┘ │
│       └───────────┬────┴────────────┬────┴───────────────┘         │
│                   │   REST + WebSocket                              │
└───────────────────┼─────────────────────────────────────────────────┘
                    │
┌───────────────────┼─────────────────────────────────────────────────┐
│                   ▼   Orchestrator (L5)                              │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                    Session Manager                             │ │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌──────────────────┐ │ │
│  │  │Session A │  │Session B │  │Session C │  │  Agent Scheduler │ │ │
│  │  │┌──┐┌──┐ │  │┌──┐┌──┐ │  │┌──┐┌──┐ │  │  - start/stop    │ │ │
│  │  ││A1││A2│ │  ││A3││A4│ │  ││A1││A5│ │  │  - replica mgmt  │ │ │
│  │  │└──┘└──┘ │  │└──┘└──┘ │  │└──┘└──┘ │  │  - health check  │ │ │
│  │  └─────────┘  └─────────┘  └─────────┘  └──────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Config Parser (YAML declarative + TypeScript programmatic)    │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                    │
┌───────────────────┼─────────────────────────────────────────────────┐
│                   ▼   Async Messaging (L4)                          │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                     Message Bus                                │ │
│  │                                                                │ │
│  │  ┌──────────┐    ┌──────────────┐    ┌──────────────────────┐ │ │
│  │  │  Router   │───▶│  Session     │───▶│  Delivery Queue      │ │ │
│  │  │          │    │  Key Index   │    │  (per agent inbox)   │ │ │
│  │  └──────────┘    └──────────────┘    └──────────────────────┘ │ │
│  │                                                                │ │
│  │  Session Key Format:                                           │ │
│  │  agent:{agentId}:{sessionId}                                   │ │
│  │  agent:{agentId}:{channel}:{peerKind}:{peerId}                 │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  MVP: in-process EventEmitter                                       │
│  Future: Redis Streams / NATS                                       │
└─────────────────────────────────────────────────────────────────────┘
                    │
┌───────────────────┼─────────────────────────────────────────────────┐
│                   ▼   Agent Iteration & Persistence (L3)            │
│                                                                     │
│  ┌──────────────────────┐    ┌──────────────────────────────────┐  │
│  │  Workspace Volume    │    │  Version Manager                 │  │
│  │  /data/agents/{id}/  │    │                                  │  │
│  │  ├── workspace/      │───▶│  git auto-commit on change      │  │
│  │  │   ├── PROMPT.md   │    │  isotope snapshot → new image   │  │
│  │  │   ├── AGENTS.md   │    │  isotope push → registry        │  │
│  │  │   ├── TOOLS.md    │    │  isotope pull → other nodes     │  │
│  │  │   └── memory/     │    │                                  │  │
│  │  └── config.yaml     │    └──────────────────────────────────┘  │
│  └──────────────────────┘                                          │
└─────────────────────────────────────────────────────────────────────┘
                    │
┌───────────────────┼─────────────────────────────────────────────────┐
│                   ▼   Agent Runtime (L2)                            │
│                                                                     │
│  ┌─────────────────────────┐  ┌──────────────────────────────────┐ │
│  │  Containerized (Docker)  │  │  Non-containerized (Local)       │ │
│  │                          │  │                                  │ │
│  │  ┌────────────────────┐ │  │  ┌────────────────────────────┐ │ │
│  │  │ isotope/agent-rt   │ │  │  │  Direct Node.js process    │ │ │
│  │  │ ┌────────────────┐ │ │  │  │  Same APIs, no isolation   │ │ │
│  │  │ │ /agent/workspace│ │ │  │  │  Dev/debug mode            │ │ │
│  │  │ │ (volume mount) │ │ │  │  └────────────────────────────┘ │ │
│  │  │ ├────────────────┤ │ │  │                                  │ │
│  │  │ │ /agent/immutable│ │ │  │                                  │ │
│  │  │ │ (baked in image)│ │ │  │                                  │ │
│  │  │ └────────────────┘ │ │  │                                  │ │
│  │  └────────────────────┘ │  │                                  │ │
│  │                          │  │                                  │ │
│  │  Supports N replicas     │  │  Single instance only            │ │
│  │  Network isolation       │  │  Host network                    │ │
│  │  Resource limits         │  │  No limits                       │ │
│  └─────────────────────────┘  └──────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                    │
┌───────────────────┼─────────────────────────────────────────────────┐
│                   ▼   Agent Core (L1)                               │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                      Agent Instance                            │ │
│  │                                                                │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │ │
│  │  │  System   │  │  Model    │  │  Tool     │  │  State       │ │ │
│  │  │  Prompt   │  │  Provider │  │  Registry │  │  Machine     │ │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────────┘ │ │
│  │                                                                │ │
│  │  ┌──────────────────────────────────────────────────────────┐ │ │
│  │  │                   Event Stream                           │ │ │
│  │  │  agent_start → turn_start → message_start → msg_update  │ │ │
│  │  │  → msg_end → tool_call → tool_result → turn_end         │ │ │
│  │  │  → agent_end                                             │ │ │
│  │  └──────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  LLM Provider Abstraction (via pi-ai)                         │ │
│  │  ┌──────────┐  ┌───────────┐  ┌─────────────────────────┐   │ │
│  │  │  OpenAI   │  │ Anthropic  │  │ GitHub Copilot API      │   │ │
│  │  └──────────┘  └───────────┘  └─────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

## 2. Message Flow (Multi-Agent Session)

```
┌─────────┐          ┌───────────┐          ┌─────────┐
│ Agent A  │          │ Message   │          │ Agent B  │
│ (Coder)  │          │ Bus       │          │(Reviewer)│
└────┬─────┘          └─────┬─────┘          └────┬─────┘
     │                      │                      │
     │  send(msg, session)  │                      │
     │─────────────────────▶│                      │
     │                      │                      │
     │                      │  route by session    │
     │                      │  key, find all       │
     │                      │  participants        │
     │                      │                      │
     │                      │  deliver(msg)        │
     │                      │─────────────────────▶│
     │                      │                      │
     │                      │         ┌────────────┤
     │                      │         │ process    │
     │                      │         │ (LLM call) │
     │                      │         └────────────┤
     │                      │                      │
     │                      │  send(reply, session)│
     │                      │◀─────────────────────│
     │                      │                      │
     │  deliver(reply)      │                      │
     │◀─────────────────────│                      │
     │                      │                      │
     ▼                      ▼                      ▼

  Messages are async (fire-and-forget)
  Each agent has its own inbox queue
  No blocking — agents process at their own pace
```

## 3. Agent Containerization & Iteration Lifecycle

```
  Developer                    Platform                     Registry
     │                            │                            │
     │  isotope init my-agent     │                            │
     │───────────────────────────▶│                            │
     │                            │  Create workspace/         │
     │                            │  ├── PROMPT.md             │
     │                            │  ├── agent.config.yaml     │
     │                            │  └── tools/                │
     │                            │                            │
     │  isotope build my-agent    │                            │
     │───────────────────────────▶│                            │
     │                            │  Build Docker image        │
     │                            │  isotope/my-agent:v1.0.0   │
     │                            │                            │
     │  isotope run my-agent      │                            │
     │───────────────────────────▶│                            │
     │                            │  ┌──────────────────────┐  │
     │                            │  │  Container running   │  │
     │                            │  │  workspace = volume  │  │
     │                            │  │  Agent modifies own  │  │
     │                            │  │  PROMPT.md, memory/  │  │
     │                            │  └──────────────────────┘  │
     │                            │         │                  │
     │                            │  git auto-tracks changes   │
     │                            │         │                  │
     │  isotope snapshot          │         │                  │
     │  my-agent --tag v1.1.0     │◀────────┘                  │
     │───────────────────────────▶│                            │
     │                            │  Bake current workspace    │
     │                            │  into new image layer      │
     │                            │                            │
     │  isotope push              │                            │
     │  my-agent:v1.1.0           │  docker push               │
     │───────────────────────────▶│───────────────────────────▶│
     │                            │                            │
     │                            │       Other Node           │
     │                            │           │                │
     │                            │  isotope pull              │
     │                            │  my-agent:v1.1.0           │
     │                            │           │◀───────────────│
     │                            │  isotope upgrade           │
     │                            │  my-agent --to v1.1.0      │
     │                            │           │                │
     │                            │  New container with        │
     │                            │  evolved prompts           │
     ▼                            ▼           ▼                ▼
```

## 4. Orchestrator Session Model

```
┌─────────────────────────────────────────────────────────┐
│                   Orchestrator                           │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Session "code-review-42"                         │  │
│  │                                                   │  │
│  │  participants: [reviewer, coder, tester]          │  │
│  │  visibility: full                                 │  │
│  │                                                   │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │              Message History                 │  │  │
│  │  │                                             │  │  │
│  │  │  [user]     Review PR #123                  │  │  │
│  │  │  [reviewer] Found 3 issues: ...             │  │  │
│  │  │  [coder]    Fixed. Ready for re-review.     │  │  │
│  │  │  [tester]   All 47 tests passing.           │  │  │
│  │  │  [reviewer] LGTM. Approved.                 │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Session "debug-incident-7"                       │  │
│  │                                                   │  │
│  │  participants: [oncall-bot, log-analyzer]         │  │
│  │  visibility: full                                 │  │
│  │                                                   │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │  [user]        Server 500 errors spiking    │  │  │
│  │  │  [log-analyzer] Found pattern in logs: ...  │  │  │
│  │  │  [oncall-bot]   Paging team, ETA 5min       │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  Agent Pool:                                            │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌────────────┐ ┌────────┐ │
│  │ revw │ │ coder│ │tester│ │log-analyzer│ │oncall  │ │
│  │ (x1) │ │ (x2) │ │ (x1) │ │   (x1)     │ │ (x1)  │ │
│  └──────┘ └──────┘ └──────┘ └────────────┘ └────────┘ │
│                                                         │
│  Same agent can join multiple sessions (e.g. reviewer)  │
└─────────────────────────────────────────────────────────┘
```

## 5. Package Structure (Monorepo)

```
project-isotope/
├── packages/
│   ├── agent-core/          # L1: Agent lifecycle, LLM abstraction
│   │   ├── src/
│   │   │   ├── agent.ts          # Agent class
│   │   │   ├── events.ts         # Event types & emitter
│   │   │   ├── tools.ts          # Tool framework
│   │   │   ├── models/
│   │   │   │   ├── openai.ts
│   │   │   │   ├── anthropic.ts
│   │   │   │   └── github-copilot.ts
│   │   │   └── index.ts
│   │   └── package.json
│   │
│   ├── agent-runtime/       # L2: Container & local execution
│   │   ├── src/
│   │   │   ├── docker.ts         # Docker container lifecycle
│   │   │   ├── local.ts          # Non-containerized runner
│   │   │   ├── image-builder.ts  # Dockerfile generation
│   │   │   └── index.ts
│   │   └── package.json
│   │
│   ├── agent-iteration/     # L3: Persistence & versioning
│   │   ├── src/
│   │   │   ├── workspace.ts      # Volume mount management
│   │   │   ├── git-tracker.ts    # Auto-commit on change
│   │   │   ├── snapshot.ts       # Build new image from workspace
│   │   │   └── index.ts
│   │   └── package.json
│   │
│   ├── messaging/           # L4: Async message protocol
│   │   ├── src/
│   │   │   ├── bus.ts            # Message bus (EventEmitter MVP)
│   │   │   ├── session.ts        # Session management
│   │   │   ├── router.ts         # Session-key-based routing
│   │   │   ├── types.ts          # Message, Session types
│   │   │   └── index.ts
│   │   └── package.json
│   │
│   ├── orchestrator/        # L5: Multi-agent coordination
│   │   ├── src/
│   │   │   ├── orchestrator.ts   # Main orchestrator class
│   │   │   ├── scheduler.ts      # Agent start/stop/replica
│   │   │   ├── config-parser.ts  # YAML config loader
│   │   │   └── index.ts
│   │   └── package.json
│   │
│   ├── dashboard/           # L6: Web UI
│   │   ├── src/
│   │   │   ├── app/
│   │   │   ├── components/
│   │   │   │   ├── SessionList.tsx
│   │   │   │   ├── ConversationView.tsx
│   │   │   │   ├── AgentRegistry.tsx
│   │   │   │   └── WorkspaceEditor.tsx
│   │   │   └── api/
│   │   │       ├── rest.ts
│   │   │       └── websocket.ts
│   │   └── package.json
│   │
│   └── cli/                 # CLI tool
│       ├── src/
│       │   ├── commands/
│       │   │   ├── init.ts
│       │   │   ├── build.ts
│       │   │   ├── run.ts
│       │   │   ├── snapshot.ts
│       │   │   ├── push.ts
│       │   │   └── pull.ts
│       │   └── index.ts
│       └── package.json
│
├── docker/
│   └── agent-runtime/
│       └── Dockerfile            # Base agent runtime image
│
├── docs/
│   └── research/
│       ├── research-notes.md
│       └── architecture-v0.1.md
│
├── package.json                  # Workspace root
├── tsconfig.base.json
└── README.md
```

## 6. Future: Multi-Node Clustering (K8s)

```
┌─────────────────────────────────────────────────────────────┐
│                      K8s Cluster                             │
│                                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Isotope Control Plane (Deployment)                    │ │
│  │  ├── Orchestrator                                      │ │
│  │  ├── Message Bus (backed by Redis/NATS)                │ │
│  │  ├── Dashboard API                                     │ │
│  │  └── Agent Registry                                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  Node 1       │  │  Node 2       │  │  Node 3       │   │
│  │  ┌──────────┐ │  │  ┌──────────┐ │  │  ┌──────────┐ │   │
│  │  │ Agent A  │ │  │  │ Agent C  │ │  │  │ Agent E  │ │   │
│  │  │ (Pod)    │ │  │  │ (Pod)    │ │  │  │ (Pod)    │ │   │
│  │  └──────────┘ │  │  └──────────┘ │  │  └──────────┘ │   │
│  │  ┌──────────┐ │  │  ┌──────────┐ │  │  ┌──────────┐ │   │
│  │  │ Agent B  │ │  │  │ Agent D  │ │  │  │ Agent A  │ │   │
│  │  │ (Pod)    │ │  │  │ (Pod)    │ │  │  │ replica  │ │   │
│  │  └──────────┘ │  │  └──────────┘ │  │  │ (Pod)    │ │   │
│  │               │  │               │  │  └──────────┘ │   │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│                                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Shared Storage                                        │ │
│  │  ├── PersistentVolumes (agent workspaces)              │ │
│  │  ├── Redis (message bus)                               │ │
│  │  └── PostgreSQL (session history, agent registry)      │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘

Agent ←──Message Bus (Redis/NATS)──→ Agent
         Cross-node transparent
         Same session key routing
```
