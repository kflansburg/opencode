# Migrating opencode Web Server to Cloudflare Workers

This document analyzes the feasibility of migrating the `opencode web` server to Cloudflare Workers, detailing the current architecture, Node.js/Bun API dependencies, and required architectural changes.

## Executive Summary

The opencode server is architected as a **stateful, long-running process** that orchestrates local tools, manages child processes, and maintains persistent filesystem state. While the HTTP/SSE layer (Hono) is highly portable to Workers, the core value proposition—running local development tools as an AI agent—requires fundamental architectural changes.

**Portable:** REST API routes, SSE event streaming, AI provider integrations, database ORM layer
**Requires Replacement:** SQLite → D1, in-memory state → Durable Objects, Bun.serve → Worker fetch handler
**Fundamentally Incompatible:** All subprocess spawning, filesystem operations, PTY terminals, file watching

The most viable migration path involves splitting the system into:

1. **Worker backend** — Hono API + SSE + D1 + Durable Objects for session state
2. **Remote execution sidecar** — Container or agent process for bash/git/LSP/file operations

---

## Current Architecture

### Server Stack

**Framework:** Hono  
**Runtime:** Bun  
**Entry Point:** `packages/opencode/src/server/server.ts` via `opencode web` command (`src/cli/cmd/web.ts`)

The server uses `Bun.serve()` to host a Hono app that serves three transport protocols:

| Protocol                     | Endpoint                                       | Purpose                         |
| ---------------------------- | ---------------------------------------------- | ------------------------------- |
| **HTTP REST**                | `/session/*`, `/config/*`, `/provider/*`, etc. | All CRUD operations             |
| **Server-Sent Events (SSE)** | `GET /global/event`, `GET /event`              | Real-time event streaming       |
| **WebSocket**                | `GET /pty/:id/connect`                         | Bidirectional terminal I/O only |

**Port:** Tries 4096 first, falls back to random  
**Auth:** Basic auth via `OPENCODE_SERVER_PASSWORD` env var  
**CORS:** Allows `localhost:*`, `127.0.0.1:*`, `tauri://localhost`, `*.opencode.ai`  
**mDNS:** Optional Bonjour service advertising via `bonjour-service` (uses UDP/dgram)

The catchall route (`/*`) proxies unmatched requests to `https://app.opencode.ai` to serve the web frontend.

**Source:** `packages/opencode/src/server/server.ts`, `packages/opencode/src/cli/cmd/web.ts`

---

## Transport Protocols

### 1. HTTP REST

Standard Hono routes for CRUD operations. All routes use:

- OpenAPI spec generation via `hono-openapi` (`describeRoute`, `validator`, `resolver`)
- Zod schemas for validation
- Instance scoping via `x-opencode-directory` header or `directory` query param
- Basic auth middleware (if `OPENCODE_SERVER_PASSWORD` is set)

**Compatibility:** ✅ Fully portable to Workers (Hono has first-class Workers support)

### 2. Server-Sent Events (SSE)

Two SSE endpoints:

**Instance-scoped** (`GET /event`):

- Calls `Bus.subscribeAll()` to subscribe to all events for a specific project directory
- Streams JSON events via `streamSSE` from `hono/streaming`
- 30-second heartbeat (`server.heartbeat`)
- Closes on `InstanceDisposed` event

**Global** (`GET /global/event`):

- Listens on `GlobalBus.on("event", ...)` for cross-instance events
- Payloads include `{ directory, payload }` for routing
- Also streams via `streamSSE` with 30s heartbeat

**Compatibility:** ✅ `streamSSE` works on Workers with standard `Response` streaming

### 3. WebSocket (PTY only)

**Route:** `GET /pty/:id/connect`  
**Upgrade:** `upgradeWebSocket` from `hono/bun`  
**Protocol:** Cursor-based reconnection via `?cursor=N` query param

On connection:

1. Server replays buffered output from `cursor` position in 64KB chunks
2. Sends binary control frame (0x00 prefix + JSON `{cursor}`)
3. Client messages → `process.write()` to PTY
4. PTY output → broadcast to all connected sockets

**Compatibility:** ⚠️ WebSocket upgrade works on Workers, but the PTY backend (`bun-pty`) requires a native process, which is incompatible. Would need to proxy to a remote container/VM.

---

## Event System Architecture

### Two-Tier Pub/Sub

**Layer 1: `Bus` (instance-scoped)**  
File: `packages/opencode/src/bus/index.ts`

