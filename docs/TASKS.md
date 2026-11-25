# MailMaster — Phased Implementation Plan

**Document Version:** 1.0.0  
**Last Updated:** 2026-11-25  
**Rule:** Never mark a task `[x]` unless its code/configuration is implemented, verified, and tested in the repository.

---

## Phase 1 — Foundation and Infrastructure

- [ ] **MM-101: Repository Scaffolding & Configuration Baseline**
  - **Dependencies:** None
  - **Description:** Create canonical `backend/`, `frontend/`, `infrastructure/`, `scripts/`, `tests/`, `.gitignore`, `.env.example`, and `LICENSE` aligned with `docs/ARCHITECTURE.md`.
  - **Expected Output:** Complete directory skeleton, `backend/pyproject.toml` (FastAPI, SQLAlchemy 2.0, Pydantic v2, Alembic, Celery, Redis, httpx, nh3, pgvector, OpenTelemetry, Ruff, Mypy, Pytest, Bandit, pip-audit), `frontend/package.json`.
  - **Acceptance Criteria:** Directory tree matches `docs/ARCHITECTURE.md`; `ruff` and `mypy` resolve configuration cleanly; `.env.example` documents all required environment variables without real secrets.

- [ ] **MM-102: Environment Configuration & Structured Logging**
  - **Dependencies:** MM-101
  - **Description:** Implement typed `Settings` via `pydantic-settings` in `app/core/config.py`, typed exception hierarchy in `app/core/exceptions.py`, and JSON structured logging with correlation IDs in `app/core/logging.py`.
  - **Expected Output:** `app/core/config.py`, `app/core/exceptions.py`, `app/core/logging.py`.
  - **Acceptance Criteria:** App fails fast on invalid/missing required secrets in non-test environments; logs emit valid JSON with `request_id`, `level`, and `timestamp`; unit tests verify validation rules.

- [ ] **MM-103: FastAPI Application Initialization & Health Endpoints**
  - **Dependencies:** MM-102
  - **Description:** Initialize FastAPI app factory with lifespan management, global exception handlers, CORS middleware, API v1 router, and liveness/readiness probes.
  - **Expected Output:** `app/main.py`, `app/api/v1/router.py`, `app/api/v1/health.py`, `app/api/dependencies.py`.
  - **Acceptance Criteria:** `GET /api/v1/health/live` returns `200`; `GET /api/v1/health/ready` verifies PostgreSQL + Redis connectivity and returns `503` if degraded.

- [ ] **MM-104: PostgreSQL 17+, pgvector & Async SQLAlchemy Engine**
  - **Dependencies:** MM-102
  - **Description:** Configure async SQLAlchemy 2.0 engine/sessionmaker (`asyncpg`), `pgvector` extension initialization script (`infrastructure/docker/postgres/init.sql`), and Alembic async migration environment.
  - **Expected Output:** `app/core/database.py`, `backend/alembic.ini`, `backend/alembic/env.py`, `infrastructure/docker/postgres/init.sql`.
  - **Acceptance Criteria:** `CREATE EXTENSION IF NOT EXISTS vector;` runs cleanly; `alembic upgrade head` and `alembic downgrade base` execute without errors against PostgreSQL 17 + pgvector.

- [ ] **MM-105: Redis Client & Celery Baseline**
  - **Dependencies:** MM-102
  - **Description:** Implement async Redis pool in `app/core/redis.py` and baseline Celery app (`sync`, `ai`, `rag` queues, `acks_late=True`, timeouts, retry defaults) in `app/workers/celery_app.py`.
  - **Expected Output:** `app/core/redis.py`, `app/workers/celery_app.py`.
  - **Acceptance Criteria:** Celery worker connects to Redis broker, responds to `inspect ping`, and shuts down cleanly on `SIGTERM`.

- [ ] **MM-106: Vue 3 + TypeScript + Tailwind + shadcn-vue Initialization**
  - **Dependencies:** MM-101
  - **Description:** Scaffold Vue 3 SPA with Vite, TypeScript strict mode, Pinia, Vue Router, Tailwind CSS dark-mode tokens from `docs/DESIGN.md`, `shadcn-vue`, Axios client, TanStack Query, ESLint, and Vitest.
  - **Expected Output:** `frontend/` app skeleton, `tailwind.config.ts`, `tsconfig.json`, `vite.config.ts`, `src/api/client.ts`.
  - **Acceptance Criteria:** `npm run build`, `npm run lint`, `vue-tsc --noEmit`, and `npx vitest run` all pass with zero errors.

