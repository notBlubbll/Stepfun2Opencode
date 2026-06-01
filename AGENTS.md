# StepFun2Opencode — Developer Guide

## Project Structure

```
STEP-PROXY/
├── proxy.js              # Main proxy implementation
├── dashboard.html        # Liquid glass dashboard with stats UI
├── .config/
│   └── config.json       # Runtime configuration
├── .cache/               # Response cache directory
├── package.json          # Project metadata (MIT, no deps)
├── start.cmd             # Auto-detect launcher (Bun preferred, Node fallback)
├── start-node.cmd        # Node.js-only launcher
├── skills.md             # Opencode provider configuration reference
├── README.md             # User documentation
└── AGENTS.md             # This file
```

## Key Components

### 1. Constants & Config

- `STEP_API_BASE` — `https://api.stepfun.ai`
- `STEP_PLAN_BASE` — `https://api.stepfun.ai/step_plan` (dedicated Step Plan endpoint for chat completions)
- `STEP_MODELS_URL` — `https://api.stepfun.ai/step_plan/v1/models`
- `API_KEY_ENV_VAR` — `STEP_API_KEY`
- `loadConfig()` — Loads `.config/config.json` with env var overrides
- `saveConfig()` — Writes config back to `.config/config.json`
- `parseDuration()` — Parses duration strings like `15m`, `6h`, `30s`

### 2. UpstreamClient

- `headers(stream)` — Returns Bearer token + Content-Type/Accept headers
- `getUserInfo()` — `GET /v1/models` with 10s timeout to validate API key
- `chatCompletions(body)` — `POST /step_plan/v1/chat/completions` (Step Plan endpoint, streaming-aware)
- `getAccountInfo()` — `GET /v1/accounts` with 10s timeout to fetch balance info

### 3. Model Registry

- `STEP_MODELS` — Hardcoded array of 6 model IDs (fallback)
- `fetchRemoteModels()` — Fetches from `STEP_MODELS_URL` with 5-minute TTL cache
- Models: step-3.5-flash, step-3.5-flash-2603, step-3.7-flash, stepaudio-2.5-tts, stepaudio-2.5-asr, step-image-edit-2

### 4. Utility Functions

- `cloneMap()` / `cloneSlice()` — Deep clone objects/arrays
- `normalizeToolSchemas(tools)` — Entry point for `$ref` resolution in tool schemas
- `extractDefinitions(schema)` — Merges `definitions` + `$defs`
- `normalizeSchemaMap(node, defs, maxDepth)` — Recursive `$ref` resolver (max depth: 12)
- `readBodyText(body)` — Handles Node streams, web ReadableStream, async iterables
- `extractUserPrompt(payload)` — Returns last user message text for logging
- `fingerprintPayload(payload)` — MD5 hash of first user message for session tracking

### 5. HTTP Handlers

- `authorized(req)` — Checks `x-api-key` header or `Authorization: Bearer` against `config.apiKeys`
- `readBody(req)` — Buffers incoming request body to string
- `writeJSON(res, statusCode, payload)` — JSON response with error-safe write
- `writeOpenAIError()` — OpenAI error format
- `handleHealthz(req, res)` — Returns uptime, API key validity, models count, runtime info, account info
- `handleModels(req, res)` — OpenAI-format model list
- `handleChatCompletions(req, res)` — Parses body, calls `proxyChatRequest`
- `handleAccountInfo(req, res)` — Proxies `GET /v1/accounts` from StepFun with 60s cache
- `proxyChatRequest(res, payload, model)` — Core proxy: clone payload, normalize tools, forward to upstream

### 6. Request Router

Routes by pathname:
- `/` or `/dashboard` → Serve `dashboard.html`
- `/api/config` (GET/POST) → Config read/write
- `/api/validate` (GET) → Validate API key
- `/api/models` (GET) → Model list
- `/api/keys` (GET/POST) → Multi-key CRUD (add/update/delete with `{name, token}`)
- `/api/account` (GET) → StepFun account info (balance, cash, voucher)
- `/api/cache` (GET/DELETE) → Cache stats/clear
- `/healthz` → Health check (includes account info)
- `/v1/models` → OpenAI models
- `/v1/chat/completions` → OpenAI chat

### 7. Session Tracking & Key Rotation