- Created via `Instance.state()` — one Bus per project directory
- Uses `Map<eventType | "*", Subscription[]>` for subscriptions
- `Bus.publish(def, properties)` notifies specific + wildcard subscribers, then emits to GlobalBus
- `Bus.subscribeAll()` subscribes to the `"*"` wildcard key
- Events: session.created, message.updated, pty.exited, permission.asked, vcs.branch.updated, etc.
- **47 event types** across 15+ domains

**Layer 2: `GlobalBus` (process-scoped)**  
File: `packages/opencode/src/bus/global.ts`

- Single Node.js `EventEmitter` with one channel: `"event"`
- Payloads: `{ directory?, payload }`
- Receives all `Bus.publish()` events plus global events (worktree lifecycle, global disposal)

**Event Flow:**

```
Producer → Bus.publish() → instance subscribers → GlobalBus.emit()
                         ↓                              ↓
                  SSE /event                    SSE /global/event
```

**Schema Registry:**  
`BusEvent.define(type, zodSchema)` registers typed events. `BusEvent.payloads()` builds a discriminated union for OpenAPI spec generation.

**Compatibility:** ✅ EventEmitter and AsyncLocalStorage are both supported in Workers with `nodejs_compat`

---

## Database Layer

**Engine:** SQLite via `bun:sqlite`  
**ORM:** Drizzle ORM (`drizzle-orm/bun-sqlite` adapter)  
**Location:** `<XDG_DATA_HOME>/opencode/opencode.db` (e.g., `~/.local/share/opencode/opencode.db`)

### Configuration

Initialization is lazy (via `lazy()` memoizer). Pragmas:

- `journal_mode = WAL` (Write-Ahead Logging)
- `synchronous = NORMAL`
- `busy_timeout = 5000`
- `cache_size = -64000` (64MB page cache)
- `foreign_keys = ON`

### Schema

Tables (all snake_case):

- `project` — projects with worktree path, VCS info, icon, sandboxes, commands
- `session` — sessions linked to projects (FK cascade delete), title, slug, summary stats
- `message` — messages linked to sessions, with `data` JSON column
- `part` — parts linked to messages and sessions, with `data` JSON column
- `todo` — todos linked to sessions, composite PK on `(session_id, position)`
- `permission` — permissions keyed by project_id, with `data` JSON column
- `session_share` — share metadata linked to sessions
- `control_account` — OAuth accounts with tokens, composite PK on `(email, url)`

**Migrations:**  
Drizzle Kit generates SQL migrations in `packages/opencode/migration/`. In dev mode, loaded via `readdirSync`/`readFileSync`. In production, bundled via compile-time `OPENCODE_MIGRATIONS` constant.

### Transaction Context

`Database.use()` and `Database.transaction()` use AsyncLocalStorage for implicit transaction propagation across async call stacks.

**Compatibility:** ⚠️ `bun:sqlite` not available on Workers. Requires migration to D1. Drizzle supports D1 with the `drizzle-orm/d1` adapter. Migrations would need to be bundled at build time (already supported via `OPENCODE_MIGRATIONS`).

**Source:** `packages/opencode/src/storage/db.ts`, `packages/opencode/src/storage/schema.sql.ts`

---

## State Management

### Instance.provide() — AsyncLocalStorage Context

File: `packages/opencode/src/project/instance.ts`

Creates a per-directory execution context using `AsyncLocalStorage`. Key pattern:

```ts
Instance.provide({
  directory,
  init: InstanceBootstrap,
  async fn() {
    // All code here can access Instance.directory, Instance.worktree, etc.
  },
})
```

Caches initialized instances by directory path in a `Map<string, Promise<Context>>`.

**Compatibility:** ✅ AsyncLocalStorage is supported in Workers

### Instance.state() — Per-Directory Singletons

File: `packages/opencode/src/project/state.ts`

Creates per-directory singleton state in a `Map<directoryKey, Map<initFn, Entry>>`.

Used by ~25+ modules:

- Bus subscriptions
- File watchers
- LSP clients
- VCS state
- Config
- MCP servers
- PTY sessions
- Provider SDK caches

On `Instance.dispose()`, all state entries for that directory are disposed via registered dispose callbacks.

**Compatibility:** ⚠️ In-memory singleton state works in Workers but does not persist across requests. Would need to migrate to **Durable Objects** for persistent, per-directory state with guaranteed single-instance semantics.