- [ ] **MM-107: Docker Compose Stack & Nginx Reverse Proxy**
  - **Dependencies:** MM-103, MM-104, MM-105, MM-106
  - **Description:** Author `backend/Dockerfile`, `frontend/Dockerfile`, `infrastructure/nginx/nginx.conf`, and `docker-compose.yml` orchestrating `postgres`, `redis`, `backend`, `worker`, `frontend`, and `nginx` with non-root users and health checks.
  - **Expected Output:** `docker-compose.yml`, Dockerfiles, `infrastructure/nginx/nginx.conf`.
  - **Acceptance Criteria:** `docker compose config` validates cleanly; `docker compose up --build` brings all services to healthy state and serves `/api/v1/health/ready` via Nginx.

- [ ] **MM-108: CI Pipeline Verification on Code Scaffolding**
  - **Dependencies:** MM-103, MM-106, MM-107
  - **Description:** Verify all `.github/workflows/ci.yml` jobs (Ruff, Mypy strict, ESLint/Vitest, Pytest matrix 3.12 & 3.13, Bandit/pip-audit, Docker build) execute end-to-end against real application code.
  - **Expected Output:** Passing GitHub Actions CI run on PR.
  - **Acceptance Criteria:** All 6 CI jobs run without skipped checks and pass green.

---

## Phase 2 — Authentication and Gmail Integration

- [ ] **MM-201: Core Database Schema Migration (Users, Auth, Accounts, Sync)**
  - **Dependencies:** MM-104
  - **Description:** Define SQLAlchemy 2.0 ORM models and Alembic migration for `users`, `refresh_tokens`, `email_accounts`, `sync_states`, and `audit_logs`.
  - **Expected Output:** ORM models under `modules/auth/`, `modules/gmail/`, `modules/audit/` and Alembic migration `0001_initial_auth_gmail.py`.
  - **Acceptance Criteria:** Migration upgrades/downgrades cleanly; all unique constraints, FKs, and indexes match `docs/ARCHITECTURE.md`.

- [ ] **MM-202: Password Hashing, JWT & Refresh Token Rotation**
  - **Dependencies:** MM-201
  - **Description:** Implement Argon2id hashing, JWT signing/verification (`15m` access TTL), and `refresh_tokens` persistence with family-based reuse detection in `app/core/security.py` and `modules/auth/`.
  - **Expected Output:** `app/core/security.py`, `modules/auth/{repository,service,schemas}.py`, `app/api/v1/auth.py`.
  - **Acceptance Criteria:** Register (`201`/`409`), login (`200`/`401`), refresh rotation (`200`), reuse detection revoking token family (`401`), logout, and `/auth/me` pass unit + API tests with >= 90% branch coverage.

- [ ] **MM-203: Credential Encryption Utility (AES-256 / Fernet)**
  - **Dependencies:** MM-102
  - **Description:** Build envelope/Fernet authenticated encryption helper in `app/core/encryption.py` supporting `encryption_key_version` for OAuth tokens at rest.
  - **Expected Output:** `app/core/encryption.py`.
  - **Acceptance Criteria:** Round-trip encrypt/decrypt passes; tampered ciphertext or wrong key raises typed `EncryptionError`; plaintext tokens never appear in DB or logs.

- [ ] **MM-204: Google OAuth 2.0 Flow & Account Management**
  - **Dependencies:** MM-202, MM-203
  - **Description:** Implement CSRF `state` + PKCE storage in Redis, least-privilege Gmail scopes (`gmail.readonly`, `gmail.compose`, `gmail.send`, `openid email`), token exchange, encrypted `email_accounts` persistence, account listing, and account disconnect + audit logging.
  - **Expected Output:** `modules/gmail/oauth.py`, `modules/gmail/repository.py`, `app/api/v1/gmail.py`.
  - **Acceptance Criteria:** Invalid/expired OAuth `state` returns `400`; successful callback encrypts tokens, creates `sync_states`, writes `audit_logs`, and rejects cross-user account deletion with `403/404`.