- `currentTokenIndex` — Module-level round-robin index
- `globalSessionCounter` — Monotonically incrementing session ID for each new conversation
- `conversationMap` — `Map<fingerprint, { tokenIndex, requestCount, sessNum }>` — tracks which token a conversation is pinned to
- `fingerprintPayload(payload)` — MD5 hash of the first user message (skips auto title prompts, strips `[label]` prefix) to identify conversation threads
- `detectSessionSignal(payload)` — Core session logic:
  1. Computes fingerprint from first user message
  2. If fingerprint exists in `conversationMap` → pins to that token (sticky session)
  3. If new fingerprint → rotates to next key round-robin, stores mapping, stamps message with `[KeyName|sessN]`
- Console logs use `HH:MM:SS [Session#N>KeyName]-[model]-"actual prompt"` format

### 8. Account Info

- `getAccountInfo()` — Fetches `GET /v1/accounts` from StepFun API with 60s in-memory cache
- `accountCache` — `{ data, time }` object for caching account responses
- Returns: `{ object, type, balance, total_cash_balance, total_voucher_balance }`
- Exposed via `GET /api/account` endpoint
- Also included in `/healthz` response under `account` key

### 9. Opencode Config

- `setupOpencodeConfig()` — Writes provider config to multiple paths:
  1. `~/.opencode/opencode.json` (Win32 priority)
  2. `~/.config/opencode/opencode.json`
  3. `C:\Windows\System32\config\systemprofile\.opencode\opencode.json` (Win32)
- Creates `openconfig.b4stepfun.json` backup before first edit
- Provider key: `stepfun`, using `@ai-sdk/openai-compatible`

### 10. Dashboard (dashboard.html)

- **Liquid Glass Engine** — Canvas-generated displacement maps with refraction profiles
- **SVG Filter Pipeline** — `feGaussianBlur` → `feDisplacementMap` → `feColorMatrix` → `feComposite` → `feBlend`
- **Account Stats** — 4 stat cards: Balance, Cash Balance, Voucher Balance, Cache Hits
- **Key Manager Modal** — Inline add/edit/delete for multiple API keys
- **Model Tags** — Toggle models on/off with checkbox UI
- **SS Mode** — `token-blurred` CSS class (blur on hover)
- **Auto-refresh** — Health check every 15s, account info every 60s
- **Collapsible Sections** — Models, API Key, Quick Actions, Environment, Proxy Configuration

## Request Lifecycle

```
Client request arrives
    ↓
Check API key authorization (if apiKeys configured)
    ↓
Route by pathname → handler
    ↓
Parse + validate request body
    ↓
Detect session signal (fingerprint first user msg)
    ↓
  ├─ Known fingerprint → pin to same token (sticky)
  └─ New fingerprint → rotate to next key, store mapping
    ↓
Clone payload, normalize tool schemas
    ↓
Forward to upstream api.stepfun.ai
    ↓
Success → pipe/buffer response (stream or JSON), log done
Error   → parse upstream error, return formatted response
```

## Startup Sequence

1. `loadConfig()` — Load `.config/config.json` + env var overrides
2. `UpstreamClient` — Initialize HTTP client
3. `validateApiKey()` — Verify via `/v1/models`
4. `fetchRemoteModels()` — Fetch models from StepFun API
5. `setupOpencodeConfig()` — Write/update opencode provider config
6. `http.createServer(handleRequest).listen(port)` — Start HTTP server

## Response Caching

LRU cache for non-streaming LLM responses:

- **Key**: MD5 hash of `(model + stream_flag + system + messages + tools)`
- **TTL**: Configurable via `CACHE_TTL` (default `60s`)
- **Max size**: Configurable via `CACHE_MAX_SIZE` (default 100 entries)
- **Disable**: Set `CACHE_ENABLED=false`
- **Stats**: `GET /api/cache` — hits, misses, evictions, size
- **Clear**: `DELETE /api/cache`
- **Excluded**: Streaming requests are never cached. Only 2xx non-streaming responses are stored.

## Testing

```bash
# Syntax check
node --check proxy.js

# Start proxy
node proxy.js

# Test endpoints
curl http://localhost:8080/healthz
curl http://localhost:8080/v1/models
curl http://localhost:8080/api/models
curl http://localhost:8080/api/account

# Test chat completion
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"step-3.5-flash","messages":[{"role":"user","content":"Hello"}]}'

# Test streaming
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"step-3.5-flash","stream":true,"messages":[{"role":"user","content":"Hello"}]}'
```

## Security

- API keys for proxy authentication (optional, via `API_KEYS` config)
- Keys masked in `/api/config` responses (`substring(0,10) + '...'`)
- No token logging in request logs
- Config file should be `.gitignore`'d
- SS Mode (`.token-blurred`) in dashboard obscures tokens for screenshots