**Source:** `packages/opencode/src/project/state.ts`, `packages/opencode/src/util/context.ts`

---

## AI Provider Integration

**SDK:** Vercel AI SDK v5 (`ai@5.0.124`)  
**Core API:** `streamText()` from `"ai"` package  
**Providers:** 22 bundled (Anthropic, OpenAI, Azure, Google Vertex/Generative AI, Amazon Bedrock, OpenRouter, xAI, Mistral, Groq, DeepInfra, Cerebras, Cohere, TogetherAI, Perplexity, Vercel, GitLab, GitHub Copilot, Cloudflare Workers AI/AI Gateway, etc.)

### Streaming Flow

```
SessionPrompt.prompt(input)
  → SessionPrompt.loop()          (multi-turn conversation loop)
    → SessionProcessor.create()   (create assistant message shell)
      → SessionProcessor.process()
        → LLM.stream()
          → Provider.getLanguage(model)   (get or create SDK, fetch LanguageModelV2)
            → streamText()                (Vercel AI SDK call → HTTP to LLM provider)
              → for await (stream.fullStream)
                → reasoning/text/tool-call/tool-result event handling
                → Session.updatePartDelta() → Bus.publish() → SSE stream
```

### Provider SDK Caching

- SDKs are cached using `Bun.hash.xxHash32()` of serialized config as the cache key
- Stored in `Map<number, SDK>` via `Instance.state()`
- Each SDK gets a custom `fetch` wrapper with:
  - Configurable timeouts via `AbortSignal.timeout()` + `AbortSignal.any()`
  - Provider-specific request transformations (e.g., stripping `id` fields for OpenAI)
  - `timeout: false` on Bun fetch (workaround for Bun bug)

### Dynamic Provider Installation

For non-bundled providers, `BunProc.install()` dynamically installs the npm package:

```ts
Bun.spawn([which(), "add", `${name}@${version}`], { cwd: "~/.opencode/cache" })
```

Then `import(modulePath)` and call the provider's factory function.

**Compatibility:**  
✅ `streamText()` uses standard `fetch()` — fully compatible  
✅ Custom fetch wrappers use `AbortSignal` — supported  
⚠️ `Bun.hash.xxHash32()` not available — use Web Crypto `crypto.subtle.digest()`  
❌ Dynamic package installation via `Bun.spawn()` not possible — would need pre-bundled providers or a build-time plugin registry

**Source:** `packages/opencode/src/provider/provider.ts`, `packages/opencode/src/session/llm.ts`, `packages/opencode/src/session/processor.ts`

---

## PTY (Pseudo-Terminal) Subsystem

**Native Module:** `bun-pty` v0.4.8 (NOT `node-pty`)  
**Loading:** Lazy via `import("bun-pty")`  
**State:** Per-instance (`Instance.state()`)

### Session Structure

```ts
ActiveSession {
  info: { id, title, command, args, cwd, status, pid }
  process: IPty              // bun-pty process handle
  buffer: string             // 2MB scrollback buffer
  bufferCursor: number       // offset of buffer start in total stream
  cursor: number             // total bytes emitted
  subscribers: Map<Socket, number>  // connected WebSocket clients
}
```

### WebSocket Protocol

**Route:** `GET /pty/:id/connect`  
**Reconnection:** Client sends `?cursor=N` to replay from that position

1. Server replays buffered data from `cursor` in 64KB chunks
2. Sends binary control frame: `0x00` + UTF-8 JSON `{ cursor }`
3. Client messages → `process.write()` to PTY
4. PTY output → broadcast to all subscribers

**Socket Tagging:** Each socket gets a unique ID via `WeakMap` to prevent stale subscriber references.

### Events

- `pty.created`
- `pty.updated`
- `pty.exited`
- `pty.deleted`

**Compatibility:** ❌ `bun-pty` is a native FFI module that spawns actual PTY processes. Not possible on Workers. Would require a container/VM backend with WebSocket proxy.

**Source:** `packages/opencode/src/pty/index.ts`, `packages/opencode/src/server/routes/pty.ts`

---

## Subprocess Management

opencode spawns subprocesses extensively for AI agent tool execution and development tools.

### Spawn Patterns

**1. `child_process.spawn`** (6 files)

- LSP language servers (need stdio pipes)
- Bash tool (needs shell interpretation + detached process groups)
- Git commands (some CLI commands)