- [ ] **MM-205: Async Gmail API Client**
  - **Dependencies:** MM-203, MM-204
  - **Description:** Build `GmailAPIClient` over `httpx.AsyncClient` with automatic OAuth token refresh, bounded concurrency (`asyncio.Semaphore`), exponential backoff on `429/5xx`, and methods for `messages.list`, `messages.get`, `history.list`, `labels.list`, and `messages.send`.
  - **Expected Output:** `modules/gmail/client.py`.
  - **Acceptance Criteria:** Mocked `respx` tests verify automatic token refresh on `401`, retry on `429`, `404` HistoryId-expired error typing, and zero blocking calls.

---

## Phase 3 — Email Synchronization and Intelligence

- [ ] **MM-301: Email, Thread, Label & Analysis Schema Migration**
  - **Dependencies:** MM-201
  - **Description:** Implement SQLAlchemy 2.0 models and Alembic migration for `email_threads`, `emails`, `email_labels`, and `email_analyses`.
  - **Expected Output:** `modules/emails/models.py`, `modules/ai/models.py`, Alembic migration `0002_emails_and_analyses.py`.
  - **Acceptance Criteria:** Unique constraints `(account_id, gmail_message_id)`, `(account_id, gmail_thread_id)`, and `(email_id)` on `email_analyses` exist and enforce integrity in PostgreSQL.

- [ ] **MM-302: Email MIME Parser & Normalization Pipeline**
  - **Dependencies:** MM-301
  - **Description:** Implement deterministic MIME traversal, HTML sanitization via `nh3`, plain-text extraction, RFC header capture (`Message-ID`, `In-Reply-To`, `References`), and quoted-reply stripping (`body_clean_no_quotes`) in `modules/gmail/normalizer.py`.
  - **Expected Output:** `modules/gmail/normalizer.py`.
  - **Acceptance Criteria:** Unit tests verify XSS `<script>`/`<iframe>`/event-handler stripping, multipart nested HTML/plain resolution, and clean separation of quoted reply chains.

- [ ] **MM-303: Initial Sync, Incremental History Sync & Celery Workers**
  - **Dependencies:** MM-205, MM-301, MM-302, MM-105
  - **Description:** Implement `GmailSyncService` and Celery task `sync_gmail_account` with Redis distributed lock (`sync:lock:{account_id}`), bounded initial sync (`500` messages / `30d`), incremental `history.list` cursor updates, `404` expired-history fallback reconciliation, and idempotent `ON CONFLICT` upserts.
  - **Expected Output:** `modules/gmail/sync_service.py`, `app/workers/tasks/sync_tasks.py`.
  - **Acceptance Criteria:** Re-running sync on identical Gmail payloads inserts 0 duplicate messages/threads; concurrent sync triggers respect Redis lock; `sync_states` updates atomically.

- [ ] **MM-304: Email & Thread Query APIs**
  - **Dependencies:** MM-301
  - **Description:** Implement `EmailService` and `EmailRepository` with keyset cursor pagination, filters (account, priority, category, unread), thread detail timeline, and read-state toggle.
  - **Expected Output:** `modules/emails/{repository,service,schemas}.py`, `app/api/v1/threads.py`, `app/api/v1/emails.py`.
  - **Acceptance Criteria:** API tests verify pagination stability, filter combinations, and strict `user_id` ownership isolation.

- [ ] **MM-305: Email Intelligence Service & Analysis Worker**
  - **Dependencies:** MM-303, MM-401, MM-402
  - **Description:** Implement `IntelligenceService` and Celery `analyze_email_task` generating structured `EmailAnalysis` (category, priority + reason, sentiment, intent, summary, action items) and on-demand thread summarization via `LLMGateway`.
  - **Expected Output:** `modules/ai/intelligence_service.py`, `modules/ai/prompts/`, `app/workers/tasks/ai_tasks.py`, `app/api/v1/ai.py`.
  - **Acceptance Criteria:** Mocked LLM tests verify Pydantic schema validation, retry on malformed JSON, prompt-version logging, untrusted-data XML wrapping, and thread-level summary persistence.

---

## Phase 4 — RAG and LLM Infrastructure

- [ ] **MM-401: Multi-Provider LLM Gateway & Usage Logging**
  - **Dependencies:** MM-201
  - **Description:** Implement `BaseLLMProvider`, `LLMGateway` (retries, circuit/fallback routing, pricing calculator, `llm_usage_logs` persistence), and `llm_usage_logs` Alembic migration.
  - **Expected Output:** `modules/ai/gateway/{base,router,schemas}.py`, `modules/ai/repository.py`, migration `0003_llm_usage_and_rag.py`.
  - **Acceptance Criteria:** Every call emits `llm_usage_logs`; transient errors retry with backoff; primary failure falls back to secondary provider when configured and records `fallback_used=True`.

