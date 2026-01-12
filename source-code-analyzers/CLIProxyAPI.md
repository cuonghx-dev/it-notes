# CLIProxyAPI Architecture Overview

CLIProxyAPI is a **Go proxy server** that provides unified API interfaces (OpenAI/Gemini/Claude/Codex compatible) for accessing multiple AI providers with multi-account support.

## Core Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Incoming Request                            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Auth Middleware (validates request via AccessManager)          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Protocol Handler (OpenAI, Claude, Gemini)                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Translator (converts between API formats)                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Core Auth Manager (selects credential, handles quota)          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Executor (makes upstream API call)                             │
└─────────────────────────────────────────────────────────────────┘
```

## Directory Structure

| Directory | Purpose |
|-----------|---------|
| `cmd/server/` | Entry point, CLI flags, initialization |
| `internal/api/` | HTTP server (Gin), routes, middleware |
| `internal/auth/` | OAuth implementations per provider |
| `internal/translator/` | Format conversion between API schemas |
| `internal/config/` | YAML config loading and validation |
| `internal/watcher/` | Hot-reload for config and auth files |
| `internal/store/` | Token persistence (file, PostgreSQL, Git, S3) |
| `sdk/cliproxy/` | Embeddable SDK with builder pattern |
| `sdk/api/handlers/` | Protocol-specific API handlers |

## Key Components

### 1. HTTP API Server (`internal/api/server.go`)

- Gin-based HTTP server with middleware chain
- Routes requests to protocol-specific handlers
- Applies CORS, logging, and authentication middleware
- Supports WebSocket upgrade for streaming responses
- Manages hot-reload configuration updates

### 2. Authentication System (`sdk/cliproxy/auth/`)

- **Token Lifecycle Management**: OAuth2 flows for Gemini, Claude, Codex, Qwen, iFlow, Antigravity
- **Core Auth Conductor** (`conductor.go`): Selects credentials based on model and routing strategy
- **Request Authentication**: AccessManager validates incoming requests per provider

### 3. Translator Layer (`internal/translator/`)

Bidirectional format conversion between provider APIs:

- OpenAI ↔ Gemini, Claude, Codex, Antigravity
- Claude ↔ Gemini, OpenAI, Codex
- Gemini ↔ Claude, OpenAI, Codex, Gemini-CLI

Preserves special features (thinking, tools, safety settings) and supports streaming.

### 4. Handler System (`sdk/api/handlers/`)

- **BaseAPIHandler**: Common functionality (error handling, streaming, client management)
- **Protocol-Specific Handlers**:
  - `openai/`: ChatCompletions, Completions, Responses
  - `claude/`: Messages, TokenCounting
  - `gemini/`: GeminiHandler, GeminiModels

### 5. Watcher System (`internal/watcher/`)

- Monitors `config.yaml` for changes and applies without restart
- Watches auth directory for new/modified/deleted token files
- Dispatches reload events to runtime components

### 6. Token Storage Backends (`internal/store/`)

| Backend | Description |
|---------|-------------|
| File Store | Local JSON files in `~/.cli-proxy-api/` |
| PostgreSQL | Distributed database storage |
| Git Store | Version-controlled credential storage |
| Object Store | S3-compatible cloud storage |

## API Endpoints

| Endpoint | Protocol |
|----------|----------|
| `/v1/chat/completions` | OpenAI-compatible chat |
| `/v1/models` | List available models |
| `/v1/messages` | Claude-compatible messages |
| `/v1beta/models/*` | Gemini-compatible endpoints |
| `/management/*` | Management API (requires secret key) |

## Request Flow

1. **Request enters** Gin engine through middleware chain
2. **AuthMiddleware** validates requester via AccessManager
3. **Router** matches request to appropriate protocol handler
4. **Handler** parses JSON body and validates schema
5. **Translator** converts request to target provider format
6. **Core Auth Manager** selects best available credential
7. **Executor** makes upstream API call
8. **Translator** converts response back to client format
9. **Response** streams or returns to client

## Credential Selection Strategies

| Strategy | Behavior |
|----------|----------|
| **Round-Robin** | Distributes load evenly across accounts |
| **Fill-First** | Uses one credential until quota exceeded, then falls back |

## Configuration

**Sources (Priority Order):**
1. PostgreSQL/Git/Object Store backend (if configured)
2. `config.yaml` file
3. Environment variables (`.env` file)
4. Hardcoded defaults

**Hot-Reload**: Config changes are watched and applied without server restart.

## SDK Embedding

External programs can embed the proxy as a library:

```go
import "github.com/router-for-me/CLIProxyAPI/v6/sdk/cliproxy"

cfg, _ := config.LoadConfig("config.yaml")
svc, _ := cliproxy.NewBuilder().
    WithConfig(cfg).
    WithConfigPath("config.yaml").
    Build()
svc.Run(ctx)
```

## Technology Stack

- **Framework**: Gin (HTTP web framework)
- **Configuration**: YAML parsing (`gopkg.in/yaml.v3`)
- **Logging**: Logrus structured logging
- **OAuth**: `golang.org/x/oauth2`
- **Storage**: PostgreSQL, Git, S3-compatible object storage
- **Testing**: Go testing package with integration tests

## Build and Run

```bash
# Build
go build -o cliproxy ./cmd/server

# Run
./cliproxy

# OAuth logins
./cliproxy -login              # Google/Gemini
./cliproxy -claude-login       # Claude
./cliproxy -codex-login        # OpenAI Codex
./cliproxy -qwen-login         # Qwen
```
