# OpenClaw Architecture — Full Technical Reference & Python Migration Guide

> **Purpose**: Comprehensive architecture documentation for AI assistants and developers planning a Python/LangGraph rewrite. Includes system diagrams, code patterns, vulnerability analysis, and MVP feature plan.

---

## Table of Contents

1. [System Overview](#system-overview)
2. [High-Level Architecture Graph](#high-level-architecture-graph)
3. [Gateway Server](#gateway-server)
4. [Channel System & Routing](#channel-system--routing)
5. [Agent Orchestration](#agent-orchestration)
6. [Plugin System](#plugin-system)
7. [Media Pipeline](#media-pipeline)
8. [Configuration & State](#configuration--state)
9. [CLI Layer](#cli-layer)
10. [Supporting Subsystems](#supporting-subsystems)
11. [Security Analysis & Vulnerabilities](#security-analysis--vulnerabilities)
12. [Python Migration Notes](#python-migration-notes)
13. [MVP Feature Plan](#mvp-feature-plan)

---

## System Overview

OpenClaw is a **multi-channel AI agent gateway** (~307k LOC TypeScript). It bridges messaging platforms to LLM-powered agents through a real-time WebSocket server.

**Tech Stack (current)**:
- Runtime: Node.js 22+ / Bun (TypeScript ESM)
- Package Manager: pnpm 10.23 (monorepo)
- HTTP: Node native `http`/`https` (no Express/Fastify)
- WebSocket: `ws` library
- LLM Providers: Anthropic, OpenAI, Google, AWS Bedrock (via `@mariozechner/pi-ai`)
- Testing: Vitest (70% coverage threshold)
- Linting: Oxlint + Oxfmt
- Mobile: Swift (iOS/macOS), Kotlin (Android)

---

## High-Level Architecture Graph

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENTS                                   │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐   │
│  │ macOS App│ │ iOS App  │ │ Android  │ │ CLI (openclaw)   │   │
│  │ (SwiftUI)│ │ (Swift)  │ │ (Kotlin) │ │ (Commander)      │   │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────────┬─────────┘   │
│       │WebSocket    │WS         │WS               │WS/HTTP      │
└───────┼─────────────┼───────────┼─────────────────┼─────────────┘
        └─────────────┴───────────┴─────────────────┘
                              │
                    ┌─────────▼──────────┐
                    │   GATEWAY SERVER   │
                    │ (WebSocket + HTTP) │
                    │                    │
                    │  ┌──────────────┐  │
                    │  │ Auth Layer   │  │──── Token / Password / Tailscale
                    │  │ (3-tier)     │  │──── Device RSA signatures
                    │  └──────┬───────┘  │
                    │         │          │
                    │  ┌──────▼───────┐  │
                    │  │ RPC Router   │  │──── 60+ methods
                    │  │ (scope-based)│  │──── read/write/admin scopes
                    │  └──────┬───────┘  │
                    │         │          │
                    │  ┌──────▼───────┐  │
                    │  │ Method       │  │
                    │  │ Handlers     │  │
                    │  └──────┬───────┘  │
                    └─────────┼──────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
   ┌─────▼──────┐    ┌───────▼────────┐   ┌──────▼──────┐
   │  CHANNELS  │    │  AGENT ENGINE  │   │   PLUGINS   │
   │            │    │                │   │             │
   │ Telegram   │    │ Model Select   │   │ 32 exts     │
   │ Discord    │◄──►│ Tool Execution │   │ Hooks       │
   │ Slack      │    │ Session Mgmt   │   │ Commands    │
   │ WhatsApp   │    │ Sandbox/Sec    │   │ Providers   │
   │ Signal     │    │ Memory Search  │   │ Services    │
   │ iMessage   │    │ Skills (54)    │   │             │
   │ LINE       │    │ Subagents      │   │             │
   │ +20 more   │    │                │   │             │
   └─────┬──────┘    └───────┬────────┘   └──────┬──────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              │
                    ┌─────────▼──────────┐
                    │  SUPPORTING INFRA  │
                    │                    │
                    │ Media Pipeline     │  Image/Video/Audio processing
                    │ Link Understanding │  URL extraction + analysis
                    │ Browser (Playwright│  Headless automation
                    │ TTS (3 providers)  │  Text-to-speech
                    │ Cron Service       │  Scheduled jobs
                    │ Memory (SQLite+Vec)│  Embedding search
                    │ Logging (tslog)    │  Structured + redaction
                    │ Config (JSON5+Zod) │  Validated settings
                    └────────────────────┘
```

---

## Gateway Server

**Source**: `src/gateway/` (~100+ files)
**Entry**: `src/gateway/server.impl.ts` → `startGatewayServer(port, opts)`

### Architecture

The gateway is a standalone WebSocket + HTTP server built on Node's native `http` module (no framework). It manages client connections, authenticates them, dispatches RPC calls, and broadcasts events.

### Startup Flow

```
startGatewayServer(port=18789)
  │
  ├─ readConfigFileSnapshot() + migrateLegacyConfig()
  ├─ loadGatewayPlugins() → pluginRegistry, gatewayMethods
  ├─ resolveGatewayAuth() → token/password/tailscale config
  ├─ createGatewayHttpServer() → HTTP request handler
  ├─ attachGatewayUpgradeHandler() → WebSocket upgrade
  ├─ startChannels() → launch all enabled channel monitors
  ├─ startCronService() → scheduled jobs
  ├─ startBonjour() → mDNS discovery
  └─ return GatewayServer { close, broadcast, clients, ... }
```

### WebSocket Protocol

JSON-based frames with three types:

```
Client → Server:  { type: "req", id: "uuid", method: "send", params: {...} }
Server → Client:  { type: "res", id: "uuid", ok: true, payload: {...} }
Server → Client:  { type: "event", event: "agent.run.finished", payload: {...}, seq: N }
```

**Handshake**:
1. Server sends `connect.challenge` with nonce
2. Client sends `connect` with auth credentials + device RSA signature
3. Server validates auth, verifies device signature, returns `HelloOk` with available methods/events

### Authentication (3-tier)

```typescript
// src/gateway/auth.ts
type ResolvedGatewayAuth = {
  mode: "token" | "password";
  token?: string;
  password?: string;
  allowTailscale: boolean;
};

// Precedence: Tailscale → Token → Password
// Device auth: RSA-SHA256 signature over {deviceId, clientId, role, scopes, nonce}
// Scope-based method authorization: read / write / admin / approvals / pairing
```

### RPC Method Dispatch

```typescript
// src/gateway/server-methods.ts
// 60+ methods grouped by scope:
//   READ:  health, logs.tail, channels.status, models.list, sessions.list, ...
//   WRITE: send, agent, wake, tts.convert, chat.send, ...
//   ADMIN: config.set, channels.start, channels.stop, sessions.reset, ...

async function handleGatewayRequest(opts) {
  const authError = authorizeGatewayMethod(req.method, client);
  if (authError) { respond(false, undefined, authError); return; }
  const handler = handlers[req.method];
  await handler({ req, params, client, respond, context });
}
```

### Node Registry

Remote nodes (mobile apps, external workers) register via the connect handshake and can be invoked:

```typescript
// src/gateway/node-registry.ts
class NodeRegistry {
  register(client, opts): NodeSession
  invoke(params: { nodeId, command, params, timeoutMs }): Promise<NodeInvokeResult>
  sendEvent(nodeId, event, payload): boolean
}
```

### Hook Endpoints

```
POST /hooks/wake    → { text, mode: "now"|"next-heartbeat" }
POST /hooks/agent   → { message, channel, to, sessionKey, model, ... }
POST /hooks/{mapping} → custom mapping with Handlebars templates
```

### Python Migration Notes — Gateway

| TypeScript Pattern | Python Equivalent |
|---|---|
| Node native `http` + `ws` | **FastAPI** + **websockets** or `starlette.websockets` |
| JSON frame protocol | Same JSON protocol, use Pydantic models for validation |
| `WebSocketServer` events | FastAPI WebSocket routes with async handlers |
| Scope-based auth | FastAPI `Depends()` with role/scope checking middleware |
| `Map<connId, client>` | Python `dict[str, ClientConnection]` |
| Event broadcast | `asyncio` event bus or Redis pub/sub for multi-process |
| Device RSA signatures | `cryptography` library (RSA-SHA256) |
| Timing-safe comparison | `hmac.compare_digest()` |

**Suggested Python structure**:
```python
# gateway/
#   server.py          # FastAPI app + startup
#   protocol.py        # Pydantic models for frames
#   auth.py            # Authentication middleware
#   methods/           # RPC method handlers (one file per domain)
#     __init__.py
#     channels.py
#     agent.py
#     sessions.py
#   node_registry.py   # Remote node management
#   hooks.py           # Webhook endpoints
```

---

## Channel System & Routing

**Source**: `src/channels/`, `src/routing/`, `src/telegram/`, `src/discord/`, `src/slack/`, etc.

### Channel Plugin Interface

Every channel (core or extension) implements `ChannelPlugin`:

```typescript
// src/channels/plugins/types.core.ts
type ChannelPlugin<ResolvedAccount = any> = {
  id: ChannelId;
  meta: ChannelMeta;                    // label, docsPath, blurb, order
  capabilities: ChannelCapabilities;     // chatTypes, polls, reactions, threads, media
  config: ChannelConfigAdapter;          // listAccountIds, resolveAccount, isConfigured
  security?: ChannelSecurityAdapter;     // DM policy, allowFrom
  outbound?: ChannelOutboundAdapter;     // sendText, sendMedia, textChunkLimit
  gateway?: ChannelGatewayAdapter;       // startAccount, stopAccount
  pairing?: ChannelPairingAdapter;       // Contact pairing flow
  groups?: ChannelGroupAdapter;          // Group management
  mentions?: ChannelMentionAdapter;      // @mention detection
  // ... 10+ more adapters
};
```

### Message Lifecycle

```
INBOUND:
  Platform SDK event (grammy/discord.js/bolt/baileys)
    │
    ├─ Deduplicate (update ID tracking)
    ├─ Self-filter (ignore own messages)
    ├─ Allowlist check (user ID, name, tag matching)
    ├─ Mention gating (require @bot in groups, bypass for commands)
    ├─ Command gating (access groups + authorization)
    ├─ Route resolution (bindings → agentId + sessionKey)
    ├─ Session recording (persist sender identity, delivery context)
    ├─ Media extraction (download attachments → temp storage)
    ├─ Link extraction (URL parsing from message text)
    └─ Dispatch to agent engine

OUTBOUND:
  Agent reply (text + media)
    │
    ├─ Chunk text by platform limit (4096 Telegram, 2000 Discord, etc.)
    ├─ Format markdown for platform (sanitize unsupported syntax)
    ├─ Send via channel outbound adapter
    ├─ TTS generation (if auto-mode enabled)
    ├─ Streaming (partial replies for supported channels)
    └─ Session update (record delivery context for reply routing)
```

### Routing via Bindings

```typescript
// src/routing/resolve-route.ts
// Bindings map (channel, accountId, peer?) → agentId
// Precedence: peer > parent > guild > team > account > channel > default

function resolveAgentRoute(input): ResolvedAgentRoute {
  // Returns: { agentId, sessionKey, matchedBy }
}
```

### Security Layers

```
1. Allowlist (per-channel config: allowFrom IDs/names/tags)
   ↓ drop if not in list
2. Mention gating (require @mention in groups; DMs bypass)
   ↓ skip if not mentioned
3. Command gating (access groups restrict /commands)
   ↓ block unauthorized commands, bypass mention for authorized ones
4. Route resolution (which agent handles this session)
```

### Python Migration Notes — Channels

| TypeScript Pattern | Python Equivalent |
|---|---|
| grammy (Telegram) | **python-telegram-bot** or **aiogram** |
| discord.js | **discord.py** or **nextcord** |
| @slack/bolt | **slack-bolt** for Python |
| Baileys (WhatsApp) | **whatsapp-web.js** via subprocess, or Baileys via Node sidecar |
| ChannelPlugin interface | Python `Protocol` or ABC class |
| AbortController for lifecycle | `asyncio.Event` or `asyncio.CancelledError` |
| Channel docks (lightweight) | Separate dataclass for capabilities/metadata |

**Suggested Python structure**:
```python
# channels/
#   base.py           # ChannelPlugin Protocol/ABC
#   registry.py       # Plugin registration + listing
#   routing.py        # Binding-based agent routing
#   security.py       # Allowlist, mention gating, command gating
#   telegram/
#     adapter.py      # ChannelPlugin implementation
#     bot.py          # aiogram/python-telegram-bot wiring
#   discord/
#     adapter.py
#     bot.py
#   slack/
#     adapter.py
#     bot.py
```

---

## Agent Orchestration

**Source**: `src/agents/` (~30+ files)

### Execution Pipeline

```
User message arrives
  │
  ├─ Lane serialization (per-session queue)
  │
  ├─ SETUP PHASE
  │   ├─ Resolve model + provider (with alias expansion)
  │   ├─ Resolve auth profile (API key rotation)
  │   ├─ Context window guard (minimum token check)
  │   ├─ Load skills (scan workspace/skills/, build prompt)
  │   ├─ Resolve sandbox context (Docker container if enabled)
  │   └─ Build system prompt (full/minimal/none mode)
  │
  ├─ ATTEMPT LOOP (with failover)
  │   ├─ Build tool definitions (SDK + channel + plugin tools)
  │   ├─ Call LLM via streamSimple() (@mariozechner/pi-ai)
  │   ├─ Subscribe to events (text_delta, tool_use, message_end)
  │   ├─ Execute tools as requested by LLM
  │   ├─ Stream block replies back to channel
  │   │
  │   └─ ON FAILURE:
  │       ├─ Classify: auth | billing | rate_limit | timeout | context_overflow
  │       ├─ Rotate auth profile or switch to fallback model
  │       └─ Retry (up to fallback chain exhaustion)
  │
  └─ RESULT
      ├─ Validate session transcript
      ├─ Apply compaction if needed
      └─ Return usage metrics
```

### Tool System

```
Tool Categories:
  ├─ Coding: read, write, edit, exec, process (from pi-coding-agent)
  ├─ Native: browser, canvas, message, tts, gateway, agents_list,
  │          sessions_list, sessions_spawn, web_search, web_fetch, image
  ├─ Channel: per-channel tools (discord dm_send, slack thread_reply, etc.)
  ├─ Plugin: registered via api.registerTool()
  └─ Client: delegated to connected client (mobile app actions)

Tool Definition:
  {
    name: string,
    description: string,
    parameters: JSONSchema,
    execute(toolCallId, params, signal, onUpdate): Promise<AgentToolResult>
  }

Tool Policy (allowlist/denylist):
  - Subagents denied: sessions_spawn, gateway, agents_list (prevent cycles)
  - Sandbox: custom allow/deny per agent config
```

### Model Selection & Failover

```typescript
// Model reference: "provider/model" or alias
// E.g.: "anthropic/claude-opus-4-5", "openai/gpt-4o", "opus-4.5" (alias)

// Failover chain:
//   1. Primary model with preferred auth profile
//   2. Same model, rotated auth profile
//   3. Fallback model #1 (from config.agents.defaults.model.fallbacks)
//   4. Fallback model #2
//   ...
//   N. Image-specific fallback (if vision error)
```

### Subagent Orchestration

```
Parent agent calls sessions_spawn(task, sessionKey)
  │
  ├─ Create child session with isolated sessionKey
  ├─ Apply subagent tool policy (deny admin tools)
  ├─ Run child agent with "minimal" system prompt
  ├─ On completion:
  │   ├─ Read child's final reply
  │   ├─ Build announcement prompt
  │   ├─ Run parent agent with child results
  │   └─ Optionally delete child session
  └─ Persisted in ~/.openclaw/agents/{id}/subagent-registry.json
```

### Memory System

```
SQLite + sqlite-vec (vector embeddings)
  │
  ├─ Sources: MEMORY.md, memory/*.md, session transcripts
  ├─ Chunking: 400 tokens, 80 overlap
  ├─ Embedding: text-embedding-3-small (OpenAI) or local
  ├─ Search: hybrid BM25 (0.3) + vector (0.7)
  ├─ Results: top 6, min score 0.35
  └─ Tools: memory_search(query) → results, memory_get(path, lines) → text
```

### Python Migration Notes — Agents

| TypeScript Pattern | Python Equivalent |
|---|---|
| `@mariozechner/pi-ai` streaming | **LangChain** `ChatModel.stream()` or **LiteLLM** |
| Tool definitions (JSON Schema) | **LangChain Tools** or **LangGraph** tool nodes |
| Multi-stage pipeline | **LangGraph** `StateGraph` with conditional edges |
| Session lane serialization | `asyncio.Lock` per session key |
| Model failover chain | LangChain fallback chains or custom retry logic |
| Subagent spawning | LangGraph subgraph invocation |
| Memory (SQLite + vec) | **ChromaDB**, **LanceDB**, or **pgvector** |
| Skills (SKILL.md files) | LangChain `StructuredTool` loaded from YAML/MD |

**Suggested Python structure (LangGraph)**:
```python
# agents/
#   graph.py           # LangGraph StateGraph definition
#   state.py           # AgentState TypedDict
#   nodes/
#     setup.py         # Model resolution, system prompt
#     llm_call.py      # LLM invocation node
#     tool_router.py   # Tool dispatch node
#     failover.py      # Error handling + model rotation
#   tools/
#     base.py          # Tool protocol
#     coding.py        # read, write, edit, exec
#     messaging.py     # send, tts
#     browser.py       # Playwright tools
#     memory.py        # memory_search, memory_get
#   models/
#     selection.py     # Model alias resolution
#     auth.py          # API key rotation
#     providers.py     # LiteLLM or LangChain model wrappers
#   skills/
#     loader.py        # Scan skills directory
#     prompt.py        # Build skills system prompt section
#   memory/
#     store.py         # Vector store (ChromaDB/LanceDB)
#     search.py        # Hybrid search implementation
```

**LangGraph Agent Graph**:
```python
from langgraph.graph import StateGraph, END

class AgentState(TypedDict):
    messages: list[BaseMessage]
    session_key: str
    model_ref: ModelRef
    tools: list[BaseTool]
    attempt: int
    max_attempts: int

graph = StateGraph(AgentState)
graph.add_node("setup", setup_node)
graph.add_node("llm_call", llm_call_node)
graph.add_node("tool_execute", tool_execute_node)
graph.add_node("failover", failover_node)
graph.add_node("respond", respond_node)

graph.set_entry_point("setup")
graph.add_edge("setup", "llm_call")
graph.add_conditional_edges("llm_call", route_after_llm, {
    "tool_call": "tool_execute",
    "final_answer": "respond",
    "error": "failover",
})
graph.add_edge("tool_execute", "llm_call")
graph.add_conditional_edges("failover", check_retries, {
    "retry": "llm_call",
    "exhausted": "respond",
})
graph.add_edge("respond", END)

agent = graph.compile()
```

---

## Plugin System

**Source**: `src/plugins/`, `src/plugin-sdk/`, `extensions/`

### Plugin Lifecycle

```
DISCOVERY → MANIFEST → LOAD → VALIDATE → REGISTER → ACTIVATE → RUNTIME
    │            │        │        │          │           │         │
    │  Scan dirs │ Read   │ Jiti   │ JSON     │ Call      │ Call    │ Hooks
    │  for       │ plugin │ import │ Schema   │ register  │ activate│ fire
    │  plugins   │ .json  │ module │ check    │ (api)     │ (api)   │
```

### Plugin API

```typescript
// Plugins receive OpenClawPluginApi with:
api.registerTool(tool)           // Add LLM-callable tools
api.registerHook(events, handler) // Lifecycle hooks (14 events)
api.registerChannel(plugin)       // New messaging channel
api.registerGatewayMethod(name, handler) // Custom RPC methods
api.registerHttpRoute({ path, handler }) // HTTP endpoints
api.registerService(service)      // Background services
api.registerProvider(provider)    // LLM provider
api.registerCommand(command)      // Slash commands
api.registerCli(registrar)        // CLI subcommands
```

### Hook Events

```
before_agent_start, agent_end
before_compaction, after_compaction
message_received, message_sending, message_sent
before_tool_call, after_tool_call, tool_result_persist
session_start, session_end
gateway_start, gateway_stop
```

### Python Migration Notes — Plugins

| TypeScript Pattern | Python Equivalent |
|---|---|
| Jiti (TypeScript loader) | `importlib` + `pluggy` or `stevedore` |
| JSON Schema config validation | Pydantic models |
| Plugin registry (Map) | Python dict + dataclass registry |
| Hook runner (priority-sorted) | `pluggy` hook system or custom event emitter |
| openclaw.plugin.json manifest | `pyproject.toml` `[tool.openclaw]` section or YAML |

**Suggested Python structure**:
```python
# plugins/
#   base.py        # PluginAPI protocol, PluginDefinition
#   loader.py      # Discovery + importlib loading
#   registry.py    # Plugin registry + hook runner
#   hooks.py       # Event definitions + dispatch
#   manifest.py    # Manifest validation (Pydantic)
```

---

## Media Pipeline

**Source**: `src/media/`, `src/media-understanding/`, `src/link-understanding/`, `src/browser/`

### Media Flow

```
Remote URL / Platform attachment
  │
  ├─ SSRF check (block private IPs, metadata endpoints)
  ├─ Download with size limit (5MB default)
  ├─ MIME sniffing (first 16KB)
  ├─ Sanitize filename (Unicode-safe)
  ├─ Store to ~/.openclaw/media/ (0o600 perms)
  │
  ├─ UNDERSTANDING (optional)
  │   ├─ Audio: Whisper (OpenAI), Deepgram, or ElevenLabs
  │   ├─ Image: GPT-4V, Claude Vision, or Gemini
  │   ├─ Video: Frame extraction + vision model
  │   └─ Links: Playwright fetch + extraction
  │
  └─ Auto-cleanup (TTL: 2 minutes default)
```

### Browser Automation

- Playwright CDP connection (local Chrome or remote Browserless)
- REST API for AI interaction: `/tabs/{id}/snapshot`, `/tabs/{id}/act`, `/tabs/{id}/screenshot`
- Accessibility tree snapshots for LLM consumption

### TTS (Text-to-Speech)

```
Providers: Edge TTS (free), OpenAI TTS, ElevenLabs
Modes: "off" | "always" | "inbound" (reply to audio) | "tagged" ([[tts]] directive)
Auto-summarization for long texts (>1500 chars)
Channel-specific formats: Opus@48kHz (Telegram), MP3@128k (default)
```

### Python Migration Notes — Media

| TypeScript Pattern | Python Equivalent |
|---|---|
| `sharp` (image processing) | **Pillow** or **wand** |
| Playwright (browser) | **playwright** for Python (official) |
| Edge TTS | **edge-tts** Python package |
| OpenAI Whisper | `openai` Python SDK |
| SSRF protection | Custom `urllib3` adapter or `ssrf-king` |

---

## Configuration & State

**Source**: `src/config/`

### Config Loading Pipeline

```
1. Resolve path: OPENCLAW_CONFIG_PATH → ~/.openclaw/config.json
2. Parse: JSON5 (with comments)
3. Includes: @import directives
4. Env substitution: ${VARIABLE} replacement
5. Path normalization: resolve ~/ and relative paths
6. Runtime overrides: CLI flags
7. Schema validation: Zod
8. Defaults: apply agent defaults
```

### State Locations

```
~/.openclaw/
  ├─ config.json          # Main config (JSON5)
  ├─ config.json.bak*     # 5 rolling backups
  ├─ credentials/         # Web provider auth
  ├─ sessions/            # JSONL session transcripts
  ├─ agents/{id}/         # Per-agent state
  │   ├─ workspace/       # Agent workspace files
  │   ├─ sessions/        # Agent session logs
  │   ├─ exec-approvals.json
  │   └─ subagent-registry.json
  ├─ media/               # Temporary media storage
  ├─ memory/              # SQLite vector stores
  ├─ skills/              # Installed skills
  ├─ cron.json            # Scheduled jobs
  ├─ auth-profiles.json   # API key profiles
  └─ settings/            # User preferences (tts.json, etc.)
```

### Python Migration Notes — Config

| TypeScript Pattern | Python Equivalent |
|---|---|
| JSON5 + Zod | **Pydantic** `BaseSettings` with JSON/YAML |
| Config file watching | `watchdog` library |
| Rolling backups | `shutil.copy2` + rotation |
| JSONL sessions | Same format, or switch to SQLite |

---

## CLI Layer

**Source**: `src/cli/`, `src/commands/`

### Structure

```
Commander.js program
  │
  ├─ ~40 subcommands
  ├─ Route-based fast paths (health, status, sessions → skip plugin load)
  ├─ Plugin CLI extensions (api.registerCli)
  └─ Dep injection via createDefaultDeps()
```

### Python Migration Notes — CLI

| TypeScript Pattern | Python Equivalent |
|---|---|
| Commander.js | **Click** or **Typer** |
| Route-based fast paths | Click groups with lazy loading |
| createDefaultDeps() DI | Dependency injection via Click context |

---

## Supporting Subsystems

### Cron Service (`src/cron/`)
- Schedule types: one-shot, interval, cron expression
- Payloads: system event (wake agent) or agent turn (LLM call)
- Persisted to `~/.openclaw/cron.json`

### Process Management (`src/process/`)
- Lane-based command serialization (prevents interleaving)
- Child process bridge with signal forwarding
- Execution with timeout + max buffer

### Logging (`src/logging/`)
- File-based: `/tmp/openclaw/openclaw-YYYY-MM-DD.log` (JSON, 24hr rotation)
- Credential redaction (18+ char tokens, PEM blocks, API keys)
- External transports (plugins can register log sinks)

### Auto-Reply (`src/auto-reply/`)
- ~50+ slash commands (`/model`, `/reset`, `/compact`, `/approve`, `/bash`, etc.)
- Inline directives: `[[elevated:true]]`, `[[think:high]]`, `[[tts:text]]`
- Reply envelope formatting (timestamp, sender, channel context)

---

## Security Analysis & Vulnerabilities

### Current Security Measures

| Area | Implementation | Status |
|---|---|---|
| **Auth** | Token + Password + Tailscale, RSA device signatures | Good |
| **SSRF** | DNS pinning, private IP blocking, hostname blocklist | Good |
| **File permissions** | 0o600 files, 0o700 dirs, audit checks | Good |
| **Credential redaction** | 18 regex patterns in logs | Good |
| **Tool sandboxing** | Docker containers, allow/deny lists | Good |
| **External content** | XML boundary markers, pattern detection | Moderate |
| **Rate limiting** | API throttler (Telegram), dedup maps | Partial |

### Identified Vulnerabilities & Risks

#### HIGH PRIORITY

1. **Shell Injection via Agent Tools**
   - `exec` tool runs shell commands from LLM output
   - Mitigation exists (approval system, allowlists) but relies on user config
   - **Python note**: Use `subprocess.run(args_list)` (not `shell=True`), enforce allowlists

2. **Prompt Injection via External Content**
   - Webhook payloads, email hooks, and link-understanding content flow into agent prompts
   - Current mitigation: XML boundary markers + suspicious pattern regex
   - **Python note**: Use LangChain's content filtering or custom sanitization layer

3. **Auth Profile Rotation Without Lockout**
   - Failed auth profiles get cooldown but no permanent lockout
   - Attacker with one leaked key could trigger rotation to discover other profiles
   - **Python note**: Implement exponential backoff with max attempts + alerting

#### MEDIUM PRIORITY

4. **WebSocket Denial of Service**
   - No connection rate limiting or per-IP limits in gateway
   - Large frame payloads could exhaust memory
   - **Python note**: Use FastAPI middleware for rate limiting, enforce max frame size

5. **JSONL Session File Tampering**
   - Sessions stored as plain JSONL files with no integrity verification
   - Compromised workspace could inject false conversation history
   - **Python note**: Add HMAC signatures to session entries

6. **Plugin Code Execution**
   - Plugins run in the same process with full Node.js access
   - No sandboxing or capability restriction for plugin code
   - **Python note**: Consider `RestrictedPython` or subprocess isolation for untrusted plugins

7. **Hook Template Injection**
   - Handlebars templates in hook mappings render user-controlled payload data
   - Could allow server-side template injection if payload is crafted
   - **Python note**: Use Jinja2 with `SandboxedEnvironment`

#### LOW PRIORITY

8. **Media TTL Race Condition**
   - 2-minute TTL cleanup could delete media being actively processed
   - **Python note**: Reference counting or lock files

9. **Log File World-Readable**
   - Logs in `/tmp/openclaw/` may be readable by other users
   - **Python note**: Use `tempfile.mkdtemp()` with restricted permissions

10. **Browser CDP Unauthenticated**
    - Local CDP connection has no auth by default
    - **Python note**: Bind to loopback only, add token auth

### Security Checklist for Python MVP

- [ ] All subprocess calls use `args` list (never `shell=True`)
- [ ] SSRF protection on all outbound HTTP (private IP + hostname checks)
- [ ] Input validation with Pydantic on all API boundaries
- [ ] Rate limiting on WebSocket connections and HTTP endpoints
- [ ] Credential redaction in all log output
- [ ] File permissions 0o600/0o700 for state directories
- [ ] Session integrity (HMAC or signatures)
- [ ] External content sanitization before agent prompt injection
- [ ] Tool execution sandboxing (Docker or subprocess isolation)
- [ ] API key storage encryption at rest

---

## Python Migration Notes

### Technology Mapping

| OpenClaw (TypeScript) | Python Equivalent | Notes |
|---|---|---|
| Node.js 22 | Python 3.12+ | asyncio for concurrency |
| TypeScript ESM | Python type hints + mypy | Use `Protocol` for interfaces |
| pnpm monorepo | **uv** or **poetry** workspaces | uv is fastest |
| Commander.js | **Typer** or **Click** | Typer for type-based CLI |
| ws (WebSocket) | **FastAPI** WebSockets | Built-in async support |
| Node http | **FastAPI** / **uvicorn** | ASGI async server |
| Vitest | **pytest** + **pytest-asyncio** | |
| Oxlint | **ruff** | Fast Rust-based linter |
| Zod | **Pydantic v2** | |
| JSON5 | **pyjson5** or YAML | Consider YAML/TOML for config |
| @mariozechner/pi-ai | **LiteLLM** | Unified LLM provider API |
| LLM streaming | **LangChain** streaming | Or LiteLLM stream |
| Tool definitions | **LangChain Tools** | JSON Schema compatible |
| Agent orchestration | **LangGraph** | State machine agent |
| Plugin system | **pluggy** | pytest's plugin framework |
| SQLite + sqlite-vec | **ChromaDB** or **LanceDB** | Or pgvector for production |
| sharp (images) | **Pillow** | |
| Playwright | **playwright** (Python) | Official port |
| edge-tts | **edge-tts** (Python) | Same package name |
| Baileys (WhatsApp) | Node sidecar or **yowsup** | WhatsApp is hardest to port |
| grammy (Telegram) | **aiogram** or **python-telegram-bot** | |
| discord.js | **discord.py** | |
| @slack/bolt | **slack-bolt** (Python) | Official port |

### Suggested Project Structure

```
openclaw-python/
├── pyproject.toml          # uv/poetry config
├── openclaw/
│   ├── __init__.py
│   ├── cli/                # Typer CLI application
│   │   ├── __init__.py
│   │   ├── main.py
│   │   └── commands/       # One file per command group
│   ├── gateway/            # FastAPI WebSocket server
│   │   ├── __init__.py
│   │   ├── server.py       # FastAPI app
│   │   ├── protocol.py     # Pydantic frame models
│   │   ├── auth.py         # Auth middleware
│   │   ├── methods/        # RPC handlers
│   │   └── hooks.py        # Webhook endpoints
│   ├── channels/           # Channel adapters
│   │   ├── base.py         # ChannelPlugin Protocol
│   │   ├── registry.py
│   │   ├── routing.py
│   │   ├── security.py
│   │   ├── telegram/
│   │   ├── discord/
│   │   └── slack/
│   ├── agents/             # LangGraph agent
│   │   ├── graph.py        # StateGraph definition
│   │   ├── state.py        # Agent state
│   │   ├── nodes/          # Graph nodes
│   │   ├── tools/          # Tool implementations
│   │   ├── models/         # Provider wrappers
│   │   ├── memory/         # Vector store
│   │   └── skills/         # Skill loader
│   ├── plugins/            # Plugin system (pluggy)
│   │   ├── base.py
│   │   ├── loader.py
│   │   └── hooks.py
│   ├── media/              # Media processing
│   ├── config/             # Pydantic settings
│   ├── security/           # SSRF, sanitization, audit
│   └── logging/            # Structured logging
├── plugins/                # Extension packages
├── skills/                 # Agent skills
├── tests/                  # pytest
└── docs/
```

---

## MVP Feature Plan

### Phase 1: Core Gateway (Week 1-2)

**Goal**: Standalone WebSocket server that clients can connect to.

- [ ] **FastAPI WebSocket server** with JSON frame protocol
- [ ] **Authentication**: token-based auth (skip Tailscale/device RSA for MVP)
- [ ] **RPC dispatch**: health, status, config.get (3-5 read methods)
- [ ] **Config loading**: Pydantic model from JSON/YAML file
- [ ] **Logging**: structlog with credential redaction

**Deliverable**: A running server that macOS/CLI clients can connect to and get health status.

### Phase 2: Single Channel + Agent (Week 3-4)

**Goal**: Process messages from one channel through an AI agent.

- [ ] **Telegram adapter** (simplest channel, good SDK support)
  - Receive messages, allowlist check, send replies
- [ ] **LangGraph agent** with basic tool set
  - LLM call via LiteLLM (Anthropic + OpenAI)
  - Text response streaming
  - Model selection (single model, no failover yet)
- [ ] **Session storage** (SQLite, basic CRUD)
- [ ] **System prompt** builder (identity, date, channel info)
- [ ] **Basic CLI** (Typer): `openclaw gateway run`, `openclaw config set`

**Deliverable**: Send a Telegram message, get an AI response.

### Phase 3: Multi-Channel + Tools (Week 5-6)

**Goal**: Support multiple channels and agent tools.

- [ ] **Discord adapter**
- [ ] **Slack adapter**
- [ ] **Tool framework**: read, write, exec (sandboxed subprocess)
- [ ] **Model failover** chain (primary → fallback)
- [ ] **Auth profile rotation** (multiple API keys)
- [ ] **Message routing** (bindings → agent selection)
- [ ] **Webhook hooks** (`/hooks/agent`, `/hooks/wake`)

**Deliverable**: Multi-channel bot with tool use and model failover.

### Phase 4: Plugin System + Memory (Week 7-8)

**Goal**: Extensibility and persistence.

- [ ] **Plugin loader** (pluggy-based, directory scanning)
- [ ] **Plugin API**: registerTool, registerHook, registerChannel
- [ ] **Memory system**: ChromaDB/LanceDB vector store
  - memory_search, memory_get tools
  - Auto-indexing of MEMORY.md files
- [ ] **Cron service** (APScheduler)
- [ ] **TTS** (edge-tts, OpenAI)
- [ ] **Media pipeline** (download, MIME detect, image/audio understanding)

**Deliverable**: Extensible platform with memory and scheduled tasks.

### Phase 5: Security Hardening + Polish (Week 9-10)

**Goal**: Production-ready security and UX.

- [ ] **SSRF protection** on all outbound HTTP
- [ ] **Tool sandboxing** (Docker subprocess isolation)
- [ ] **Rate limiting** (WebSocket + HTTP)
- [ ] **Security audit** command
- [ ] **Session compaction** (token limit management)
- [ ] **Subagent orchestration** (LangGraph subgraphs)
- [ ] **Full CLI** (~20 essential commands)
- [ ] **Test suite** (pytest, 70% coverage target)

**Deliverable**: Hardened, tested platform ready for beta users.

### MVP vs Full Feature Comparison

| Feature | MVP (Phase 1-3) | Full (Phase 4-5) | Original |
|---|---|---|---|
| Channels | Telegram + Discord + Slack | + plugins | 30+ |
| Models | 2-3 providers | Failover chain | All providers |
| Tools | read, write, exec | + browser, memory, TTS | 25+ tools |
| Plugins | None | pluggy-based | 32 extensions |
| Memory | None | Vector search | SQLite + vec |
| Security | Basic auth | Full audit | 40 checks |
| Mobile apps | None | None (future) | iOS + Android + macOS |
| Sessions | SQLite | + compaction | JSONL |
| Skills | None | File-based | 54 skills |

### Critical Path Items

1. **WhatsApp**: Baileys (Node.js only) has no Python equivalent. Options:
   - Run Baileys as a Node sidecar process with IPC
   - Use WhatsApp Business API (official, but different capabilities)
   - Skip for MVP

2. **Signal**: Signal protocol requires specific crypto. Options:
   - Use `signal-cli` as subprocess
   - Skip for MVP

3. **iMessage**: macOS-only, requires AppleScript bridge. Options:
   - Keep as macOS-only feature
   - Skip for MVP

4. **Mobile Apps**: Swift/Kotlin apps communicate via WebSocket. As long as the gateway protocol is compatible, existing apps should work with the Python backend.

---

## Appendix: File Reference

| Subsystem | Key Files |
|---|---|
| Gateway entry | `src/gateway/server.impl.ts` |
| HTTP server | `src/gateway/server-http.ts` |
| WebSocket handler | `src/gateway/server/ws-connection.ts` |
| Protocol | `src/gateway/protocol/index.ts` |
| Auth | `src/gateway/auth.ts` |
| RPC methods | `src/gateway/server-methods.ts` |
| Node registry | `src/gateway/node-registry.ts` |
| Hooks | `src/gateway/hooks.ts`, `hooks-mapping.ts` |
| Channel types | `src/channels/plugins/types.core.ts` |
| Routing | `src/routing/resolve-route.ts` |
| Allowlists | `src/channels/allowlist-match.ts` |
| Mention gating | `src/channels/mention-gating.ts` |
| Agent runner | `src/agents/pi-embedded-runner/run.ts` |
| Tools | `src/agents/pi-tools.ts`, `openclaw-tools.ts` |
| Model selection | `src/agents/model-selection.ts` |
| Auth profiles | `src/agents/model-auth.ts` |
| Sandbox | `src/agents/sandbox/context.ts` |
| Skills | `src/agents/skills.ts` |
| Memory | `src/memory/`, `src/agents/memory-search.ts` |
| Plugin loader | `src/plugins/loader.ts` |
| Plugin registry | `src/plugins/registry.ts` |
| Plugin API | `src/plugins/types.ts` |
| Config | `src/config/config.ts`, `schema.ts` |
| CLI | `src/cli/program/build-program.ts` |
| Media | `src/media/`, `src/media-understanding/` |
| Browser | `src/browser/` |
| TTS | `src/tts/` |
| Security | `src/security/`, `src/infra/net/ssrf.ts` |
| Logging | `src/logging/` |
| Cron | `src/cron/` |
| Process mgmt | `src/process/` |