- [ ] **MM-402: OpenAI & Anthropic Provider Adapters**
  - **Dependencies:** MM-401
  - **Description:** Implement `OpenAIProvider` (chat, structured JSON schema, SSE streaming, embeddings) and `AnthropicProvider` (chat, tool-schema structured outputs, SSE streaming) behind `BaseLLMProvider`.
  - **Expected Output:** `modules/ai/gateway/openai_provider.py`, `modules/ai/gateway/anthropic_provider.py`.
  - **Acceptance Criteria:** Mocked SDK/HTTP tests verify identical normalized `LLMResponse` / `StructuredLLMResponse` / `LLMStreamChunk` outputs across both providers.

- [ ] **MM-403: Deterministic Email Chunker & Embedding Worker**
  - **Dependencies:** MM-302, MM-402
  - **Description:** Implement `tiktoken` email chunker (`400` tokens, `60` overlap, metadata header prefix, `chunker_version`) in `modules/rag/chunker.py`, batch embedding service (`1536` dims), `email_chunks` (with `GIN` `search_tsv`) & `email_embeddings` (with `HNSW` index) persistence, and Celery `index_email_chunks_task`.
  - **Expected Output:** `modules/rag/{chunker,embeddings,models,repository}.py`, `app/workers/tasks/rag_tasks.py`.
  - **Acceptance Criteria:** Chunker produces deterministic spans; re-indexing same email/version is idempotent; `pgvector` HNSW and FTS `GIN` indexes exist in DB.

- [ ] **MM-404: Hybrid Retriever (pgvector + FTS + RRF) & Reranker**
  - **Dependencies:** MM-403
  - **Description:** Implement `HybridRetriever` executing parallel/CTE vector cosine search and PostgreSQL `ts_rank_cd` full-text search strictly filtered by `user_id` + `account_id`, fusing with RRF (`k=60`), and applying `Reranker` to select top-N chunks.
  - **Expected Output:** `modules/rag/{retriever,reranker,service,schemas}.py`, `POST /api/v1/emails/search`.
  - **Acceptance Criteria:** Integration tests against real PostgreSQL + pgvector verify hybrid ranking outperforms single-mode queries on synthetic test cases and guarantees 0 results leaked across `user_id` or unowned `account_id`.

- [ ] **MM-405: Context Builder, Citation Generator & RAG Evaluation Suite**
  - **Dependencies:** MM-404
  - **Description:** Build `ContextBuilder` wrapping chunks in `<retrieved_email_context>` XML fences with `[1]..[N]` citation maps, and build `modules/rag/evaluator.py` + `scripts/evaluate_rag.py` + `tests/rag_eval/` measuring `Recall@5`, `Recall@10`, `MRR`, and citation grounding.
  - **Expected Output:** `modules/rag/{context_builder,evaluator}.py`, `scripts/evaluate_rag.py`, `tests/rag_eval/`.
  - **Acceptance Criteria:** Evaluation script runs reproducibly against seeded golden fixtures and reports `Recall@5`, `Recall@10`, and `MRR` metrics meeting baseline thresholds (`Recall@10 >= 0.85`).

---

## Phase 5 — LangGraph and MCP

- [ ] **MM-501: Chat Persistence & LangGraph Assistant Workflow**
  - **Dependencies:** MM-402, MM-404, MM-405
  - **Description:** Add `chat_sessions` & `chat_messages` migration (`0004_chat_and_drafts.py`), define `AssistantState`, and implement LangGraph `StateGraph` (`classify_intent` -> conditional routing -> `load_thread_context` / `rag_retrieve` -> `rerank_and_build_context` -> `generate_response` -> `verify_citations`).
  - **Expected Output:** `modules/ai/workflows/{state,nodes,assistant_graph}.py`, `app/api/v1/chat.py`.
  - **Acceptance Criteria:** Unit/API tests verify all conditional branches, SSE token streaming, conversation windowing, and stripping/rejection of fabricated `[n]` citation markers.

