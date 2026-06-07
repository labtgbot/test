# Repository Analysis Report — Issue #17

**Date:** 2026-06-07
**Branch:** issue-17-95cb861efcc9
**Repository:** labtgbot/test

---

## 1. Executive Summary

The repository is a Telegram bot application built in Python that bridges
Telegram users with Claude AI through a self-hosted proxy
([free-claude-code](https://github.com/labtgbot/free-claude-code)). The bot
supports text, images, documents, and voice messages, maintains per-user
conversation histories, and can run in polling or webhook modes. The codebase
is modular, uses modern async Python (aiogram 3.x, FastAPI, httpx), and is
packaged for Docker deployment.

---

## 2. Architecture Overview

### Technology Stack

| Layer             | Technology                          |
|-------------------|-------------------------------------|
| Bot framework     | aiogram 3.3.0                       |
| HTTP / API server | FastAPI 0.104.1 + uvicorn 0.24.0 |
| HTTP client       | httpx 0.27.0                        |
| Configuration     | pydantic-settings 2.1.0            |
| Logging           | structlog 23.2.0                    |
| Image handling    | Pillow 10.1.0                       |
| PDF parsing       | PyPDF2 3.0.1                        |
| DOCX parsing      | python-docx 1.1.0                   |
| Test framework    | pytest + pytest-asyncio             |
| Optional (voice)  | openai-whisper, ffmpeg-python       |

### Architecture Diagram

```
Telegram API
    │
    ▼
FastAPI ── Webhook ──► Dispatcher ── Middlewares
(uvicorn)    │                    │
    │       ▼                    ▼
    │   Polling              Handlers
    │                            │
    │       └────────────────────┘
    │                    │
    │              ClaudeProxyClient
    │                    │
    │                    ▼
    │           free-claude-code proxy
    │                    │
    │                    ▼
    │           Anthropic-compatible API
    │
    ▼
/health  endpoint
```

### Module Structure

```
bot/
├── main.py            FastAPI + aiogram lifecycle
├── config.py          Pydantic settings (env vars)
├── middleware/
│   ├── logging.py     Structured JSON logging per update
│   └── rate_limit.py  Per-user sliding-window rate limiter
├── handlers/
│   ├── commands.py    /start, /help, /model, /settings, /clear
│   ├── chat.py        Main text + multimedia handler (core logic)
│   └── inline.py      Basic inline query placeholder
├── services/
│   └── claude_proxy.py  HTTP client for free-claude-code API
└── utils/
    ├── storage.py     In-memory per-user histories + settings
    └── media.py       Voice transcription (Whisper) + document extraction
```

---

## 3. Component Analysis

### 3.1 `bot/config.py`

Uses `pydantic_settings.BaseSettings` to load all configuration from
environment variables (and an optional `.env` file). Includes a custom
field validator that parses `TELEGRAM_ALLOWED_CHAT_IDS` from a
comma-separated string into a list of integers.

Key settings:
- `FREE_CLAUDE_BASE_URL` — proxy endpoint (default: `http://localhost:8082`)
- `FREE_CLAUDE_AUTH_TOKEN` — bearer token for the proxy
- `FREE_CLAUDE_DEFAULT_MODEL` — model identifier (e.g. `nvidia_nim/z-ai/glm4.7`)
- `TELEGRAM_BOT_TOKEN` — Telegram bot token from BotFather
- `TELEGRAM_WEBHOOK_URL` — optional webhook URL; absent → polling mode
- `TELEGRAM_GUEST_MODE_ENABLED` — whether group chats require `@bot` mention
- `RATE_LIMIT_REQUESTS_PER_MINUTE` — per-user throughput cap

### 3.2 `bot/main.py`

Application entry point. Creates a FastAPI app and an aiogram `Dispatcher`,
registers middlewares and routers, and exposes two HTTP endpoints:

- `POST /webhook` — receives Telegram updates, validates the
  `X-Telegram-Bot-Api-Secret-Token` header (if configured), deserialises
  the JSON payload into an `aiogram.types.Update`, and feeds it to the
  dispatcher.
- `GET /health` — health-check endpoint.

On startup the bot either registers a webhook or launches a background
polling task. On shutdown it cancels polling and closes the HTTP session.

### 3.3 `bot/middlewares/logging.py`

An aiogram `BaseMiddleware` that writes a structured JSON log line for every
incoming update (messages, callback queries, inline queries). Uses
`structlog` with ISO timestamps.

### 3.4 `bot/middlewares/rate_limit.py`

Per-user sliding-window rate limiter. Stores request timestamps in a
`collections.deque` per user. If the user has already sent N requests in the
last 60 seconds (configurable), the middleware short-circuits the handler and
optionally replies with a rate-limit notice.

### 3.5 `bot/handlers/commands.py`

Five slash-command handlers:

| Command   | Behaviour                                              |
|-----------|--------------------------------------------------------|
| `/start`  | Sends a welcome message                                |
| `/help`   | Lists available commands and supported input types     |
| `/model`  | Without args — shows current model + list of available models. With arg — sets per-user model |
| `/settings`| Displays current user settings (model, streaming, etc.)|
| `/clear`  | Clears the calling user's conversation history         |

Model selection is persisted per-user via `storage.set_setting`. Model list
is retrieved directly from the proxy (`GET /v1/models`).

### 3.6 `bot/handlers/chat.py`

The core handler. Triggered on `text`, `photo`, `voice`, or `document`
messages. Behaviour depends on whether the bot is in **guest mode** (in
group/supergroup chats users must mention `@botusername` or reply to a bot
message; otherwise history is not included).

Workflow:

1. Determine guest-mode status.
2. If not guest, load conversation history from storage.
3. Download media (photo / voice / document) and convert it to the content
   block format expected by the Anthropic Messages API:
   - **Images** — base64-encoded JPEG with MIME type.
   - **Voice** — transcribed via `whisper` (optional) into plain text.
   - **Documents** — extracted via `PyPDF2` or `python-docx` into plain text.
4. Call `ClaudeProxyClient.send_message` (streaming or non-streaming).
5. Reply to the user (streaming edits the same message in-place).
6. Save user + assistant turns to conversation history (non-guest only).

Streaming uses the SSE `data:` line format and watches for
`content_block_delta` / `text_delta` events, updating the Telegram message
as tokens arrive.

### 3.7 `bot/handlers/inline.py`

Registers a single inline query result that tells users to send a direct
message. Placeholder behaviour.

### 3.8 `bot/services/claude_proxy.py`

An async HTTP client (`httpx.AsyncClient`) for the free-claude-code proxy.
Three public methods:

- `list_models()` — `GET /v1/models`, handles both Anthropic
  (`models[]`) and OpenAI (`data[]`) response shapes.
- `count_tokens()` — `POST /v1/messages/count_tokens` with an optional
  `model` field.
- `send_message()` — `POST /v1/messages`, returns either a full JSON
  response or an async iterator over SSE chunks (when `stream=True`).

Authentication is a `Bearer` token plus the `anthropic-version: 2023-06-01`
header.

### 3.9 `bot/utils/storage.py`

`MemoryStorage` holds per-user conversation histories (FIFO list, max 20
turns) and per-user settings (key/value dict). A module-level singleton
instance (`storage`) is imported by handlers.

```python
storage = MemoryStorage()
```

Data is volatile — lost on process restart.

### 3.10 `bot/utils/media.py`

Two blocking functions executed in a thread pool to avoid blocking the
event loop:

- `transcribe_voice(audio_data)` — saves bytes to a `.ogg` temp file,
  loads the `whisper` "base" model (cached after first call), transcribes,
  removes the temp file.
- `extract_document_text(mime_type, data)` — dispatches to `PyPDF2`,
  `python-docx`, or a plain-text decode depending on MIME type.

Both return an empty string on any failure (broad `except Exception`).

---

## 4. Data Flow (End-to-End)

```
User sends message in Telegram
    │
    ▼  Telegram sends HTTPS POST to webhook URL
FastAPI /webhook endpoint
    │  validates secret token, parses JSON → aiogram Update
    ▼
aiogram Dispatcher
    │  runs LoggingMiddleware, RateLimitMiddleware
    ▼
handler (commands / chat / inline)
    │
    ├─ commands ── replies directly, no external call
    │
    └─ chat ── downloads media → builds content blocks
              │
              ▼
         ClaudeProxyClient (httpx)
              │  POST /v1/messages  ←→  free-claude-code proxy
              ▼
         streaming / non-streaming response
              │
              ▼
         reply sent to Telegram user
              │
              ▼
         (if not guest) history saved to MemoryStorage
```

---

## 5. Testing

Location: `tests/unit/`

| File                       | Focus                                  | Tests  |
|----------------------------|----------------------------------------|--------|
| `test_config.py`           | Settings parsing (chat IDs, booleans)  | 4      |
| `test_storage.py`          | CRUD, FIFO trimming, defaults          | 4      |
| `test_claude_proxy.py`     | Model listing, non-streaming, streaming | 3 (+1 incomplete) |

`conftest.py` provides shared fixtures for `test_settings` and `storage`.
There are no integration tests or handler-level tests.

Notable gap: `test_list_models_openai_format` is a stub that never asserts
anything. The `storage` fixture in `conftest.py` imports `MemoryStorage`
without an explicit import statement (relies on side-effect imports).

---

## 6. Configuration Reference

### Environment variables (`.env.example`)

| Variable                           | Required | Default              | Description                      |
|------------------------------------|----------|----------------------|----------------------------------|
| `FREE_CLAUDE_BASE_URL`             | yes      | —                    | Proxy URL                        |
| `FREE_CLAUDE_AUTH_TOKEN`           | yes      | —                    | Bearer token for proxy           |
| `FREE_CLAUDE_DEFAULT_MODEL`        | yes      | `nvidia_nim/.../glm4.7` | Default Claude model          |
| `FREE_CLAUDE_TIMEOUT_SECONDS`      | no       | 120                  | HTTP timeout                     |
| `FREE_CLAUDE_STREAMING_ENABLED`    | no       | true                 | Token streaming UX               |
| `TELEGRAM_BOT_TOKEN`               | yes      | —                    | Bot token from BotFather         |
| `TELEGRAM_WEBHOOK_URL`             | no       | —                    | Set for webhook mode             |
| `TELEGRAM_GUEST_MODE_ENABLED`      | no       | true                 | Require @mention in groups       |
| `TELEGRAM_ALLOWED_CHAT_IDS`        | no       | []                   | Comma-separated allowed chat IDs |
| `API_SECRET_TOKEN`                 | no       | —                    | Webhook secret header            |
| `RATE_LIMIT_REQUESTS_PER_MINUTE`   | no       | 60                   | Per-user rate cap                |
| `LOG_LEVEL`                        | no       | INFO                 | structlog level                  |

---

## 7. Deployment

### Docker Compose (production)

Two services:
1. **telegram-bot-agent** — built from Dockerfile; depends on
   `free-claude-code`; exposes port 8000.
2. **free-claude-code** — uses the pre-built image
   `ghcr.io/labtgbot/free-claude-code:latest`; exposes port 8082; model
   defaults to `nvidia_nim/z-ai/glm4.7`.

### Dockerfile

Multi-step: `python:3.11-slim` → install requirements → copy source →
`uvicorn bot.main:app --host 0.0.0.0 --port 8000`.

### Systemd (production, host)

A template unit file is described in the README. The application runs as
a regular systemd service started after network.target.

---

## 8. Limitations and Risks

| #  | Issue                                | Impact                                  |
|----|--------------------------------------|-----------------------------------------|
| 1  | In-memory storage                    | History lost on restart; no horizontal scaling |
| 2  | Broad `except Exception` in media    | Silent failures — user sees no error    |
| 3  | Voice transcription optional dep not in `requirements.txt` | Feature may fail unexpectedly |
| 4  | No retry / circuit breaker on proxy calls | Transient errors break the user flow |
| 5  | Message text logged in plaintext     | Potential PII / sensitive content leak  |
| 6  | No token-budget-based history truncation | Context window may overflow silently   |
| 7  | `test_list_models_openai_format` is a stub | Incomplete test coverage        |
| 8  | FIFO trimming is turn-based, not token-based | May still exceed model context   |

---

## 9. Conclusion

The repository is a well-structured, production-oriented Telegram bot with a
clean layered architecture. Core functionality (command handling, chat with
media support, streaming, rate limiting, structured logging) is implemented
and working.

The main gaps are operational: in-memory storage prevents horizontal scaling
and causes data loss on restart, media-related code swallows exceptions
silently, and the optional Whisper dependency is not declared in
`requirements.txt`. Addressing these and adding integration tests for the
handler layer would bring the project to a robust production-ready state.
