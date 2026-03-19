# Project Isotope - Research Notes

## Date: 2026-03-19

## Goal
Build a multiagent platform covering: multiagent system setup, distribution, conversation visualization, and clustering.

## Reference Projects Analyzed

### OpenClaw (openclaw/openclaw)
- **What:** Personal AI assistant gateway, single-user, runs on your own device
- **Architecture:**
  - Gateway (Node.js) = control plane
  - Multi-channel support (WhatsApp, Telegram, Slack, Discord, Signal, etc.)
  - Plugin-based channel system (`src/channels/`)
  - ACP (Agent Control Protocol) for agent lifecycle (`src/acp/`)
  - Session management (`src/sessions/`)
  - Workspace-based memory (file system)
  - Daemon mode (launchd/systemd)
  - Docker support
  - Pairing system for mobile devices
  - Cron, hooks, context-engine, TTS, image gen
- **Key insight:** Single-user, single-agent design. Not built for multi-tenant or multi-agent orchestration.

### Pi Mono (badlogic/pi-mono)
- **What:** Tools for building AI agents and managing LLM deployments
- **Packages:**
  - `pi-ai` — Unified multi-provider LLM API (OpenAI, Anthropic, Google, etc.) with tool calling
  - `pi-agent-core` — Agent runtime with tool calling, state management, event streaming
  - `pi-coding-agent` — Interactive coding agent CLI
  - `pi-mom` — Slack bot delegating to pi coding agent (self-managing, Docker sandbox)
  - `pi-tui` — Terminal UI library
  - `pi-web-ui` — Web components for AI chat interfaces (mini-lit + Tailwind)
  - `pi-pods` — CLI for managing vLLM deployments on GPU pods (DataCrunch, RunPod, etc.)
- **Key insights:**
  - `pi-agent-core` has clean event-based architecture (agent_start, turn_start, message_start/update/end, tool calls)
  - `pi-web-ui` already has chat UI components with streaming, tool execution, artifacts
  - `pi-pods` handles GPU pod management, vLLM deployment, multi-model per pod
  - `pi-mom` is a working example of agent-as-bot with Docker sandbox

## Key Capabilities Needed (from Steins' brief)
1. **Multiagent system setup** — define, configure, deploy multiple agents
2. **Distribution** — distribute agents across infrastructure
3. **Conversation visualization** — see agent conversations, debug, monitor
4. **Clustering** — scale across multiple nodes/pods

## Technology to Research
- K8s: container orchestration, pod management, scaling, service discovery
- Docker: containerization for agent sandboxing (already used by both projects)