- [ ] **MM-502: Smart Compose & Draft Approval Backend**
  - **Dependencies:** MM-501, MM-205
  - **Description:** Implement `email_drafts` state machine (`draft` -> `pending_approval` -> `approved` -> `sent`), AI reply generation grounded in RAG/thread context, optimistic `draft_version` locking, auto-reset to `draft` on edits, and Gmail send via `GmailAPIClient` with `audit_logs` emission.
  - **Expected Output:** `modules/drafts/{models,schemas,repository,service}.py`, `app/api/v1/drafts.py`.
  - **Acceptance Criteria:** Attempting `send` on `draft` or `pending_approval` returns `409 Conflict`; editing an `approved` draft invalidates approval; successful send records `gmail_message_id` and `audit_logs` entry.

- [ ] **MM-503: Custom MCP Server (Streamable HTTP) & Auth Middleware**
  - **Dependencies:** MM-202, MM-501, MM-502
  - **Description:** Build official Python `mcp` SDK server mounted at `/mcp` using Streamable HTTP transport, bearer token authentication (`app/mcp/auth.py`), and `MCPAuthContext` propagation.
  - **Expected Output:** `app/mcp/server.py`, `app/mcp/auth.py`.
  - **Acceptance Criteria:** Unauthenticated `/mcp` requests return `401`; valid bearer tokens resolve `user_id` cleanly.

- [ ] **MM-504: MCP Tools, Resources, Prompts & Security Tests**
  - **Dependencies:** MM-503
  - **Description:** Implement tools (`search_emails`, `get_thread`, `ask_email_assistant`, `list_action_items`, `create_email_draft`), resources (`mailmaster://accounts`, `mailmaster://threads/{thread_id}`), and prompts (`triage_inbox`, `draft_executive_reply`) reusing domain services and writing `audit_logs`.
  - **Expected Output:** `app/mcp/{tools,resources,prompts}.py`, `tests/mcp/`.
  - **Acceptance Criteria:** MCP tests verify Pydantic argument validation, strict `user_id` ownership isolation, `create_email_draft` never sends email, and every invocation persists an `audit_logs` record.

---

## Phase 6 — Frontend and Smart Compose

- [ ] **MM-601: App Shell, Navigation & Auth Screens**
  - **Dependencies:** MM-106, MM-202
  - **Description:** Implement Pinia `authStore`, Axios JWT refresh interceptor, Vue Router auth guards, `AppSidebar`, `TopNavBar`, `LoginPage`, and `RegisterPage` styled to `docs/DESIGN.md`.
  - **Expected Output:** `frontend/src/stores/auth.ts`, `frontend/src/layouts/`, `frontend/src/pages/{Login,Register}Page.vue`, `frontend/src/components/layout/`.
  - **Acceptance Criteria:** Login/register flows handle validation errors cleanly, store access token securely in memory/store, refresh transparently on `401`, and pass Vitest component tests.

- [ ] **MM-602: Inbox Dashboard & Email Thread Detail View**
  - **Dependencies:** MM-601, MM-304, MM-305
  - **Description:** Implement `InboxThreadList`, `EmailPreviewPane`, `EmailThreadDetail`, `PriorityBadge`, `CategoryLabel`, `LoadingSkeletons`, `EmptyState`, and `DOMPurify`-sanitized HTML rendering backed by TanStack Query.
  - **Expected Output:** `frontend/src/pages/{InboxPage,ThreadDetailPage}.vue`, `frontend/src/components/inbox/`.
  - **Acceptance Criteria:** Users can filter by priority/category/unread, paginate threads, view AI summaries and extracted action items, toggle read state optimistically, and traverse via keyboard (`j`/`k`).

- [ ] **MM-603: AI Chat Assistant & Interactive Citations UI**
  - **Dependencies:** MM-601, MM-501
  - **Description:** Implement `AIAssistantView`, session sidebar, SSE streaming client, `ChatMessageBubble`, and `CitationCard` linking citations directly to `/inbox/threads/:threadId`.
  - **Expected Output:** `frontend/src/pages/AssistantPage.vue`, `frontend/src/components/ai/`.
  - **Acceptance Criteria:** Streaming responses render incrementally, `[n]` citation pills highlight/expand corresponding source excerpts, and clicking **View Thread** opens the exact cited email.

