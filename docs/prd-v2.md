# Project Isotope — PRD v2

**Multi-agent orchestration platform with native Apple integration**

---

## 0. Research & Prior Work

### Existing Architecture (v0.1)

The project started as a TypeScript multi-agent orchestration platform with 6 layers:

| Layer | Component | Status |
|---|---|---|
| L1 | Agent Core — single agent lifecycle, unified LLM API, event stream | Designed |
| L2 | Agent Runtime — Docker containerized + local execution | Designed |
| L3 | Agent Iteration — workspace persistence, git-tracked prompt evolution, image snapshots | Designed |
| L4 | Async Messaging — IM-style inter-agent communication, session-key routing | Designed |
| L5 | Orchestrator — multi-agent session management, YAML declarative + programmatic API | Designed |
| L6 | Web Dashboard — conversation visualization, agent registry, workspace editor | Designed |

### Additional Research (2026-03-23)

We investigated 5 repos to expand the scope:

| Repo | Key Takeaways |
|---|---|
| **[smolagents](https://github.com/huggingface/smolagents)** (HuggingFace) | Separate core engine from high-level framework. Multi-agent orchestrator design. Tool schema conventions. |
| **[isotope-core](https://github.com/GhostComplex/isotopo-core)** (ours) | Our existing Python async agent loop engine (5.3k LoC, 97% test coverage). Providers, middleware, events, context management. |
| **[Sisyphus](https://github.com/steins-z/sisyphus)** (ours) | TypeScript prototype: daemon architecture (Unix socket IPC), `soul.md` identity system, fire-and-forget tasks. |
| **[Gemini CLI](https://github.com/google-gemini/gemini-cli)** (Google) | 20+ built-in tools, macOS Seatbelt sandbox, Linux bwrap+seccomp sandbox, tool confirmation bus, A2A server package, Conseca safety policies. |
| **[ACP/A2A](https://github.com/i-am-bee/acp)** (IBM/Linux Foundation) | Agent-to-agent protocol: HTTP-based task delegation, agent discovery, streaming. Now merging under Linux Foundation. |

### Key Decisions

| Decision | Rationale |
|---|---|
| ❌ No CodeAgent | Claude Code, Gemini CLI, Codex already do this. Wrap them as tools via `ShellAgentTool` or A2A. |
| ✅ Use isotope-core as L1 | We already have a production-quality Python agent loop. No need to rewrite L1 in TypeScript. |
| ✅ OS-level sandboxing | macOS Seatbelt + Linux bwrap — lighter than Docker-only, Apple-native. Docker remains an option. |
| ✅ Tool confirmation | Human-in-the-loop approval (allow-once / allow-always / deny) for dangerous operations. |
| ✅ A2A protocol | Both client (call external agents) and server (expose our agents). |
| ✅ Daemon architecture | From Sisyphus — always-on headless process, Unix socket + HTTP API. |
| ✅ Infra-first | Python layers are headless infrastructure with no UX opinions. Any frontend connects via the same API. |
| ✅ Apple-native product layer | Native macOS/iOS app on top of the infra, with deep OS integration. |

---

## 1. Vision

A **multi-agent orchestration platform** that works as cross-platform infrastructure, with a **native Apple product** as the primary user-facing experience.

Two audiences:
- **Developers** — use the Python infra to build, deploy, and orchestrate agents on any OS
- **Apple users** — install a native macOS/iOS app that provides an ambient personal AI with deep OS integration

### Design Principles

1. **Infra vs Product** — infrastructure is headless, cross-platform, no UX opinions. Products are platform-specific UX layers.
2. **Use isotope-core** — don't rewrite the agent loop. Build on what exists.
3. **Ship layers** — each layer is independently useful and separately packaged.
4. **Protocol-native** — MCP for tools, A2A for agent-to-agent communication.
5. **Delegate coding** — wrap existing CLI agents (Claude Code, Gemini CLI, Codex) as tools instead of building a CodeAgent.

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  PRODUCTS (platform-specific UX)                                    │
│                                                                     │
│  ┌─────────────────┐  ┌────────────────┐  ┌─────────────────────┐  │
│  │  isotope-app    │  │  Web Dashboard │  │  CLI (optional)     │  │
│  │  (Swift)        │  │  (React)       │  │  (Python)           │  │
│  │  macOS/iOS/iPad │  │  Conversation  │  │  isotope run/chat   │  │
│  │  Menu bar, Siri │  │  viz, agent    │  │  isotope daemon     │  │
│  │  iCloud sync    │  │  management    │  │  isotope tasks      │  │
│  └────────┬────────┘  └───────┬────────┘  └──────────┬──────────┘  │
│           └───────────────────┼───────────────────────┘             │
│                               │  all connect via same API          │
├───────────────────────────────┼─────────────────────────────────────┤
│  PLATFORM EXTENSIONS (optional, platform-specific tools)           │
│                               │                                     │
│  ┌────────────────────────┐   │   ┌──────────────────────────────┐  │
│  │  isotope-apple-tools   │   │   │  isotope-linux-tools (future)│  │
│  │  (Swift package)       │   │   │  D-Bus, systemd, etc.        │  │
│  │  Calendar, Mail, etc.  │   │   │                              │  │
│  │  Local HTTP bridge     │   │   │                              │  │
│  └───────────┬────────────┘   │   └──────────────┬───────────────┘  │
│              └────────────────┼──────────────────┘                  │
│                               │  register as tools via HTTP        │
├───────────────────────────────┼─────────────────────────────────────┤
│  INFRASTRUCTURE (cross-platform, headless, pure Python)            │
│                               │                                     │
│  ┌────────────────────────────▼──────────────────────────────────┐  │
│  │  isotope-agents                                               │  │
│  │                                                               │  │
│  │  Agent Types    │ Built-in Tools │ Sandbox (Seatbelt/bwrap)   │  │
│  │  ToolAgent      │ Shell, FileOps │ Tool Confirmation          │  │
│  │  MultiAgent     │ Web, Grep, Ask │ Tool Registry              │  │
│  │  Orchestrator   │ ShellAgentTool │                            │  │
│  │                 │                │                            │  │
│  │  Daemon API     │ Protocols      │ Memory & Planning          │  │
│  │  HTTP + Unix    │ MCP Client     │ AgentMemory                │  │
│  │  socket server  │ A2A Client     │ Plan → Act → Re-plan       │  │
│  │  SSE streaming  │ A2A Server     │ Context compression        │  │
│  │  Fire & forget  │ Agent-Tool     │                            │  │
│  │                 │ Bridge         │ Middleware                  │  │
│  │  Agent Runtime  │                │ Logging, Safety, Retry     │  │
│  │  Container (Docker) + Local     │ Telemetry, TokenBudget     │  │
│  │  Agent Iteration (workspace     │                            │  │
│  │  persistence, git-tracked,      │                            │  │
│  │  image snapshots)               │                            │  │
│  │                                                               │  │
│  │  Async Messaging                                              │  │
│  │  Session-key routing, message bus                             │  │
│  │  In-process (MVP) → Redis/NATS (scale)                       │  │
│  └───────────────────────────┬───────────────────────────────────┘  │
│                               │                                     │
│  ┌───────────────────────────▼───────────────────────────────────┐  │
│  │  isotope-core (exists)                                        │  │
│  │  Agent loop │ Providers │ Middleware │ Events │ Context        │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Repos

| # | Repo | Language | Type | What it owns |
|---|---|---|---|---|
| 1 | `isotope-core` | Python | Infra | Agent loop, providers, middleware, events, context. **Exists.** |
| 2 | `isotope-agents` | Python | Infra | Agent types, built-in tools, sandbox, tool confirmation, daemon API (HTTP + Unix socket), protocols (MCP/A2A), memory/planning, agent runtime (Docker + local), async messaging, agent iteration (workspace persistence), tool registry, middleware. Optional CLI as `[cli]` extra. |
| 3 | `isotope-apple-tools` | Swift | Platform ext. | Apple framework tools (Calendar, Mail, Reminders, etc.) + local HTTP bridge server. Reusable Swift package. |
| 4 | `isotope-app` | Swift | Product | Native macOS/iOS/iPadOS app. SwiftUI, iCloud sync. Talks to isotope-agents daemon + isotope-apple-tools. |
| — | `project-isotope` | — | Meta | Project docs, PRDs, research notes, architecture decisions. This repo. |

---

## 4. Daemon API

The daemon is a headless HTTP + Unix socket server. **All frontends** (macOS app, web dashboard, CLI, external agents) connect through the same API.

```
POST /agents                         # Create/configure agent
GET  /agents                         # List agents
POST /agents/{id}/run                # Submit task (fire-and-forget)
GET  /agents/{id}/stream             # SSE event stream
POST /agents/{id}/chat               # Interactive message
POST /agents/{id}/steer              # Mid-run steering
DELETE /agents/{id}                  # Remove agent

GET  /sessions                       # List sessions
GET  /sessions/{id}                  # Get session state + history
DELETE /sessions/{id}                # Cancel/cleanup

POST /tasks                          # Submit fire-and-forget task
GET  /tasks                          # List tasks
GET  /tasks/{id}                     # Get task status + result

POST /tools/register                 # Register external tools (HTTP bridge)
GET  /tools                          # List registered tools
POST /tools/mcp                      # Load tools from MCP server
POST /tools/a2a                      # Load A2A agent as tool

GET  /.well-known/agent.json         # A2A agent card (expose as A2A server)
POST /a2a/tasks                      # A2A task endpoint
```

---

## 5. Built-in Tools

### Cross-Platform (ship with isotope-agents)

| Tool | Description | Priority |
|---|---|---|
| `FinalAnswerTool` | Terminates agent loop with result | P0 |
| `ShellTool` | Execute shell commands | P0 |
| `FileReadTool` | Read file contents | P0 |
| `FileWriteTool` | Write/create files | P0 |
| `FileEditTool` | Diff-based file editing (search & replace) | P0 |
| `GlobTool` / `LsTool` | List files, glob patterns | P0 |
| `GrepTool` | Search file contents (ripgrep-backed) | P0 |
| `WebSearchTool` | Web search (Brave/SerpAPI/Tavily) | P0 |
| `WebFetchTool` | Fetch and extract content from URLs | P0 |
| `AskUserTool` | Pause and ask human for input | P0 |
| `MemoryTool` | Read/write session memory | P1 |
| `ShellAgentTool` | Wrap CLI agents (Claude Code, Gemini CLI, Codex) as tools | P1 |
| `TodoTool` | Task/todo management within a session | P2 |
| `HttpRequestTool` | Make arbitrary HTTP requests | P2 |

### Apple Platform (ship with isotope-apple-tools)

| Tool | Framework | Priority |
|---|---|---|
| `CalendarTool` | EventKit | P0 |
| `RemindersTool` | EventKit | P0 |
| `ContactsTool` | Contacts.framework | P0 |
| `NotesTool` | AppleScript/Shortcuts | P1 |
| `MailTool` | MessageUI/AppleScript | P1 |
| `ClipboardTool` | NSPasteboard/UIPasteboard | P1 |
| `NotificationTool` | UserNotifications | P1 |
| `ShortcutsTool` | Shortcuts.framework | P1 |
| `PhotosTool` | PhotoKit | P2 |
| `FilesTool` | FileManager + iCloud | P2 |
| `LocationTool` | CoreLocation | P2 |
| `SystemInfoTool` | Various | P2 |
| `FocusModeTool` | — | P2 |
| `AppControlTool` | NSWorkspace/UIApplication | P2 |

---

## 6. Sandbox / Tool Isolation

| Backend | Platform | How it works | Setup |
|---|---|---|---|
| `OSSandbox` | macOS | `sandbox-exec` with Seatbelt profiles (deny-all + allowlist) | None (built-in) |
| `OSSandbox` | Linux | `bwrap` (bubblewrap) + seccomp BPF filters | `bwrap` installed |
| `DockerSandbox` | Any | Run tools inside Docker containers | Docker installed |

Modes: `off` (default dev) / `on` (all tools sandboxed) / `per-tool` (configure individually)

---

## 7. Agent Runtime & Iteration

From the original v0.1 architecture — agent containerization and workspace persistence:

### Runtime Modes

| Mode | Description | Use Case |
|---|---|---|
| **Local** | Agent runs in host process, no isolation | Dev/debug |
| **Docker** | Agent runs in container, volume-mounted workspace | Production, multi-agent |
| **OS Sandbox** | Agent runs on host but tools are sandboxed (Seatbelt/bwrap) | Single-agent, Apple app |

### Agent Iteration (Docker mode)

```
Container: /agent/workspace (rw, volume mount)
           /agent/immutable (ro, baked in image)
                    │
                    ▼ volume mount
Host: /data/agents/{id}/workspace → git auto-tracked
```

- Agent modifies its workspace (prompts, memory) at runtime
- Changes persist via volume mount and are git-tracked
- `isotope snapshot {id}` → bake current workspace into new image
- `isotope push/pull` → distribute via container registry (GHCR)

### Agent Identity Files

```
~/.isotope/agents/{name}/
├── soul.md              # Personality, capabilities, system prompt
├── agent.config.yaml    # Model, tools, sandbox settings
├── tools/               # Custom tool implementations
└── memory/              # Persistent memory
```

---

## 8. Async Messaging (Multi-Agent)

IM-style async inter-agent communication:

```typescript
interface Message {
  id: string;
  sessionId: string;
  senderId: string;       // agent ID
  recipientId?: string;   // null = broadcast to session
  content: string;
  attachments?: Attachment[];
  metadata: { timestamp, replyTo?, threadId? };
}
```

- Session-key routing (from OpenClaw pattern): `agent:{agentId}:{sessionId}`
- Fire-and-forget delivery, per-agent inbox queues
- MVP: in-process event bus
- Scale: Redis Streams / NATS

---

## 9. Protocol Integrations

| Protocol | Direction | What it does |
|---|---|---|
| **MCP Client** | isotope → external tool servers | Load tools from any MCP server |
| **A2A Client** | isotope → external agents | Delegate tasks to external agents (Gemini CLI, LangGraph, etc.) |
| **A2A Server** | external agents → isotope | Expose isotope agents as A2A-compatible services |
| **Agent-Tool Bridge** | A2A agent → isotope tool | Wrap any A2A agent as an isotope tool |
| **Shell Agent Bridge** | CLI agent → isotope tool | Wrap Claude Code / Gemini CLI / Codex as tools |
| **Apple Tools Bridge** | isotope ← HTTP → Swift | Register Apple platform tools from isotope-apple-tools |

---

## 10. macOS / iOS App (isotope-app)

### macOS

- **Menu bar agent** — always-available, click to open chat panel
- **Global hotkey** (⌘⇧Space) — floating input bar for quick prompts
- **Chat interface** — streaming, tool call visualization, session history
- **Settings** — model providers, tool permissions, agent management
- **System integration** — Notifications, Shortcuts, Share sheet, Widgets

### iOS / iPadOS

- Chat interface, Siri Shortcuts, Widgets, Share sheet
- Background processing, push notifications
- iCloud sync (configs, history, Keychain for API keys)

---

## 11. Web Dashboard

Conversation visualization and agent management (from v0.1 L6):

- **Agent Registry** — view/manage all agents and images
- **Sessions** — view active sessions, enter any session to observe
- **Conversation View** — real-time multi-agent conversation with per-agent color coding, tool call expansion
- **Workspace Editor** — edit agent prompts/config online, trigger snapshots
- **Task Monitor** — view fire-and-forget task status

Tech: React + WebSocket, connects to daemon API.

Priority: P2 — after macOS app.

---

## 12. Delivery Plan (with AI dev agent)

| Phase | Days | Deliverable | What's Usable |
|---|---|---|---|
| **1: Python Foundation** | 3-5 | `pip install isotope-agents` on PyPI | ToolAgent + 10 built-in tools + sandbox + confirmation. Run agents from Python code on any OS. |
| **2: Multi-Agent & Daemon** | 2-4 | Daemon running as background service | Headless daemon API (HTTP + Unix socket). Multi-agent orchestrator. ShellAgentTool wraps Claude Code/Gemini CLI. MCP client. Fire-and-forget tasks. Agent identities (`soul.md`). |
| **3: Apple Tools** | 4-7 | `isotope-apple-tools` Swift package | 14 Apple tools accessible from Python agent. "What's on my calendar?" works. |
| **4: macOS App MVP** | 5-8 | macOS menu bar app | Native app with chat, hotkey, streaming, tool visualization. The product. |
| **5: A2A & Ecosystem** | 2-3 | A2A interop + web dashboard | Agents callable by/calling external agents. Web dashboard for conversation viz. |
| **6: iOS App** | 4-6 | App Store submission | iPhone/iPad app with Siri, widgets, share sheet, iCloud sync. |
| **7: Polish** | ongoing | MLX, voice, handoff | Local models, voice I/O, multi-device handoff. |

**Total to macOS app: ~14-24 days. Total to iOS: ~20-33 days.**

Realistic with human review cycles: **4-6 weeks to full stack.**

---

## 13. Open Questions

1. **Monorepo vs multi-repo** — existing v0.1 was a monorepo. Current plan is 4 separate repos + this meta repo. Which approach?
2. **TypeScript vs Python** — v0.1 was TypeScript, isotope-core is Python. Decision: **Python for infra** (leverage isotope-core), TypeScript for web dashboard only?
3. **Product name** — `isotope` for everything, or a friendlier consumer name for the app?
4. **Distribution** — Mac App Store vs direct download vs both?
5. **Python embedding** — bundle Python runtime in macOS app, or require user to install?
6. **Container registry** — GHCR for agent images, or self-hosted?
7. **License** — MIT for all packages, or different for the app?
8. **Async messaging backend** — in-process EventEmitter for MVP. When to introduce Redis/NATS?

---

## 14. Non-Goals (for v1)

- CodeAgent / built-in code execution (delegate to existing CLI agents)
- Android/Windows native apps
- Browser extension
- K8s multi-node clustering (designed for, but not built in v1)
- Enterprise features (SSO, team management, billing)
- Training or fine-tuning integration
- Third-party agent marketplace