**2. `Bun.spawn()`** (~25 files)

- Git operations (`src/util/git.ts`)
- Ripgrep (`src/file/ripgrep.ts`)
- Code formatters (`src/format/formatter.ts`)
- Bun package manager (`src/bun/index.ts`)
- Clipboard operations (`src/cli/cmd/tui/util/clipboard.ts`)
- IDE launching (`src/ide/index.ts`)

**3. `Bun.$` shell template** (~15 files)

- Simple shell commands (worktree management, archive extraction, etc.)

### Bash Tool (AI Agent)

File: `packages/opencode/src/tool/bash.ts`

The bash tool is how the AI agent executes shell commands. It:

1. Parses the command with **web-tree-sitter** (`tree-sitter-bash` grammar) for AST analysis
2. Extracts individual commands and arguments for permission checks
3. Detects filesystem-modifying commands (`cd`, `rm`, `cp`, `mv`, `mkdir`, `touch`, `chmod`, `chown`, `cat`)
4. Resolves target paths to check if they're outside the project boundary (triggers `external_directory` permission)
5. Spawns via `child_process.spawn()` with `{ shell: <selected-shell>, detached: true }`
6. Streams stdout/stderr to UI via metadata updates (capped at 30KB)
7. Supports 2-minute timeout (configurable) and AbortSignal cancellation

### Process Tree Killing

`Shell.killTree()` in `src/shell/shell.ts`:

- Unix: `process.kill(-pid, 'SIGTERM')` (negative PID = process group), waits 200ms, then `SIGKILL`
- Windows: `spawn('taskkill', ['/f', '/t', '/pid', pid])`

**Compatibility:** ❌ All subprocess spawning is incompatible with Workers. `child_process` and `Bun.spawn()` are non-functional stubs. Would require:

- Remote execution sidecar (agent running on user's machine or in a container)
- **Cloudflare Containers** for ephemeral sandboxed execution
- Or hybrid architecture where Workers proxy commands to a container service

**Source:** `packages/opencode/src/tool/bash.ts`, `packages/opencode/src/shell/shell.ts`, `packages/opencode/src/lsp/server.ts`

---

## Filesystem Operations

### Bun.file() — Pervasive Usage

**~100+ call sites** across the codebase for:

- Reading files (`.json()`, `.text()`, `.bytes()`, `.exists()`, `.stat()`)
- Writing files (via `Bun.write()` — ~30 call sites)
- File pattern matching (`Bun.Glob` — ~20 call sites)

### Node.js fs Module

- `fs.readFileSync` / `fs.readdirSync` — migration loading, config discovery
- `fs.existsSync` — checking file existence
- `fs.createReadStream` — line-by-line file reading in the read tool
- `fs.realpathSync` — symlink resolution
- `fs/promises` — mkdir, stat, rm, unlink

### File Watcher

File: `packages/opencode/src/file/watcher.ts`

Uses platform-specific watching:

- macOS: FSEvents
- Linux: inotify
- Windows: ReadDirectoryChangesW

Emits `file.watcher.updated` events for `.gitignore` changes, project config changes, etc.

**Compatibility:** ❌ No persistent filesystem on Workers. The ephemeral filesystem is per-request and isolated. File watching is not possible. Options:

- **R2** for blob storage
- Remote filesystem proxy to a sidecar agent
- Workers with **hyperdrive** + filesystem abstraction layer (experimental)

---

## Client Architecture

**Framework:** SolidJS  
**Package:** `packages/app/`  
**SDK:** `@opencode-ai/sdk/v2/client` (auto-generated from OpenAPI via `@hey-api/openapi-ts`)

### Client-Server Communication

| Protocol  | Endpoint                              | Purpose                                               |
| --------- | ------------------------------------- | ----------------------------------------------------- |
| REST      | Various (`/session`, `/config`, etc.) | CRUD operations via SDK `client.session.list()`       |
| SSE       | `GET /global/event`                   | Single persistent connection for all real-time events |
| WebSocket | `GET /pty/:id/connect`                | Terminal I/O only                                     |

### Provider Hierarchy

```
PlatformProvider          (web vs desktop abstraction)
  └─ ServerProvider       (server URL, health polling)
       └─ GlobalSDKProvider   (SSE connection + REST client)
            └─ GlobalSyncProvider   (event reducer → SolidJS stores)
                 └─ Router
                      └─ SDKProvider      (per-directory scoped client)
                           └─ SyncProvider     (per-directory state access)
```

### Event Streaming

**GlobalSDKProvider** (`packages/app/src/context/global-sdk.tsx`):

- Opens persistent SSE connection to `GET /global/event`
- Events are queued and flushed in batched `batch()` calls every ~16ms
- High-frequency events (`session.status`, `lsp.updated`, `message.part.updated`) are coalesced — only latest per key survives
- On failure, reconnects after 250ms delay

**GlobalSyncProvider** (`packages/app/src/context/global-sync.tsx`):

- Routes events to `applyGlobalEvent()` or `applyDirectoryEvent()`
- Maintains SolidJS stores: sessions, messages, parts, permissions, questions, todos, diffs, VCS, LSP, MCP
- Uses binary search for efficient sorted insertion/update/removal
- LRU eviction for per-directory child stores

**Compatibility:** ✅ Client architecture is frontend-only and independent of backend runtime

---

## Node.js/Bun API Usage Summary

### Fully Compatible with Workers (via `nodejs_compat`)

| API                                  | Files | Usage                                      |
| ------------------------------------ | ----- | ------------------------------------------ |
| `path`                               | ~49   | join, resolve, basename, relative, dirname |
| `AsyncLocalStorage`                  | 1     | Context DI (`util/context.ts`)             |
| `EventEmitter`                       | 1     | GlobalBus (`bus/global.ts`)                |
| `crypto.randomBytes`                 | 1     | ID generation (`id/id.ts`)                 |
| `http.STATUS_CODES`                  | 2     | Error messages                             |
| `url` (pathToFileURL, fileURLToPath) | 5     | LSP URI conversions                        |

### Partially Compatible (need alternatives)

| API                       | Files | Usage                  | Workers Alternative                   |
| ------------------------- | ----- | ---------------------- | ------------------------------------- |
| `os.homedir`              | ~8    | Config/skill discovery | Use env vars or D1 config             |
| `os.platform` / `os.arch` | ~10   | Platform detection     | `process.platform` works              |
| `os.EOL`                  | ~10   | Line endings           | Hardcode `\n` or use constant         |
| `process.env`             | ~150+ | Environment variables  | Supported (but env works differently) |
| `process.platform`        | ~50+  | Platform checks        | Supported                             |

### Incompatible with Workers

| API                                   | Files | Usage                      | Migration Path                        |
| ------------------------------------- | ----- | -------------------------- | ------------------------------------- |
| **`bun:sqlite`**                      | 2     | Primary database           | → D1 (Drizzle D1 adapter)             |
| **`child_process.spawn`**             | 6     | LSP, bash, git             | → Remote sidecar or Containers        |
| **`Bun.spawn()`**                     | ~25   | Git, ripgrep, formatters   | → Remote sidecar or Containers        |
| **`Bun.$`**                           | ~15   | Shell commands             | → Remote sidecar or Containers        |
| **`Bun.serve()`**                     | 3     | HTTP server                | → `export default { fetch }`          |
| **`bun-pty`**                         | 1     | Terminal sessions          | → Container/VM backend                |
| **`Bun.file()`**                      | ~100+ | All file I/O               | → R2 or remote FS proxy               |
| **`Bun.write()`**                     | ~30+  | File writing               | → R2 or remote FS proxy               |
| **`Bun.Glob`**                        | ~20   | Pattern matching           | → R2 list or remote FS                |
| **`Bun.which()`**                     | ~40   | Executable lookup          | → Not applicable (no binaries)        |
| **`Bun.hash.xxHash32()`**             | 1     | Provider cache keys        | → Web Crypto `crypto.subtle.digest()` |
| **`fs.createReadStream`**             | 1     | Line-by-line reading       | → Fetch + stream processing           |
| **`fs.readFileSync`** / `readdirSync` | 5     | Migration/config loading   | → Bundle at build time                |
| **`fs.existsSync`**                   | 3     | File existence checks      | → R2 HEAD or remote FS                |
| **`fs.realpathSync`**                 | 1     | Symlink resolution         | → Not applicable                      |
| **`bonjour-service`** (mDNS)          | 1     | Service discovery          | → Remove (uses UDP/dgram)             |
| **`bun:ffi`**                         | 1     | Windows console (TUI only) | → Not applicable                      |
| **`process.kill()`**                  | 2     | Process management         | → Not applicable                      |
| **`process.stdin.setRawMode`**        | 3     | Terminal raw mode (TUI)    | → Not applicable                      |
| **`v8.writeHeapSnapshot`**            | 1     | Debugging                  | → Not applicable                      |

---

## Migration Strategy

### Option A: Hybrid Architecture (Recommended)

**Workers Backend:**

- Hono HTTP server (`export default { fetch }`)
- REST API routes (minimal changes)
- SSE event streaming via `streamSSE`
- D1 database (Drizzle migration)
- Durable Objects for per-directory session state
- AI provider integrations (streamText via fetch)

**Remote Execution Sidecar:**

- Agent process running on user's machine (or Cloudflare Container)
- Handles: bash commands, git operations, file I/O, LSP servers, PTY sessions
- Communicates with Worker via:
  - WebSocket for bidirectional command/result streaming
  - Or HTTP API for request/response pattern

**Benefits:**

- Preserves existing AI agent capabilities
- Scales the HTTP/SSE layer to Workers' global network
- Offloads compute-intensive operations to containers
- Maintains local development workflow

**Challenges:**

- Requires deploying and managing sidecar infrastructure
- Latency for tool execution (network round-trip)
- Authentication and security between Worker and sidecar

### Option B: Cloud-Native Rewrite

Redesign the system to run entirely on Workers primitives:

**Database:** D1  
**State:** Durable Objects for session state  
**File Storage:** R2  
**Tool Execution:** Cloudflare Containers (ephemeral sandboxes)  
**Terminal:** VNC/web-based terminal via Container

**Benefits:**

- Fully serverless, no sidecar management
- Global edge deployment
- Automatic scaling

**Challenges:**

- Significant rewrite required
- Containers may not support all LSP servers/formatters
- Higher latency for file operations (R2 reads)
- Loss of local development workflow (everything remote)

### Option C: Hono API Only (Minimal Migration)

Migrate only the REST API + SSE layer to Workers. Keep tool execution and filesystem on a traditional Node.js/Bun server.

Workers handle:

- HTTP routing (Hono)
- SSE event streaming
- AI provider SDK calls
- Session/message CRUD (D1)

Traditional server handles:

- Bash tool execution
- LSP servers
- File operations
- PTY sessions

**Benefits:**

- Smallest migration scope
- Reuses Workers for global CDN + DDoS protection
- Keeps complex subprocess logic on traditional runtime

**Challenges:**

- Dual infrastructure (Workers + VMs)
- Cross-service communication overhead
- Limited benefit over just using a global load balancer

---

## Detailed Component Analysis

### ✅ Portable to Workers (Minimal Changes)

| Component                | Files                           | Notes                                                 |
| ------------------------ | ------------------------------- | ----------------------------------------------------- |
| Hono HTTP framework      | `server/server.ts`              | Replace `Bun.serve()` with `export default { fetch }` |
| REST API routes          | `server/routes/*`               | No changes needed                                     |
| SSE streaming            | `server/server.ts:502-540`      | `streamSSE` from `hono/streaming` works on Workers    |
| OpenAPI spec generation  | `server/server.ts:559-572`      | `hono-openapi` compatible                             |
| Zod schemas              | All `*.sql.ts`, route files     | No runtime dependency on Node/Bun                     |
| Bus pub/sub system       | `bus/index.ts`, `bus/global.ts` | EventEmitter + AsyncLocalStorage supported            |
| AI provider integrations | `provider/provider.ts`          | Uses standard `fetch()`                               |
| Vercel AI SDK streaming  | `session/llm.ts`                | `streamText()` works on Workers                       |
| URL/path utilities       | 5 files                         | `url` module fully supported                          |
| Crypto (randomBytes)     | `id/id.ts`                      | Supported, or use Web Crypto                          |

### ⚠️ Requires Replacement

| Component           | Current                          | Workers Alternative           |
| ------------------- | -------------------------------- | ----------------------------- |
| Database            | `bun:sqlite`                     | D1 + `drizzle-orm/d1`         |
| Per-directory state | `Instance.state()` in-memory Map | Durable Objects               |
| HTTP server         | `Bun.serve()`                    | `export default { fetch }`    |
| Provider cache keys | `Bun.hash.xxHash32()`            | `crypto.subtle.digest()`      |
| Migration loading   | `fs.readdirSync`                 | Bundle at build time          |
| Config discovery    | `os.homedir()` + `Bun.file()`    | Env vars or D1 config storage |

### ❌ Fundamentally Incompatible

| Component                   | Current Implementation           | Why Incompatible         | Workaround                               |
| --------------------------- | -------------------------------- | ------------------------ | ---------------------------------------- |
| **Bash tool**               | `child_process.spawn()`          | No process spawning      | Remote sidecar or Containers             |
| **LSP servers**             | `child_process.spawn()`          | No process spawning      | Remote sidecar or language-specific APIs |
| **Git operations**          | `Bun.spawn(['git', ...])`        | No process spawning      | Remote sidecar or Git HTTP API           |
| **Ripgrep**                 | `Bun.spawn(['rg', ...])`         | No binaries              | Remote sidecar or R2 + Workers search    |
| **Code formatters**         | `Bun.spawn()`                    | No binaries              | Remote sidecar or WASM formatters        |
| **PTY sessions**            | `bun-pty` native FFI             | No native modules        | Container-based terminal                 |
| **File I/O**                | `Bun.file()`, `Bun.write()`      | No persistent filesystem | R2 or remote FS proxy                    |
| **File watching**           | Platform file watchers           | No filesystem            | Polling or webhook-based updates         |
| **Dynamic package install** | `Bun.spawn(['bun', 'add', ...])` | No process spawning      | Pre-bundle or runtime import from CDN    |
| **mDNS discovery**          | `bonjour-service` (UDP)          | No UDP sockets           | Remove or use HTTP service registry      |
| **Process tree killing**    | `process.kill(-pid)`             | No process management    | N/A                                      |

---

## Key Architectural Patterns to Preserve

### 1. Instance.provide() Context Propagation

AsyncLocalStorage-based context works on Workers. The pattern of `Instance.provide({ directory, fn })` can remain unchanged.

### 2. Bus Event System

EventEmitter + wildcard subscriptions + GlobalBus is fully compatible. The `BusEvent.define()` registry and `Bus.publish()` flow can remain as-is.

### 3. SSE Event Streaming

`streamSSE` from `hono/streaming` works on Workers. The 30s heartbeat and JSON event serialization pattern is portable.

### 4. Drizzle ORM Queries

The query layer can remain unchanged by switching to the D1 adapter. Only the `Database.Client` initialization needs to change from `bun:sqlite` to D1 binding.

---

## Migration Checklist

### Phase 1: Hono + SSE + D1 (No Tool Execution)

- [ ] Replace `Bun.serve()` with `export default { fetch(request, env, ctx) }`
- [ ] Add D1 binding in `wrangler.toml`
- [ ] Change Drizzle adapter from `drizzle-orm/bun-sqlite` to `drizzle-orm/d1`
- [ ] Bundle migrations at build time (remove `fs.readdirSync` loading)
- [ ] Replace `Bun.hash.xxHash32()` with `crypto.subtle.digest()`
- [ ] Test REST API routes
- [ ] Test SSE event streaming
- [ ] Remove mDNS functionality

### Phase 2: Durable Objects for State

- [ ] Create Durable Object class for per-directory state
- [ ] Migrate `Instance.state()` to Durable Object storage
- [ ] Migrate Bus subscriptions to Durable Object
- [ ] Test instance scoping and disposal

### Phase 3: Remote Execution Sidecar

- [ ] Design sidecar protocol (WebSocket or HTTP)
- [ ] Implement bash tool proxy
- [ ] Implement file I/O proxy (read/write/glob)
- [ ] Implement git operations proxy
- [ ] Implement LSP server proxy
- [ ] Implement PTY session proxy
- [ ] Add authentication between Worker and sidecar

### Phase 4: Production Hardening

- [ ] Add R2 for blob storage (file uploads, archives)
- [ ] Implement rate limiting
- [ ] Add observability (logs, metrics, traces)
- [ ] Security audit (especially sidecar communication)
- [ ] Performance testing (latency of proxied operations)
- [ ] Documentation and deployment guides

---

## Conclusion

Migrating opencode to Cloudflare Workers is **architecturally feasible** but requires a **hybrid approach**: Workers for the HTTP/SSE/database layer, and a remote execution sidecar (or Cloudflare Containers) for subprocess/filesystem operations.

**Recommended Path:** Start with Phase 1 (Hono + D1) as a proof-of-concept, then design the sidecar protocol in Phase 3. Evaluate whether Cloudflare Containers can replace the sidecar for fully serverless operation.

The most significant challenge is maintaining the developer experience of local tool execution while operating in a distributed, serverless environment.
