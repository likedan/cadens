# Cadens Design Document

A deep dive into the architecture, design decisions, and production-hardened systems that power Cadens.

---

## Table of Contents

- [System Overview](#system-overview)
- [The ReAct Loop](#the-react-loop)
- [Streaming Pipeline](#streaming-pipeline)
- [Sandbox Execution](#sandbox-execution)
- [Context Management](#context-management)
- [Multi-Provider LLM Integration](#multi-provider-llm-integration)
- [Sub-Agent System](#sub-agent-system)
- [Security Model](#security-model)
- [Resilience & Fault Tolerance](#resilience--fault-tolerance)
- [Skills System](#skills-system)
- [Artifact Rendering](#artifact-rendering)
- [Database Architecture](#database-architecture)
- [Design Decisions & Trade-offs](#design-decisions--trade-offs)

---

## System Overview

Cadens is a four-service architecture with strict separation of concerns:

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              USERS                                      │
│                    Web Browser (any device)                              │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │ HTTPS
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                     Frontend (Next.js 15)                     :3000     │
│                                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────┐  ┌───────────────┐  │
│  │ Chat UI     │  │ Artifact     │  │ File      │  │ Settings &    │  │
│  │ Streaming   │  │ Renderer     │  │ Manager   │  │ Auth UI       │  │
│  │ Messages    │  │ (HTML/React/ │  │ Upload/   │  │               │  │
│  │ Typewriter  │  │  Charts/SVG) │  │ Download  │  │               │  │
│  └─────────────┘  └──────────────┘  └───────────┘  └───────────────┘  │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │ REST + SSE
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                     Backend (FastAPI)                          :8000     │
│                                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────┐  ┌───────────────┐  │
│  │ Auth        │  │ Session      │  │ File      │  │ Chat Proxy    │  │
│  │ JWT +       │  │ Management   │  │ Storage   │  │ to Brain      │  │
│  │ OAuth       │  │              │  │ (S3)      │  │ (SSE relay)   │  │
│  └─────────────┘  └──────────────┘  └───────────┘  └───────────────┘  │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │ HTTP (internal)
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                     Brain (FastAPI)                            :9000     │
│                                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────┐  ┌───────────────┐  │
│  │ ReAct Loop  │  │ Tool         │  │ Sub-Agent │  │ LLM Provider  │  │
│  │ Reason →    │  │ Registry &   │  │ Spawner   │  │ Manager       │  │
│  │ Act →       │  │ Executor     │  │ (Task     │  │ (Anthropic/   │  │
│  │ Observe     │  │              │  │  tool)    │  │  OpenAI/      │  │
│  │             │  │              │  │           │  │  Gemini)      │  │
│  └─────────────┘  └──────────────┘  └───────────┘  └───────────────┘  │
│                                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────┐  ┌───────────────┐  │
│  │ SSE Stream  │  │ Context      │  │ Langfuse  │  │ Message       │  │
│  │ Background  │  │ Compaction   │  │ Tracing   │  │ History       │  │
│  │ Runner      │  │ Engine       │  │           │  │ Manager       │  │
│  └─────────────┘  └──────────────┘  └───────────┘  └───────────────┘  │
└──────────┬───────────────────────────────────┬───────────────────────────┘
           │ HTTP                              │ MCP (Streamable HTTP)
           ▼                                   ▼
┌────────────────────┐              ┌──────────────────────────────────────┐
│ Crawl4AI    :11235 │              │ Executor Service                :9100│
│                    │              │                                      │
│ Web search,        │              │ ┌──────────────┐  ┌──────────────┐  │
│ page scraping,     │              │ │ MCP Server   │  │ Sandbox      │  │
│ content extraction │              │ │ (FastMCP)    │  │ Manager      │  │
└────────────────────┘              │ └──────────────┘  └──────────────┘  │
                                    │ ┌──────────────┐  ┌──────────────┐  │
                                    │ │ Bash Tool    │  │ File Tools   │  │
                                    │ │ (Python,     │  │ (Read/Write/ │  │
                                    │ │  Node.js)    │  │  Edit/Glob)  │  │
                                    │ └──────────────┘  └──────────────┘  │
                                    │ ┌──────────────┐  ┌──────────────┐  │
                                    │ │ Session      │  │ UID          │  │
                                    │ │ Workspace    │  │ Isolation    │  │
                                    │ │ (/mnt/       │  │ (per-session │  │
                                    │ │  workspace/) │  │  OS users)   │  │
                                    │ └──────────────┘  └──────────────┘  │
                                    └──────────────────────────────────────┘
```

### Why four services?

| Service | Responsibility | Why separate? |
|---|---|---|
| **Frontend** | User interface, rendering | Can be replaced entirely (mobile app, CLI, different framework) |
| **Backend** | Auth, sessions, file storage, routing | Handles fast CRUD operations; doesn't need GPU/LLM resources |
| **Brain** | AI reasoning, tool orchestration | Compute-heavy (LLM calls, long-running ReAct loops); scales independently |
| **Executor** | Code execution, file operations | Highest-risk component (runs untrusted code); isolated for security |

Each service owns specific database collections with a single-writer pattern -- no write conflicts, ever.

---

## The ReAct Loop

The Brain runs a ReAct (Reasoning + Acting) loop that powers all agent behavior:

```
                    ┌─────────────────────┐
                    │   User Message      │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   REASON            │
                    │   LLM analyzes the  │
                    │   message + context  │
                    │   + tool results     │
                    └──────────┬──────────┘
                               │
                        ┌──────▼──────┐
                        │  Response?  │
                        └──┬──────┬───┘
                    Tool   │      │  Text
                    Call   │      │  Response
                           │      │
              ┌────────────▼─┐  ┌─▼─────────────┐
              │   ACT        │  │   DONE         │
              │   Execute    │  │   Stream final  │
              │   tool(s)    │  │   response to   │
              │              │  │   user          │
              └────────┬─────┘  └─────────────────┘
                       │
              ┌────────▼─────┐
              │   OBSERVE    │
              │   Collect    │
              │   tool       │
              │   results    │
              └────────┬─────┘
                       │
                       └──── Loop back to REASON
```

### Key implementation details

**Parallel tool execution** -- When the LLM requests multiple tool calls in a single turn, they execute concurrently (not sequentially). This dramatically speeds up multi-tool operations.

**Tool result injection** -- Tool results are injected back into the conversation as structured `tool_result` messages that the LLM can reason about in the next turn.

**Graceful termination** -- The loop has configurable maximum iterations to prevent runaway agents. If the agent exceeds the limit, it generates a summary of progress and asks for guidance.

**Streaming during reasoning** -- Token-by-token streaming begins immediately as the LLM starts generating, so users see thinking in real-time.

---

## Streaming Pipeline

Most agent frameworks treat streaming as an afterthought. Cadens has a full production-grade SSE (Server-Sent Events) pipeline:

```
Brain (ReAct Loop)
    │
    │ Tokens generated
    ▼
┌───────────────────────┐
│ Background Runner     │   ← Decouples HTTP from agent execution
│                       │      Agent keeps running even if HTTP disconnects
│ ┌───────────────────┐ │
│ │ Event Buffer      │ │   ← Buffers events for reconnecting clients
│ │ (missed events)   │ │
│ └────────┬──────────┘ │
│          │            │
│ ┌────────▼──────────┐ │
│ │ Stream Registry   │ │   ← Maps session_id → active stream
│ │ (reconnection)    │ │      Clients reconnect to same stream
│ └────────┬──────────┘ │
│          │            │
│ ┌────────▼──────────┐ │
│ │ Keep-Alive        │ │   ← 15-second heartbeat prevents
│ │ Heartbeat         │ │      ALB/proxy idle timeout (60s)
│ └───────────────────┘ │
└───────────┬───────────┘
            │ SSE
            ▼
Backend (relay)
            │ SSE
            ▼
Frontend (EventSource)
            │
            ▼
┌───────────────────────┐
│ useSmoothStream       │   ← rAF-based typewriter effect
│ (requestAnimationFrame│      Smooth token rendering
│  batched rendering)   │
└───────────────────────┘
```

### Why this matters

**Problem:** User is on a flaky mobile connection. They disconnect for 3 seconds during a long agent response.

**Without Cadens:** The stream dies. The agent's work is lost. User has to re-send their message.

**With Cadens:** The background runner keeps the agent running. When the client reconnects, the stream registry routes them back. The event buffer replays missed tokens. The user sees a seamless continuation.

### Event types streamed

| Event | Data | Purpose |
|---|---|---|
| `token` | Text chunk | Streaming text response |
| `tool_call` | Tool name + args | Show user what the agent is doing |
| `tool_result` | Output | Show tool execution results |
| `thinking` | Reasoning text | Extended thinking (Anthropic) |
| `status` | State change | Loading, executing, done |
| `error` | Error details | Graceful error display |
| `heartbeat` | Timestamp | Keep connection alive |

---

## Sandbox Execution

The Executor service runs untrusted code in isolated sandboxes. This is the most security-critical component.

```
┌──────────────────────────────────────────────────────────────┐
│                    /mnt/workspace/                            │
│                                                              │
│  ┌─────────────────────┐   ┌─────────────────────┐          │
│  │ session_abc/         │   │ session_xyz/         │          │
│  │ (UID 10001)          │   │ (UID 10002)          │          │
│  │  ├── uploads/        │   │  ├── uploads/        │          │
│  │  ├── home/           │   │  ├── home/           │          │
│  │  ├── outputs/        │   │  ├── outputs/        │          │
│  │  └── .venv → shared/ │   │  └── .venv → shared/ │          │
│  │                      │   │                      │          │
│  │  Permissions: 700    │   │  Permissions: 700    │          │
│  │  Owner: UID 10001    │   │  Owner: UID 10002    │          │
│  └──────────┬───────────┘   └──────────┬───────────┘          │
│             │                          │                      │
│             ╳ ◄── OS-LEVEL BLOCK ──► ╳                       │
│             (cross-session access denied)                     │
│                                                              │
│  ┌──────────────────────────────────────┐                    │
│  │ shared/ (read-only)                  │                    │
│  │  ├── base_venv/ (pandas, numpy, etc) │                    │
│  │  ├── npm_global/ (react, d3, etc)    │                    │
│  │  └── skills/ (shared skill files)    │                    │
│  └──────────────────────────────────────┘                    │
└──────────────────────────────────────────────────────────────┘
```

### Isolation layers

| Layer | What it prevents | Overhead |
|---|---|---|
| **UID isolation** | Session A accessing session B's files | ~5-10ms per operation |
| **Path validation** | Directory traversal (`../../etc/passwd`) | <1ms |
| **Symlink detection** | Symlink escape attacks | <1ms |
| **Null byte blocking** | Null byte injection in file paths | <1ms |
| **Permission gates** | Unauthorized tool execution | Configurable (allow/deny/ask) |
| **Process isolation** | Runaway processes affecting other sessions | OS-level (setuid) |

### MCP-native

The Executor is an MCP (Model Context Protocol) server. This means:
- Any MCP-compatible client can connect to it
- The Brain uses standard MCP tool calls
- Third-party MCP tools plug in without modification
- The tool protocol is standardized, not custom

### Available sandbox tools

| Tool | What it does |
|---|---|
| `Bash` | Execute shell commands (Python, Node.js, system tools) with timeout |
| `Read` | Read file contents with line range support |
| `Write` | Create or overwrite files |
| `Edit` | Surgical string replacement in files |
| `Glob` | Fast file pattern matching (`**/*.py`, `src/**/*.ts`) |
| `Grep` | Content search with regex support (ripgrep-powered) |

---

## Context Management

Long conversations are the #1 reliability issue in agent systems. Most frameworks either crash or silently truncate context. Cadens uses a smart compaction system:

```
Context Window State:
┌──────────────────────────────────────────────────┐
│ System prompt                           (pinned) │
│──────────────────────────────────────────────────│
│ Summary of turns 1-45          (LLM-generated)   │
│──────────────────────────────────────────────────│
│ Turn 46: user message                            │
│ Turn 47: assistant + tool_use                    │
│ Turn 48: tool_result           (pair preserved)  │
│ Turn 49: assistant response                      │
│ ...                                              │
│ Turn 78: user message          (recent, pinned)  │
│ Turn 79: assistant + tool_use  (recent, pinned)  │
│ Turn 80: tool_result           (recent, pinned)  │
└──────────────────────────────────────────────────┘
```

### Compaction rules

1. **Never break tool pairs** -- `tool_use` and `tool_result` messages are always kept together. Breaking them causes LLM confusion and errors.

2. **Protect recent turns** -- The last N user messages (default: 2) are always kept in full. This ensures the agent remembers recent context.

3. **Summarize, don't truncate** -- Old turns are summarized by the LLM into a condensed checkpoint, not simply deleted. Key facts, decisions, and file paths are preserved.

4. **Threshold-based** -- Compaction only triggers when removing old turns would save >20K tokens. No unnecessary work.

5. **Deterministic ordering** -- The resulting message array always maintains correct alternating user/assistant/tool structure.

### Why this matters

A typical agent conversation can easily reach 100+ turns when the agent is:
- Searching through files (each search = 1 tool call = 2 messages)
- Editing code (read + edit + verify = 6 messages)
- Running multi-step plans (10+ tool calls per plan)

Without smart compaction, agents hit context limits within minutes of real work. With Cadens, conversations run indefinitely.

---

## Multi-Provider LLM Integration

Cadens doesn't just "support multiple providers." It integrates deeply with each provider's unique capabilities:

### Anthropic Claude

| Feature | Implementation |
|---|---|
| **Extended thinking** | Full support with thinking signatures (verified reasoning chains) |
| **Interleaved thinking** | Thinking blocks between tool calls for transparent reasoning |
| **Prompt caching** | System prompts cached for ~90% cost reduction on subsequent messages |
| **Token-efficient tools** | Uses Anthropic's tool format for minimal token overhead |

### OpenAI

| Feature | Implementation |
|---|---|
| **Reasoning effort** | Configurable levels: low / medium / high / xhigh |
| **Structured outputs** | JSON mode with schema validation |
| **Function calling** | Native function call format |

### Google Gemini

| Feature | Implementation |
|---|---|
| **Thinking modes** | Level-based: minimal / low / medium / high |
| **Long context** | Optimized for Gemini's extended context windows |
| **Grounding** | Google Search grounding integration |

### Unified interface

```
LLMProvider (Protocol)
    ├── AnthropicProvider
    ├── OpenAIProvider
    └── GeminiProvider

Each implements:
    - stream_response()
    - format_tools()
    - parse_tool_calls()
    - handle_thinking()
```

Python Protocol-based dependency injection. No abstract base classes. No framework decorators. Swap providers with a single config change.

---

## Sub-Agent System

The Task tool spawns specialized sub-agents that work in parallel:

```
Main Agent (processing user request)
    │
    ├─── Task: "Research competitor pricing"
    │         └── Sub-Agent A (web search + analysis)
    │
    ├─── Task: "Analyze our current metrics"
    │         └── Sub-Agent B (code execution + data processing)
    │
    └─── Task: "Draft comparison report"
              └── Sub-Agent C (waits for A and B, then writes)

Each sub-agent:
  ✓ Gets its own ReAct loop
  ✓ Has access to all tools
  ✓ Runs in its own execution context
  ✓ Returns results to the parent agent
  ✓ Can be configured with different LLM providers/models
```

### How it works

1. The main agent decides a task would benefit from parallel work
2. It calls the `Task` tool with a description and configuration
3. Brain spawns a new ReAct loop instance for the sub-agent
4. The sub-agent runs autonomously, using tools as needed
5. Results flow back to the main agent as tool results
6. The main agent synthesizes sub-agent outputs into its response

This is the same pattern Claude Code uses for its sub-agent system.

---

## Security Model

### Authentication

```
┌─────────────────────────────────────────────────┐
│                Authentication Flow               │
│                                                  │
│  Login → JWT access token (24h) + refresh (30d)  │
│                                                  │
│  Tokens stored as HttpOnly cookies:              │
│    ✓ XSS-resistant (JS can't read them)          │
│    ✓ SameSite=Lax (CSRF protection)              │
│    ✓ Secure flag in production                   │
│                                                  │
│  Refresh token rotation:                         │
│    ✓ Each refresh issues new pair                 │
│    ✓ Old refresh token invalidated               │
│    ✓ Prevents replay attacks                     │
│                                                  │
│  OAuth providers:                                │
│    ✓ Google                                      │
│    ✓ GitHub                                      │
│    ✓ Email/password (bcrypt hashed)              │
└─────────────────────────────────────────────────┘
```

### Service-to-service auth

| Connection | Auth method |
|---|---|
| Frontend → Backend | JWT in HttpOnly cookie |
| Backend → Brain | Shared secret (`X-Backend-Secret` header) |
| Brain → Executor | Internal network (no external access) |
| Brain → Crawl4AI | Internal network |

### Threat model

| Threat | Mitigation |
|---|---|
| User A reads User B's files | UID isolation at OS level; each session runs as different Unix user |
| Path traversal (`../../etc/passwd`) | Explicit path validation: resolve, check prefix, block traversal patterns |
| Symlink escape | Symlink detection before file operations |
| Null byte injection | Null byte check on all file paths |
| Code execution breakout | Sandboxed workspace with restricted permissions; no network access from sandbox |
| Session hijacking | HttpOnly cookies, token rotation, secure flag |
| CSRF | SameSite=Lax, origin header validation |

---

## Resilience & Fault Tolerance

All tool execution in Cadens inherits from `ResilientMCPMixin`:

```
┌─────────────────────────────────────────────┐
│           Resilient Tool Execution           │
│                                              │
│  Request ──► Retry Logic ──► Circuit Breaker │
│                  │                │          │
│                  │          ┌─────▼────────┐ │
│                  │          │   CLOSED     │ │
│                  │          │   (normal)   │ │
│                  │          └──────┬───────┘ │
│                  │                 │ N fails │
│                  │          ┌──────▼───────┐ │
│                  │          │   OPEN       │ │
│                  │          │   (fast fail)│ │
│                  │          └──────┬───────┘ │
│                  │                 │ timeout │
│                  │          ┌──────▼───────┐ │
│                  │          │  HALF-OPEN   │ │
│                  │          │  (test one)  │ │
│                  │          └─────────────┘  │
│                  │                           │
│  Exponential backoff:                        │
│    Attempt 1: immediate                      │
│    Attempt 2: 1s delay                       │
│    Attempt 3: 2s delay                       │
│    Attempt 4: 4s delay                       │
│                                              │
│  Graceful degradation:                       │
│    If executor is down, agent reports the     │
│    issue to the user instead of crashing     │
└─────────────────────────────────────────────┘
```

---

## Skills System

Skills are reusable, composable capabilities that extend the agent with specialized knowledge:

```
User types: /analyze-data

┌──────────────────────────────────────┐
│ Skill: analyze-data                  │
│                                      │
│ 1. Reads skill definition file       │
│ 2. Injects specialized prompt        │
│ 3. Pre-configures tool access        │
│ 4. Sets output format expectations   │
│ 5. Agent executes with skill context │
└──────────────────────────────────────┘
```

Skills can be:
- **Shared** across all users (installed globally)
- **Per-user** (personal workflows)
- **Chained** (one skill triggers another)
- **Parameterized** (accept arguments)

This is the same pattern Claude Code uses for its skill system.

---

## Artifact Rendering

When the agent creates visual outputs, they render inline in the chat:

| Type | Rendering | Example |
|---|---|---|
| **HTML** | Full page in sandboxed iframe | Interactive dashboards, styled reports |
| **React/JSX** | Babel-transpiled, rendered in iframe | Data visualizations, interactive components |
| **Mermaid** | Rendered as SVG diagrams | Flowcharts, sequence diagrams, architecture |
| **SVG** | Direct SVG rendering | Charts, icons, graphics |
| **Markdown** | Rendered markdown | Documents, READMEs |
| **CSV/Data** | Table rendering | Data exports, reports |

### How it works

1. Agent writes a file to `/outputs/` (e.g., `dashboard.html` or `chart.jsx`)
2. Agent references it with a `<file>` tag in the response
3. Frontend detects the file type and renders it inline
4. For React: Babel transpiles JSX in-browser, injects into sandboxed iframe
5. External libraries available: Recharts, D3, Three.js, Plotly, Chart.js, Tailwind

---

## Database Architecture

Two MongoDB databases with a strict single-writer-per-collection pattern:

```
┌─────────────────────────────────┬──────────────────────────────────┐
│   Backend Database               │   Brain Database                 │
│   (Writer: Backend only)         │   (Writer: Brain only)           │
│   (Reader: Backend + Brain RO)   │   (Reader: Brain + Backend RO)   │
│                                  │                                  │
│   ┌──────────────────────┐      │   ┌──────────────────────┐      │
│   │ users                │      │   │ agent_runs           │      │
│   │ sessions             │◄─────│──▶│ tool_events          │      │
│   │ messages             │  RO  │   │ file_captions        │      │
│   │ tasks                │reads │   │ agents               │      │
│   │ dashboard_widgets    │      │   │ agent_tasks          │      │
│   │ file_attachments     │      │   │ todos                │      │
│   └──────────────────────┘      │   └──────────────────────┘      │
└─────────────────────────────────┴──────────────────────────────────┘
```

### Why two databases?

| Benefit | Explanation |
|---|---|
| **No write conflicts** | Each collection has exactly one writer. No distributed locking needed. |
| **Independent scaling** | Backend DB handles fast CRUD; Brain DB handles AI workload writes. |
| **Clear data ownership** | Easy to reason about who mutates what. |
| **Simpler operations** | Each database is independently consistent for backup/restore. |

### Why MongoDB?

| Reason | Detail |
|---|---|
| **Schema flexibility** | Chat messages embed interactive components, tool results, artifacts -- the shape varies wildly per message |
| **Document model fits chat** | A conversation is naturally a document (session) with embedded documents (messages) |
| **Async driver** | Motor provides native async/await support for FastAPI |
| **Atlas managed hosting** | Operational simplicity for production deployments |

---

## Design Decisions & Trade-offs

### SSE over WebSockets

**Chose:** Server-Sent Events for streaming

**Why:** Unidirectional streaming (server → client) is all we need for token streaming. SSE is simpler than WebSockets, works through all proxies/load balancers without special config, has built-in browser reconnection via EventSource API, and doesn't require connection state management on the backend.

**Trade-off:** User messages go via REST (not real-time). This is fine -- users send messages infrequently compared to the token stream.

### MCP over custom HTTP for Brain ↔ Executor

**Chose:** Model Context Protocol (Streamable HTTP transport)

**Why:** Standardized tool protocol means third-party MCP tools work out of the box. The executor is a "dumb sandbox" -- it doesn't know about LLMs, prompts, or agents. It just executes tool calls. This clean separation means the executor can be replaced with any MCP-compatible server.

**Trade-off:** MCP adds a small abstraction layer vs direct HTTP. Debugging MCP calls is slightly harder than plain REST. Worth it for the interoperability.

### Four services vs monolith

**Chose:** Microservice split (Frontend / Backend / Brain / Executor)

**Why:** Security isolation (executor runs untrusted code), independent scaling (brain is compute-heavy), clear ownership (each service owns its data writes), and the ability to swap any layer independently.

**Trade-off:** More operational complexity. Acceptable because Docker Compose handles it locally, and the services are well-bounded.

### Protocol-based DI over class inheritance

**Chose:** Python Protocols for dependency injection

**Why:** Swap the LLM provider, executor, or storage layer without touching any base classes. Test anything in isolation with simple mock objects. No framework-specific decorators or abstract base classes.

**Trade-off:** Less discoverable for developers used to inheritance-based frameworks. We trade discoverability for flexibility and testability.

### MongoDB over PostgreSQL

**Chose:** MongoDB

**Why:** Chat messages are inherently document-shaped -- each message can contain text, tool calls, artifacts, interactive components, or embedded data. The schema evolves rapidly as we add new features. MongoDB's flexible schema lets us iterate without migrations.

**Trade-off:** No foreign key constraints (referential integrity is application-level), no ACID transactions across collections (mitigated by single-writer pattern). If your use case is heavily relational, PostgreSQL would be better -- but chat-based agents aren't.

---

## What's next

This design document covers the current architecture. As Cadens evolves, we're building:

- **Connector framework** -- Standardized OAuth + tool bundles for external platforms (Slack, GitHub, Google Workspace, etc.)
- **BYOK (Bring Your Own Key)** -- Use your own LLM API keys with per-user key management
- **Proactive agent** -- Scheduled and event-driven tasks; agent initiates work, not just responds
- **Team & permissions** -- Role-based access control, shared sessions, team-wide knowledge
- **Approval workflows** -- Human-in-the-loop gates for high-stakes agent actions
- **Webhooks & headless API** -- Programmatic access and event-driven integrations
- **White-label / branding** -- Deploy as your own product with custom identity
- **Horizontal scaling guide** -- Running Brain and Executor across multiple nodes

---

*This document reflects the system as extracted from production. Every pattern described here has been tested under real workloads with real users.*
