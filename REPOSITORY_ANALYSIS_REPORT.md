# Repository Analysis Report

**Project:** Telegram Claude Agent
**Date:** 2026-05-17
**Branch:** issue-5-c7ee3a8d80c0
**Issue:** #5 - Аназиз репо (Deep repository analysis)

---

## Executive Summary

This repository implements a professional Telegram bot that provides access to Claude AI capabilities via the [free-claude-code](https://github.com/labtgbot/free-claude-code) proxy service. The bot is built with modern Python async frameworks, follows good separation of concerns, and includes comprehensive features such as streaming responses, media handling, guest mode, and rate limiting.

The codebase is well-structured, modular, and production-ready with Docker deployment support. It demonstrates a clean architecture pattern with distinct layers for configuration, business logic, API integration, and presentation.

---

## 1. Project Overview

### Purpose
The Telegram Claude Agent allows users to interact with Claude AI through Telegram. It acts as a bridge between Telegram users and the Anthropic Messages API (via free-claude-code proxy), supporting rich media inputs and maintaining conversation context.

### Key Features
- Real-time streaming responses with live text updates
- Guest mode for privacy in group chats
- Image analysis and multimodal conversations
- Document processing (PDF, TXT, DOCX)
- Voice message transcription via Whisper
- Per-user conversation history management
- Model selection and user settings
- Rate limiting (per user, per minute)
- Structured JSON logging
- Webhook and polling modes for Telegram
- Health check endpoint for monitoring
- Docker containerization

### Tech Stack
| Component | Technology | Version |
|-----------|------------|---------|
| Bot Framework | aiogram 3.x | 3.3.0 |
| HTTP Server | FastAPI | 0.104.1 |
| ASGI Server | uvicorn[standard] | 0.24.0 |
| HTTP Client | httpx | 0.27.0 |
| Configuration | pydantic-settings | 2.1.0 |
| Validation | pydantic | (via settings) |
| Logging | structlog | 23.2.0 |
| Media - Images | Pillow | 10.1.0 |
| Media - PDF | PyPDF2 | 3.0.1 |
| Media - DOCX | python-docx | 1.1.0 |
| Media - Voice | openai-whisper | (optional) |
| Form parsing | python-multipart | 0.0.6 |
| Testing | pytest, pytest-asyncio | - |

### Requirements
- Python 3.11 or higher
- Telegram bot token from [@BotFather](https://t.me/botfather)
- Running free-claude-code instance (default: http://localhost:8082)

---

## 2. Repository Structure

```
telegram-claude-agent/
├── bot/
│   ├── __init__.py
│   ├── main.py                 # FastAPI app, lifecycle, webhook endpoint
│   ├── config.py               # Pydantic settings class
│   ├── middlewares/
│   │   ├── __init__.py
│   │   ├── logging.py          # Structured logging middleware
│   │   └── rate_limit.py       # Rate limiting middleware
│   ├── handlers/
│   │   ├── __init__.py
│   │   ├── commands.py         # /start, /help, /model, /settings, /clear
│   │   ├── chat.py             # Main chat + media message handler
│   │   └── inline.py           # Inline query handler
│   ├── services/
│   │   ├── __init__.py
│   │   └── claude_proxy.py     # Client for free-claude-code API
│   └── utils/
│       ├── __init__.py
│       ├── storage.py          # In-memory conversation storage
│       └── media.py            # Voice transcription, document extraction
├── tests/
│   ├── __init__.py
│   ├── conftest.py             # Shared fixtures
│   └── unit/
│       ├── __init__.py
│       ├── test_config.py      # Config validation tests
│       ├── test_storage.py     # Storage class tests
│       └── test_claude_proxy.py # ClaudeProxyClient tests
├── .env.example                # Environment variables template
├── .gitignore
├── CLAUDE_CODE_CAPABILITIES.md
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
├── README.md                   # Documentation (comprehensive)
└── REPOSITORY_ANALYSIS_REPORT.md (this file)
```

---

## 3. Architecture & Design

### 3.1 High-Level Architecture

```
┌─────────────────┐
│   Telegram      │
│   (users)       │
└───────┬─────────┘
        │ Telegram Bot API (webhook or polling)
        ▼
┌─────────────────────────────────────────────┐
│         FastAPI + aiogram                   │
│  ┌───────────────────────────────────────┐ │
│  │ Entry Point: bot/main.py              │ │
│  │  • FastAPI app with /webhook endpoint│ │
│  │  • aiogram Dispatcher with routers   │ │
│  │  • Startup/shutdown lifecycle        │ │
│  └───────────────────────────────────────┘ │
│  ┌───────────────────────────────────────┐ │
│  │ Middlewares (applied globally)        │ │
│  │  • LoggingMiddleware                  │ │
│  │  • RateLimitMiddleware                │ │
│  └───────────────────────────────────────┘ │
│  ┌───────────────────────────────────────┐ │
│  │ Routers (handler modules)             │ │
│  │  • commands_router                    │ │
│  │  • chat_router                        │ │
│  │  • inline_router                      │ │
│  └───────────────────────────────────────┘ │
│  ┌───────────────────────────────────────┐ │
│  │ Services                              │ │
│  │  • ClaudeProxyClient                  │ │
│  └───────────────────────────────────────┘ │
│  ┌───────────────────────────────────────┐ │
│  │ Utilities                             │ │
│  │  • MemoryStorage                      │ │
│  │  • media.py (transcription, extract) │ │
│  └───────────────────────────────────────┘ │
└─────────────────┬───────────────────────────┘
                  │ HTTP requests (async)
                  ▼
┌─────────────────────────────────────────────┐
│   free-claude-code Proxy                    │
│   (Anthropic Messages API compatible)      │
│   e.g., http://localhost:8082               │
└─────────────────────────────────────────────┘
```

### 3.2 Design Patterns

1. **Router Pattern** (aiogram): Separation of message handlers into logical routers (commands, chat, inline)
2. **Service Layer**: `ClaudeProxyClient` encapsulates external API communication
3. **Middleware Pipeline**: Cross-cutting concerns (logging, rate limiting) handled via middlewares
4. **Singleton Config**: Global `settings` instance from pydantic-settings
5. **Dependency Injection**: Not formal DI container, but objects passed explicitly (storage, client)
6. **Repository Pattern**: `MemoryStorage` as abstract storage interface (could be swapped for Redis)

---

## 4. Component Analysis

### 4.1 Configuration (`bot/config.py`)

**Class:** `Settings(BaseSettings)`

Environment variablesLoaded from `.env` file automatically.

**Fields:**
- `free_claude_base_url`: Proxy URL (required)
- `free_claude_auth_token`: Proxy Bearer token (required)
- `free_claude_default_model`: Default model ID (e.g., `claude-3-haiku-20240307`)
- `free_claude_timeout_seconds`: HTTP timeout (default 120)
- `free_claude_streaming_enabled`: Streaming toggle (default `true`)
- `telegram_bot_token`: Bot token from BotFather (required)
- `telegram_webhook_url`: HTTPS webhook URL (optional; if empty, uses polling)
- `telegram_guest_mode_enabled`: Enable guest mode in groups (default `true`)
- `telegram_allowed_chat_ids`: Comma-separated list of chat IDs for whitelist (default `[]`)
- `api_secret_token`: Secret for webhook verification (optional but recommended)
- `rate_limit_requests_per_minute`: Rate limit (default 60)
- `log_level`: Logging level (default `INFO`)

**Validator:** Custom `parse_chat_ids` converts comma-separated string to `List[int]`.

**Quality:** Clean, straightforward Pydantic usage with sensible defaults.

---

### 4.2 Main Application (`bot/main.py`)

**Responsibilities:**
- FastAPI app initialization (`app = FastAPI(...)`)
- aiogram Bot and Dispatcher setup
- Middleware registration
- Router inclusion
- Lifecycle management (startup/shutdown)
- Webhook endpoint (`POST /webhook`)
- Health check endpoint (`GET /health`)
- Polling task management

**Key Details:**
- Uses `uvicorn` as ASGI server
- On startup: caches bot username, sets webhook if configured, starts polling task otherwise
- On shutdown: cancels polling task, closes bot session
- Webhook endpoint verifies `X-Telegram-Bot-Api-Secret-Token` header if `API_SECRET_TOKEN` is set
- Parse mode set to HTML for bot responses
- Structured logging configured with JSONRenderer

**Observations:**
- Polling task is managed globally; could be encapsulated but acceptable for this scale
- Webhook secret token validation is correctly implemented
- Graceful shutdown handles polling cancellation
- Update flow: FastAPI receives → validates → feeds to Dispatcher

---

### 4.3 Claude Proxy Service (`bot/services/claude_proxy.py`)

**Class:** `ClaudeProxyClient`

Wraps the free-claude-code API with Anthropic Messages API format.

**Methods:**
- `__init__(base_url, auth_token, timeout=120)`: Creates httpx.AsyncClient
- `close()`: Closes the HTTP client
- `_auth_headers()`: Returns headers with Bearer token and anthropic-version
- `list_models()`: GET `/v1/models`; supports both `{models: [...]}` and `{data: [...]}` formats
- `count_tokens(text, model=None)`: POST `/v1/messages/count_tokens`
- `send_message(messages, model=None, stream=False, max_tokens=4096, system=None)`: Main API call
  - Returns full JSON response or async iterator for streaming
- `_stream_response(response)`: Parses Server-Sent Events (SSE) lines

**Streaming Format:** Expects lines like `data: {...}` with chunk types:
- `content_block_delta` with `delta.text`
- `message_stop` to break

**Quality:** Robust handling of both Anthropic and OpenAI-style model list responses, proper error propagation, clean resource management.

**Minor Notes:**
- `max_tokens` is hardcoded to 4096; could be configurable
- No explicit retry logic (but httpx `follow_redirects` is enabled)

---

### 4.4 Command Handlers (`bot/handlers/commands.py`)

Handlers registered via `commands_router`.

| Command | Handler | Description |
|---------|---------|-------------|
| `/start` | `cmd_start` | Welcome message |
| `/help` | `cmd_help` | List available commands and supported media |
| `/model` | `cmd_model` | Show current model + available models; or set new model |
| `/settings` | `cmd_settings` | Display user's settings (model, streaming, guest mode, rate limit) |
| `/clear` | `cmd_clear` | Clear conversation history from storage |

**Implementation Notes:**
- Uses `storage.get_setting`/`set_setting` for per-user model overrides
- `/model` without args fetches model list from proxy; caches client lifecycle within handler
- Error handling: listeners see exceptions, but some handlers use try/except to provide fallback message
- Messages use HTML parse mode for settings display

**Quality:** Simple, readable, well-documented behavior.

---

### 4.5 Chat Handler (`bot/handlers/chat.py`)

**Main Handler:** `handle_chat_message` decorated for `F.text | F.photo | F.voice | F.document`

This is the core message processor. It handles:
1. **Guest mode detection:** In group/supergroup, checks for bot mention `@username` or reply to bot's message.
2. **Conversation history:** If not guest, loads history from `storage`.
3. **Content extraction:**
   - Text → content block
   - Photo: downloads file, base64 encodes, creates image block + optional caption
   - Voice: downloads, calls `transcribe_voice`, creates text block
   - Document: downloads, calls `extract_document_text`, creates text block
4. **Claude API call:**
   - Creates `ClaudeProxyClient`
   - Chooses streaming (`handle_streaming`) or non-streaming
   - Streaming: uses placeholder "..." message, edits progressively with `edit_text`
   - Non-streaming: extracts text from `content` blocks, sends full reply
5. **Storage update:** If not guest, adds user message + assistant response to history.

**Helper: `send_reply_safely`** - Splits long messages (>4096 chars) into multiple answers.

**Streaming handler:**
- Sends initial "..." message
- Iterates chunks, looks for `content_block_delta.text_delta`
- Accumulates full text, edits message with latest content
- Stops at `message_stop`
- Catches exceptions and displays error

**Quality:**
- Good separation: detection → build messages → API → storage update
- Proper resource cleanup (`finally: await client.close()`)
- Handles multimodal content correctly
- User feedback: streaming placeholder, rate limit messages, error messages
- Robust error handling at outermost level

**Potential Improvements:**
- Base64 encoding of images done in-memory; for very large images this could use significant memory (acceptable for Telegram size limits)
- Could batch large text responses with Markdown; but 4096 limit correctly handled with splitting

---

### 4.6 Inline Handler (`bot/handlers/inline.py`)

**Handler:** `handle_inline_query`

Provides a basic inline query result. Very simple:
- Returns `InlineQueryResultArticle` with title "Telegram Claude Agent"
- Content: instructs user to send direct message

**Evaluation:** Minimal implementation suitable for placeholder. Could be expanded to show recent responses, but not critical.

---

### 4.7 Storage (`bot/utils/storage.py`)

**Class:** `MemoryStorage`

In-memory conversation and settings storage.

**Data Structures:**
- `self.histories: Dict[int, List[Dict]]` maps user_id → list of messages `{"role": "...", "content": ...}`
- `self.user_settings: Dict[int, Dict]` maps user_id → dict of settings (e.g., model override)
- `self.max_history` limits number of messages retained (default 20)

**Methods:**
- `get_history(user_id)` → list
- `add_message(user_id, role, content)` → appends, trims oldest if exceeding max history (FIFO)
- `clear_history(user_id)` → empty list
- `get_setting(user_id, key, default)` → nested get
- `set_setting(user_id, key, value)` → creates nested dict if needed

**Quality:** Simple, thread-safe? aiogram runs single event loop, so okay. For multi-worker deployment, need Redis or database.

**Note:** The storage is shared globally (`storage = MemoryStorage()`), so all users use same instance.

---

### 4.8 Media Utils (`bot/utils/media.py`)

Two main functions:

1. **`transcribe_voice(audio_data: bytes) -> str`**
   - Runs synchronous Whisper in thread pool executor
   - Saves OGG to temp file, loads Whisper "base" model (cached on first use), transcribes, deletes temp file
   - Returns empty string on failure

2. **`extract_document_text(mime_type: str, data: bytes) -> str`**
   - Runs extraction in thread pool (PDF, plain text, DOCX)
   - PDF: PyPDF2 reads all pages, concatenates text
   - TXT: decodes UTF-8 with ignore errors
   - DOCX: joins paragraphs
   - Returns empty string on unsupported mime type or failure

**Quality:** Proper async boundary via `run_in_executor`. Whisper model loaded once and reused. Temp files securely cleaned.

**Dependencies:** `openai-whisper` is optional; if not installed, transcription will fail. Could be more graceful (e.g., catch ImportError).

---

### 4.9 Logging Middleware (`bot/middlewares/logging.py`)

**Class:** `LoggingMiddleware(BaseMiddleware)`

Logs incoming updates with structured fields.

**Logged Events:**
- `Message`: user_id, username, chat_id, chat_type, text snippet
- `CallbackQuery`: user_id, data
- `InlineQuery`: user_id, query
- Others: `update_received` with type name

Uses `structlog` with JSONRenderer configured in `main.py`.

**Quality:** Minimal overhead, sufficient context for audit and debugging.

---

### 4.10 Rate Limit Middleware (`bot/middlewares/rate_limit.py`)

**Class:** `RateLimitMiddleware(BaseMiddleware)`

Per-user rate limiting in a sliding 60-second window.

**Algorithm:**
- Maintains `user_timestamps: Dict[int, deque]`
- On each request: remove timestamps older than 60 seconds
- If `len(timestamps) >= rate_limit`, reject request with rate limit message
- Otherwise append current timestamp and continue

**Scope:** applies to Message, CallbackQuery, InlineQuery events.

**Quality:** Efficient O(k) per request (k = number of recent requests by user), accurate sliding window. Good balance of simplicity and effectiveness.

**Note:** The window is fixed at 60 seconds, while rate limit config uses "requests per minute". Could be made configurable (e.g., 55-60 seconds), but acceptable.

---

## 5. API Interaction (free-claude-code)

The bot calls the proxy's Messages API endpoints:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/messages` | POST | Send a message (streaming or non-streaming) |
| `/v1/models` | GET | List available models |
| `/v1/messages/count_tokens` | POST | Token counting (used? not in current flow) |

**Authentication:** Bearer token via `Authorization: Bearer <token>` header.

**Streaming:** Server-Sent Events (SSE) with `data: {...}` lines. The client parses and yields delta chunks.

**Response Format Expectations:**
- Non-streaming: `{"content": [{"type": "text", "text": "..."}]}`
- Streaming: chunks with `type` such as `content_block_delta`, `message_stop`

The proxy is expected to be compatible with Anthropic's Messages API format.

---

## 6. Testing Coverage

### 6.1 Unit Tests

Located in `tests/unit/`.

**`test_config.py`:**
- Parsing of comma-separated chat IDs (including negative values for channels)
- Empty string handling → empty list
- Boolean parsing from string ("false", "False")
- 4 test cases total

**`test_storage.py`:**
- `test_add_and_get_history`: basic add and retrieve
- `test_clear_history`: reset functionality
- `test_max_history_trims_oldest`: FIFO trimming with max_history=3
- `test_user_settings`: get/set with defaults
- 4 test cases

**`test_claude_proxy.py`:**
- `test_list_models_anthropic_format`: handles `{models: [...]}`
- `test_list_models_openai_format`: placeholder (incomplete/inactive)
- `test_send_message_non_streaming`: constructs correct payload and parses response
- `test_send_message_streaming`: SSE parsing logic with mocked async iterator
- Uses `pytest-asyncio` and `unittest.mock.AsyncMock`

**Coverage Assessment:**
- Config: good (parsing edge cases)
- Storage: good (CRUD, trimming, settings)
- ClaudeProxyClient: moderate (main send_message covered for both modes; model listing partially; token count not tested)
- Handlers: **none** - no unit tests for command/chat/inline handlers
- Middlewares: **none** - no tests for rate limiting or logging
- Media utils: **none** - no tests for transcription or document extraction

**Overall:** Core utilities are well-tested. Business logic layer (handlers) lacks direct unit tests, which could be improved.

### 6.2 Integration Tests

Directory `tests/integration/` does not exist; no integration tests present.

**Opportunity:** Write tests that spin up FastAPI app and Telegram updates, possibly using aiogram's testing utilities or httpx client.

### 6.3 Test Execution

```bash
pytest tests/unit
```

The project includes `pytest` and `pytest-asyncio` in `requirements.txt`. No `pytest.ini` or `pyproject.toml` found, so uses defaults.

---

## 7. Code Quality Assessment

| Criterion | Evaluation |
|-----------|------------|
| **Structure** | Clean separation: config, handlers, services, utils, middlewares. |
| **Naming** | Clear, descriptive names matching Python conventions. |
| **Type Hints** | Used extensively (Python 3.10+ style where appropriate). |
| **Docstrings** | Minimal (top-level module doc missing), but code is self-documenting. |
| **Error Handling** | Try/except at chat handler level for API errors; rate limit middleware silently blocks; streaming errors reported to user. |
| **Logging** | Structured JSON logging, includes useful context. |
| **Async/Await** | Properly used throughout; no blocking calls on event loop (media uses executor). |
| **Constants** | Few hardcoded values: 4096 (Telegram limit), 20 (max history), 60 (rate limit window), 120 (default timeout). Acceptable. |
| **Security** | Webhook secret token, Bearer auth, HTTPS recommendation, secret token support. Rate limiting. |
| **Config Management** | Pydantic settings with env var validation and .env support. |
| **Dependencies** | Reasonable, pinned versions. Optional dependencies clearly commented in requirements.txt. |
| **README** | Excellent: features, installation, usage, architecture, security, limitations. |
| **Docker** | Multi-stage Dockerfile and docker-compose provided. |
| **Git History** | Several previous commits from issue-1 and issue-3; current branch has only initial and revert. |

**Potential Improvements:**
1. Add unit tests for handlers (commands, chat, inline)
2. Add integration tests
3. Add docstrings to public modules/classes/functions
4. Extract magic numbers to constants module
5. Consider moving `max_history` and rate limit window to config
6. Add health checks for free-claude-code connectivity
7. Add graceful handling when Whisper is not installed (better error)
8. Consider persistent storage (Redis) for multi-process deployments
9. Add `__all__` exports in `__init__.py` for cleaner imports
10. Add mypy for static type checking (optional but helpful)

---

## 8. Security Considerations

**Implemented:**
- API tokens kept out of version control (via .env)
- `API_SECRET_TOKEN` for webhook verification (prevents spoofed updates)
- HTTPS recommended for webhook
- Rate limiting prevents abuse
- Structured logging avoids sensitive data exposure (username/user_id logged, not full text? Actually text logged in LoggingMiddleware - see `text=event.text`). **Potential info leak:** message text is logged at INFO level. Consider redacting or lowering log level for text.

**Recommendations:**
- Do not log full message text in production; log only metadata (user_id, chat_id, message length). Adjust LoggingMiddleware.
- Ensure `.env` is in `.gitignore` (already present)
- Rotate tokens regularly
- Consider using `TELEGRAM_ALLOWED_CHAT_IDS` as a security measure for private bots

---

## 9. Deployment & Operations

### 9.1 Local Development

```bash
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
# Edit .env with your tokens
uvicorn bot.main:app --reload --port 8000
```

Prerequisite: free-claude-code running on configured URL (default http://localhost:8082).

### 9.2 Production (Docker)

```bash
docker-compose up -d
```

Includes both bot and free-claude-code. Exposes port 8000.

**Note:** For webhook mode, must set `TELEGRAM_WEBHOOK_URL` to a publicly accessible HTTPS URL with valid certificate; also set `API_SECRET_TOKEN`.

### 9.3 Monitoring

- Health endpoint: `GET /health` returns `{"status": "ok"}`
- Structured logs in JSON format for easy ingestion (ELK, Loki)
- No built-in metrics (Prometheus) – could add

### 9.4 Scalability

- Single process, in-memory storage
- For horizontal scaling, switch to Redis storage and shared rate limit storage (currently per-process)
- Consider using a database for persistent conversation history

---

## 10. Limitations & Future Work

Documented in README:

- **In-memory storage** loses history on restart
- **No admin panel** or metrics
- **Advanced Telegram features** (polls, message effects, custom styles, scheduled messages) not implemented
- **Bot-to-bot communication** not supported
- **Voice transcription** slow with Whisper base model; consider external service

Additional observations:
- **Error handling**: Some API errors may propagate to user; could implement retries with backoff
- **Voice transcription**: Currently synchronous blocking in executor; could be queued for long audio
- **Rate limit per process**: Not shared if multiple workers; if scaling, need external store (Redis)
- **No conversation truncation** based on token count; may exceed model context window if history gets large. Could implement sliding window based on tokens.

---

## 11. Conclusion

The **Telegram Claude Agent** is a **well-architected, production-ready** Telegram bot that successfully integrates Claude AI capabilities through a proxy service. The codebase demonstrates:

- Clear separation of concerns across layers
- Proper async patterns and resource management
- Comprehensive configurations
- Good test coverage for core utilities
- User-friendly features (streaming, guest mode, multimodal)
- Thoughtful security practices (auth tokens, rate limiting)
- Professional deployment options (Docker, uvicorn, health checks)

The implementation is concise yet complete, with approximately **~500 lines of application code** (excluding tests and config) providing substantial functionality.

**Recommendation:** The project is mature enough for initial deployment. Focus on:
1. Adding handler unit tests to increase confidence
2. Externalizing storage for persistence and scaling
3. Considering improvements to logging (remove message text)
4. Optional: adding integration tests

Overall, this is a high-quality codebase that serves as a solid foundation for a Claude-powered Telegram assistant.

---

## Appendix A: File Inventory

| File | Lines (approx.) | Purpose |
|------|-----------------|---------|
| `bot/main.py` | 78 | FastAPI app, lifecycle, webhook |
| `bot/config.py` | 37 | Settings definition |
| `bot/services/claude_proxy.py` | 95 | API client |
| `bot/handlers/chat.py` | 153 | Main message handling + streaming |
| `bot/handlers/commands.py` | 78 | Command handlers |
| `bot/handlers/inline.py` | 18 | Inline query |
| `bot/utils/storage.py` | 30 | Memory storage |
| `bot/utils/media.py` | 56 | Transcription & document extraction |
| `bot/middlewares/logging.py` | 35 | Logging middleware |
| `bot/middlewares/rate_limit.py` | 37 | Rate limiting |

---

## Appendix B: Test Summary

| Test File | Tests | Status |
|-----------|-------|--------|
| `tests/unit/test_config.py` | 4 | ✓ pass |
| `tests/unit/test_storage.py` | 4 | ✓ pass |
| `tests/unit/test_claude_proxy.py` | 4 | ✓ pass (one incomplete) |

**Total:** 12 unit tests

---

**Report prepared by:** Claude Code (Anthropic)
**Analysis based on:** Branch `issue-5-c7ee3a8d80c0`, commit `42edef6` (reverted) and working tree state.
