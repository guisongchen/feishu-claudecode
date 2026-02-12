# Feishu-ClaudeCode Architecture

## Project Structure

```
src/
├── index.ts                    # Entry point, dependency injection, lifecycle
├── config.ts                   # Central configuration from env vars
├── bridge/
│   ├── message-bridge.ts       # Core orchestrator (commands + query execution)
│   └── rate-limiter.ts         # Throttles Feishu card updates (1.5s default)
├── claude/
│   ├── executor.ts             # Agent SDK wrapper (query() as async generator)
│   ├── stream-processor.ts     # Transforms SDK stream → CardState
│   └── session-manager.ts      # In-memory sessions per chatId (24h TTL)
├── feishu/
│   ├── event-handler.ts        # WS event parsing, auth, @mention filtering
│   ├── card-builder.ts         # Interactive card JSON builder
│   └── message-sender.ts       # Feishu API wrapper (send/update/upload)
└── utils/
    └── logger.ts               # Pino logger setup
```

## Design Architecture

### 1. Layered Pipeline Architecture

```
Feishu WS → Event Handler → Message Bridge → Claude Executor → Stream Processor → Card Updates
```

Each layer has a single responsibility:
- **Event Handler**: Authentication, message parsing, @mention stripping
- **Message Bridge**: Command routing, task lifecycle, coordination
- **Claude Executor**: SDK interaction with streaming
- **Stream Processor**: State aggregation from raw SDK events

### 2. Session Isolation Model

- Sessions keyed by `chatId` (not userId) — each group chat and DM is isolated
- Each session has: working directory, Claude session ID for resume, lastUsed timestamp
- Changing working directory resets the Claude session (session ID cleared)
- 24-hour TTL with hourly cleanup

### 3. Concurrency Control

- One task per chat: `runningTasks: Map<chatId, RunningTask>`
- 10-minute timeout per task
- AbortController for cancellation (`/stop` command or shutdown)

### 4. Streaming State Machine

`StreamProcessor` maintains state across SDK events:
- `thinking` → `running` (when tools execute) → `complete` | `error`
- Accumulates tool calls, response text, cost/duration metrics
- Tracks image paths from Write tool for auto-sending back to Feishu

### 5. Rate-Limited Updates

- `RateLimiter` throttles card updates to avoid Feishu API limits
- Keeps only the latest pending update (drops intermediate states)
- Flush on completion ensures final state is always sent

### 6. Security Model

- **Authorization**: Whitelist by userId or chatId (env-configured)
- **Group chat filtering**: Only responds when @mentioned
- **Permission mode**: `bypassPermissions` (runs without terminal prompts)
- **Tool allowlist**: Configurable (default: Read, Edit, Write, Glob, Grep, Bash)

### 7. Key Design Decisions

| Aspect | Choice | Rationale |
|--------|--------|-----------|
| Transport | WebSocket (not webhook) | No public IP required |
| Module system | ESM with `.js` extensions | TypeScript Node best practice |
| Image handling | Download to temp, reference in prompt | Claude analyzes via Read tool |
| Output images | Auto-detect from Write tool + text scan | Seamless image workflow |
| Card content | 28KB truncation | Feishu card size limits |

### 8. Message Flow

1. User sends message → Feishu WS → `EventHandler`
2. Auth check, parse text/image, strip @mentions
3. `MessageBridge.handleMessage()` routes commands or queries
4. Commands (`/cd`, `/reset`, `/stop`, `/status`, `/help`) handled synchronously
5. Queries: Create `ClaudeExecutor` stream → `StreamProcessor` → throttled card updates
6. Final card sent + any output images uploaded and sent separately
