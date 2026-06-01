# StepFun2Opencode

OpenAI-compatible proxy server for [StepFun AI](https://platform.stepfun.ai), providing access to StepFun LLM models through a unified API. Zero external dependencies — uses only Node.js built-in modules.

## Features

- **OpenAI-Compatible API** — Standard `/v1/chat/completions` and `/v1/models` endpoints
- **Streaming Support** — SSE streaming for chat completions
- **Tool Schema Normalization** — Resolves `$ref` and `$defs` in tool schemas before forwarding
- **Dashboard UI** — Liquid glass effects, model/key management, real-time plan & usage stats
- **Auto-Config** — Automatically configures opencode provider on startup
- **Dynamic Model Fetch** — Fetches available models from `https://api.stepfun.ai/step_plan/v1/models`
- **Account Info** — Real-time balance, cash balance, and voucher balance from StepFun API
- **Step Plan Integration** — Live plan name, 5h/weekly usage %, and reset timers from platform.stepfun.ai
- **Response Caching** — LRU cache for non-streaming responses
- **Multi-Key Support** — Rotate between multiple StepFun API keys
- **Zero Dependencies** — No npm packages required

## Available Models

| Model | Description |
|-------|-------------|
| `step-3.5-flash` | StepFun 3.5 Flash |
| `step-3.5-flash-2603` | StepFun 3.5 Flash (2603) |
| `step-3.7-flash` | StepFun 3.7 Flash (reasoning tiers: low/medium/high) |
| `step-tts-2` | StepFun TTS v2 |
| `stepaudio-2.5-tts` | StepFun Audio 2.5 TTS |
| `stepaudio-2.5-asr` | StepFun Audio 2.5 ASR |
| `step-image-edit-2` | StepFun Image Edit v2 |

Models are dynamically fetched from `https://api.stepfun.ai/step_plan/v1/models` on startup.

## Quick Start

```bash
# Clone and start (zero deps — no npm install needed)
cd STEP-PROXY
node proxy.js

# Or use launcher (auto-detects Bun, falls back to Node)
start.cmd

# Open dashboard
open http://localhost:8080
```

## Authentication

Get a StepFun API key from [platform.stepfun.ai](https://platform.stepfun.ai).

Add to `.config/config.json`:

```json
{
  "API_KEY": "your-stepfun-api-key"
}
```

Or set environment variable:

```bash
set STEP_API_KEY=your-stepfun-api-key
node proxy.js
```

## Step Plan vs Standard API

StepFun has two separate systems:

| | Standard API | Step Plan |
|--|-------------|-----------|
| **Base URL** | `https://api.stepfun.ai/v1` | `https://api.stepfun.ai/step_plan/v1` |
| **Billing** | Prepaid balance | Subscription quota (5h/weekly limits) |
| **Chat endpoint** | `/v1/chat/completions` | `/step_plan/v1/chat/completions` |
| **Models** | All models | step-3.5-flash, step-3.5-flash-2603, step-3.7-flash |

The proxy automatically routes chat completions to the Step Plan endpoint. Using the standard endpoint with a Step Plan subscription returns `quota_exceeded` errors.

## Configuration

Edit `.config/config.json` or set environment variables:

| Key | Description | Default |
|-----|-------------|---------|
| `LISTEN_ADDR` | Proxy listen address | `127.0.0.1:8080` |
| `UPSTREAM_BASE_URL` | StepFun API URL (for models/accounts) | `https://api.stepfun.ai` |
| `API_KEY` | StepFun API key | — |
| `REQUEST_TIMEOUT` | Upstream request timeout | `15m` |
| `API_KEYS` | Client API keys for proxy auth | `[]` (open access) |
| `TOKENS` | Array of `{name, token}` for multi-key support | auto-populated |
| `OASIS_TOKEN` | Oasis JWT token for platform.stepfun.ai API | — |
| `OASIS_WEBID` | Oasis web device ID | — |
| `CACHE_TTL` | Response cache TTL | `60s` |
| `CACHE_MAX_SIZE` | Max cached responses | `100` |
| `CACHE_ENABLED` | Enable response caching | `true` |

### Multi-Key Management

The proxy supports multiple StepFun API keys. Set `TOKENS` in config:

```json
{
  "TOKENS": [
    { "name": "Key 1", "token": "your-key-1" },
    { "name": "Key 2", "token": "your-key-2" }
  ]
}
```

Manage keys via the **Dashboard → Manage Keys** modal (inline add/edit/delete).

### Key Rotation & Session Tracking

The proxy automatically rotates tokens across conversations using **fingerprint-based session tracking**.
Each conversation is identified by an MD5 hash of the first user message (skipping auto title prompts).
Follow-up requests (tool calls, continuations) in the same conversation are pinned to the same token
automatically. A global session counter increments for each new conversation.

```
Conversation A (fingerprint: a1b2c3) → sess1 → Token 1
├── tool call follow-up → Token 1 (same fingerprint)
└── user follow-up → Token 1 (same fingerprint)
Conversation B (fingerprint: d4e5f6) → sess2 → Token 2 (next in pool)
```

Console logs show:
```
14:30:15 [Session#1>Key1]-[step-3.5-flash]-"user prompt"
14:30:15 [Session#1>Key1]-[step-3.5-flash]-done:3996ms
```

### Proxy API Keys

By default the proxy is open access. To restrict access, set `API_KEYS`:

```json
{
  "API_KEYS": ["my-secret-key-1", "my-secret-key-2"]
}
```

Clients must include the key:

```bash
curl -H "x-api-key: my-secret-key-1" http://localhost:8080/v1/models
curl -H "Authorization: Bearer my-secret-key-1" http://localhost:8080/v1/models
```

## Usage

### OpenAI-Compatible

```javascript
import OpenAI from 'openai';
const client = new OpenAI({
  baseURL: 'http://localhost:8080/v1',
  apiKey: 'not-needed'
});
const response = await client.chat.completions.create({
  model: 'step-3.5-flash',
  messages: [{ role: 'user', content: 'Hello!' }]
});
```

### opencode Integration

The proxy auto-configures opencode on startup. Restart opencode after starting the proxy, then select the `stepfun` provider.

## Dashboard

Access at `http://localhost:8080`:

- **Plan & Usage Stats** — Plan name (hover for dates), 5h/weekly usage % with reset timers (auto-refreshes every 30s)
- **Cache Stats** — Real-time cache hits and performance
- **API Key Status** — Online/Offline indicator
- **SS Mode** — Blur sensitive tokens for screenshots
- **Liquid Glass Effects** — Canvas-generated SVG displacement maps
- **Model Management** — Toggle models on/off
- **Key Manager** — Add/edit/delete API keys with inline editing
- **Collapsible Sections** — Models, API Key, Quick Actions, Environment, Proxy Configuration

## API Endpoints

### Core API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/healthz` | Health check with API key status, uptime, and account info |
| `GET` | `/v1/models` | OpenAI models list |
| `POST` | `/v1/chat/completions` | OpenAI chat completions (streaming supported) |

### Management API

| Method | Path | Description |
|--------|------|-------------|
| `GET` / `POST` | `/api/config` | Read/write proxy configuration |
| `GET` | `/api/validate` | Validate API key |
| `GET` | `/api/models` | List available model IDs |
| `GET` | `/api/bg` | Bing wallpaper image (cached daily) |
| `GET` / `POST` | `/api/keys` | Multi-key CRUD (add/update/delete) |
| `GET` | `/api/account` | StepFun account info (balance, cash, voucher) |
| `GET` | `/api/step-plan-status` | Platform plan + usage rates (60s cache) |
| `GET` / `DELETE` | `/api/cache` | View/clear response cache |

## Architecture

```
proxy.js
├── Config System         — JSON + env vars + API key validation
├── UpstreamClient        — HTTP client for StepFun API
│   ├── getUserInfo()     — GET /v1/models (validate key)
│   ├── chatCompletions() — POST /v1/chat/completions
│   ├── getAccountInfo()  — GET /v1/accounts (balance info)
│   ├── getStepPlanStatus()— POST platform QueryStepPlanRateLimit (usage)
│   └── getPlanStatus()   — POST platform GetStepPlanStatus (plan)
├── Tool Schema Norm.     — $ref resolution and schema normalization
├── HTTP Handlers         — OpenAI + management + plan status endpoints
├── Request Router        — Pathname-based routing
├── Opencode Config       — Auto-configures opencode provider
└── Server Startup        — Validation, config write, listen

dashboard.html
├── Liquid Glass Engine   — Canvas-based displacement/specular maps
├── Plan & Usage Cards    — Plan name, 5h/weekly usage %, reset timers
├── Model Management      — Toggle models on/off
├── Key Manager           — Add/edit/delete API keys
├── Bing Wallpaper        — Daily rotating background
├── Cache Stats           — Real-time cache performance
└── Configuration Forms   — Listen addr, upstream URL, timeout
```
proxy.js
├── Config System         — JSON + env vars + API key validation
├── UpstreamClient        — HTTP client for StepFun API
│   ├── getUserInfo()     — GET /v1/models (validate key)
│   ├── chatCompletions() — POST /step_plan/v1/chat/completions
│   └── getAccountInfo()  — GET /v1/accounts (balance info)
├── Decompression         — Brotli/gzip/deflate auto-decode
├── Tool Schema Norm.     — $ref resolution and schema normalization
├── HTTP Handlers         — OpenAI + management + account + wallpaper
├── Request Router        — Pathname-based routing
├── Session Tracking      — Fingerprint-based sticky sessions
├── Opencode Config       — Auto-configures opencode provider
└── Server Startup        — Validation, config write, listen

dashboard.html
├── Liquid Glass Engine   — Canvas-based displacement/specular maps
├── Account Stats Cards   — Balance, Cash Balance, Voucher Balance
├── Bing Wallpaper        — Daily rotating backgrounds
├── Model Management      — Toggle models on/off
├── Key Manager           — Add/edit/delete API keys
├── Cache Stats           — Real-time cache performance
└── Configuration Forms   — Listen addr, upstream URL, timeout
```

## Dependencies

No external npm dependencies — uses Node.js built-in modules only: `fs`, `path`, `os`, `http`, `https`, `url`, `crypto`, `zlib`.

## License

MIT