- [ ] **MM-604: Smart Compose Editor & Human Approval Flow**
  - **Dependencies:** MM-602, MM-502
  - **Description:** Implement `SmartComposeEditor`, `DraftApprovalBar`, `/drafts` queue view, tone/RAG controls, edit-resets-approval visual indicator, and explicit **Approve & Send** confirmation modal.
  - **Expected Output:** `frontend/src/pages/DraftsPage.vue`, `frontend/src/components/compose/`.
  - **Acceptance Criteria:** Draft generation, editing, approval modal confirmation, and Gmail send work end-to-end; Vitest verifies send cannot fire without confirmation modal acceptance.

- [ ] **MM-605: Settings, Gmail Integration & AI Usage Screens**
  - **Dependencies:** MM-601, MM-204, MM-401
  - **Description:** Implement `GmailAccountsPanel`, `AIProviderSettings`, usage/cost breakdown table, and MCP connection token generator/copy UI.
  - **Expected Output:** `frontend/src/pages/SettingsPage.vue`, `frontend/src/components/settings/`.
  - **Acceptance Criteria:** Users can initiate Gmail OAuth, trigger manual sync, monitor sync status, disconnect an account with confirmation, and switch primary LLM provider.

---

## Phase 7 — Production Hardening, Chaos Testing and Packaging

- [ ] **MM-701: Rate Limiting & Security Audit Hardening**
  - **Dependencies:** MM-504, MM-605
  - **Description:** Implement Redis sliding-window rate limiter (`app/core/rate_limit.py`), review OWASP Top 10 / prompt-injection / XSS vectors, tighten CSP and Nginx security headers, and resolve all `Bandit` and `pip-audit` findings.
  - **Expected Output:** `app/core/rate_limit.py`, updated `infrastructure/nginx/nginx.conf`, security test suite.
  - **Acceptance Criteria:** Auth/AI endpoints return `429` with `Retry-After` when exceeded; `bandit` and `pip-audit` report zero high/critical issues.

- [ ] **MM-702: Retry, Recovery & Idempotency Chaos Tests**
  - **Dependencies:** MM-303, MM-401
  - **Description:** Write fault-injection tests simulating Gmail `429`/`404` expired history, mid-sync worker crashes, LLM primary provider outages, and concurrent sync lock contention.
  - **Expected Output:** `tests/integration/test_resilience.py`.
  - **Acceptance Criteria:** Zero duplicate emails/chunks created across replayed failures; LLM fallback succeeds automatically; lock releases cleanly after failure.

- [ ] **MM-703: Database Performance & Query Plan Verification**
  - **Dependencies:** MM-404
  - **Description:** Benchmark inbox pagination and hybrid RAG CTE queries against 50,000 synthetic email chunks using `EXPLAIN (ANALYZE, BUFFERS)` and tune indexes/HNSW `ef_search` if needed.
  - **Expected Output:** Performance benchmark script in `scripts/` and documented query plan results in `docs/MEMORY.md`.
  - **Acceptance Criteria:** Inbox list p95 <= 150 ms and hybrid retrieval p95 <= 350 ms on benchmark dataset.

- [ ] **MM-704: OpenTelemetry Observability & Production Docker Hardening**
  - **Dependencies:** MM-107, MM-701
  - **Description:** Wire OpenTelemetry spans across FastAPI, SQLAlchemy, `httpx`, and Celery (`app/core/telemetry.py`), add multi-stage minimal Docker builds, non-root runtime enforcement, and `infrastructure/compose/docker-compose.prod.yml`.
  - **Expected Output:** `app/core/telemetry.py`, hardened Dockerfiles, `infrastructure/compose/docker-compose.prod.yml`.
  - **Acceptance Criteria:** Traces emit cleanly to OTLP/console exporter; Docker images pass build & non-root user verification.

- [ ] **MM-705: End-to-End Playwright Suite & Release v1.0.0**
  - **Dependencies:** MM-701, MM-702, MM-703, MM-704
  - **Description:** Implement Playwright E2E suite (`tests/e2e/`) covering registration, login, mocked Gmail OAuth/sync, inbox triage, RAG chat citations, and Smart Compose approval; finalize release documentation and tag `v1.0.0`.
  - **Expected Output:** `tests/e2e/`, updated `README.md`, `docs/TASKS.md`, `docs/MEMORY.md`, GitHub Release `v1.0.0`.
  - **Acceptance Criteria:** All unit, integration, API, RAG eval, MCP, Vitest, and Playwright suites pass in CI; branch coverage >= 90% on critical business logic.
