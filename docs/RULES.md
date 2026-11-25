# MailMaster — AI and Developer Engineering Rulebook

**Document Version:** 1.0.0  
**Last Updated:** 2026-11-25  
**Scope:** Mandatory for all human contributors and AI coding agents operating on MailMaster.

---

## 1. Architectural Rules

1. **Modular Monolith Boundaries:** Domain modules live under `backend/app/modules/<domain>/`. Cross-module calls must go through another module's public `service.py` or `schemas.py`—never import another module's private `repository.py` or raw ORM `models.py` for mutation.
2. **Thin API Controllers:** Files in `backend/app/api/v1/` must only handle request parsing, dependency injection, calling a service method, and returning a Pydantic response schema. Zero business logic and zero SQLAlchemy queries are permitted inside route handlers.
3. **Repository Isolation:** All database reads and writes belong in `repository.py`. Every query targeting user- or account-scoped entities (`email_accounts`, `email_threads`, `emails`, `email_analyses`, `email_chunks`, `email_embeddings`, `chat_sessions`, `chat_messages`, `email_drafts`) **must** require and filter by `user_id` (and `account_id` where applicable).
4. **Provider Adapters Only:** External services (Gmail API, OpenAI API, Anthropic API) must be accessed exclusively through dedicated adapter classes (`modules/gmail/client.py`, `modules/ai/gateway/*`).
5. **No Direct LLM SDK Coupling:** Application services, LangGraph nodes, RAG components, and workers must **never** import `openai` or `anthropic` SDKs directly. All model calls go through `LLMGateway`.
6. **Idempotent Sync & Retry-Safe Background Tasks:** Email sync and indexing tasks must use deterministic upserts (`ON CONFLICT (account_id, gmail_message_id)`) and Redis distributed locks (`sync:lock:{account_id}`) so duplicate triggers or worker restarts never duplicate rows or corrupt state.
7. **Explicit Transaction Boundaries:** Write operations spanning multiple repositories must share a single explicit async transaction (`async with session.begin():` or unit-of-work context) managed by the service layer. Repositories must `flush()`, not `commit()`, unless explicitly acting as standalone single-operation boundaries.
8. **Auditable External Actions:** Linking/unlinking Gmail accounts, approving/sending drafts, and every MCP tool execution must append an entry to `audit_logs`.
9. **MCP Service Reuse:** MCP tools in `backend/app/mcp/tools.py` must reuse domain services (`emails`, `rag`, `ai`, `drafts`) rather than reimplementing database queries or business logic.

---

## 2. Python Standards

1. **Python Version:** Target **Python 3.12+** (tested in CI on both 3.12 and 3.13). Use modern union syntax (`str | None`), `list[...]`, `dict[...]`, and `StrEnum`.
2. **Complete Type Annotations:** All public functions, methods, class attributes, and FastAPI dependencies must carry explicit type annotations. Avoid `Any`; where dynamic JSON payloads exist, model them with `TypedDict`, Pydantic `BaseModel`, or `dict[str, object]`.
3. **Strict Pydantic v2 Schemas:**
   - Request schemas must set `model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)`.
   - ORM response DTOs must set `model_config = ConfigDict(from_attributes=True)`.
   - Never return raw SQLAlchemy ORM instances directly from FastAPI endpoints.
4. **SQLAlchemy 2.0 Patterns:** Use `Mapped[...]` and `mapped_column(...)` exclusively with `select(...)` / `insert(...).on_conflict_do_update(...)`. Legacy `Session.query()` is forbidden.
5. **Async-Safe Database Access:** Use `AsyncEngine`, `async_sessionmaker[AsyncSession]`, and `asyncpg`. Never enable `expire_on_commit=True` on async sessions.
6. **Dependency Injection:** Wire services, gateways, and repositories via FastAPI `Depends(...)` factories in `app/api/dependencies.py` (or clean container factories for Celery tasks) so unit and API tests can override dependencies deterministically.
7. **Linting, Formatting & Static Typing:**
   - `ruff check .` and `ruff format --check .` must pass with zero warnings.
   - `mypy --strict app` must pass; any third-party untyped stub exclusions must be narrowly scoped in `pyproject.toml` with an inline justification comment.
8. **Explicit Exception Hierarchy:** Raise typed domain exceptions from `app/core/exceptions.py` (e.g., `ResourceNotFoundError`, `OwnershipViolationError`, `DraftStateConflictError`, `LLMProviderError`, `GmailSyncError`). Map them centrally in FastAPI exception handlers.
9. **No Silent Exceptions:** Bare `except:` or `except Exception: pass` is strictly prohibited. Every caught exception must either re-raise a contextual domain error or log structured diagnostic context with stack trace.
10. **Zero Hardcoded Credentials:** Secrets (`JWT_SECRET_KEY`, `ENCRYPTION_MASTER_KEY`, OAuth client secrets, LLM keys) must come solely from `Settings` (`pydantic-settings`). `bandit` checks run in CI.

---

## 3. Async and Concurrency Rules

1. **Non-Blocking Event Loop:** Synchronous blocking I/O (`requests`, `time.sleep`, blocking file I/O, synchronous DB drivers) is forbidden inside FastAPI async routes and async services. Use `httpx.AsyncClient` and `asyncio.sleep`.
2. **CPU-Bound Parsing Offloading:** Heavy HTML sanitization (`nh3`) or large MIME/tokenization batches inside async contexts must run in Celery workers or via `asyncio.to_thread` when invoked inline.
3. **HTTP & External Timeouts:** Every outbound `httpx` or LLM SDK call must configure explicit connect and read timeouts (e.g., `connect=5s`, `read=45s` for LLM streaming/completions, `read=20s` for Gmail API).
4. **Concurrency Limits:** Batch Gmail message fetches and embedding calls must enforce bounded concurrency (`asyncio.Semaphore(10)` for Gmail, batch size `<= 64` texts for embeddings) to avoid exhausting connection pools or triggering provider rate-limit bans.
5. **Database Session Lifecycle:** Never share a single `AsyncSession` concurrently across `asyncio.gather()` coroutines. Each concurrent unit must either run sequentially within the session or open its own scoped session.
6. **Celery Task Boundaries:**
   - Pass primitive IDs (`str(account_id)`, `str(email_id)`) across Celery queues—never serialized ORM objects or raw OAuth tokens.
   - Configure `task_acks_late=True`, `task_reject_on_worker_lost=True`, explicit `soft_time_limit` / `time_limit`, and bounded retries (`max_retries=3`, exponential backoff + jitter).
   - Async Celery handlers must use a controlled single event-loop runner (`asyncio.run` per task with cleanly disposed engine/pool or worker-lifecycle loop) so `asyncpg` pools never leak across closed loops.
7. **Graceful Shutdown:** FastAPI `lifespan` must cleanly dispose the SQLAlchemy `AsyncEngine`, close the Redis pool, and flush OpenTelemetry spans on `SIGTERM`.

---

## 4. AI Engineering Rules

1. **Structured Outputs Everywhere:** Classification, prioritization, sentiment, intent, action items, and LangGraph intent routing must return JSON validated against strict Pydantic v2 schemas.
2. **Validate All Model Outputs:** Never pass raw LLM text directly into database columns requiring enums or JSON structures without `model_validate` verification.
3. **Prompt & Model Versioning:** All system/task prompts must reside in `app/modules/ai/prompts/` with immutable version strings (`email_classify.v1`). Every `email_analyses` and `llm_usage_logs` row must record `model_provider`, `model_name`, and `prompt_version`.
4. **Mandatory Token & Cost Tracking:** Every call through `LLMGateway` (including failed/fallback attempts) must emit an `llm_usage_logs` record.
5. **Untrusted Email Data Principle:** Email subjects, bodies, sender display names, and retrieved RAG chunks are **untrusted external input**.
   - Always delimit retrieved/email text inside `<untrusted_email_content>` / `<retrieved_email_context>` XML tags.
   - Never concatenate raw email text into system instructions.
   - Never allow LLM output to directly specify SQL clauses, file paths, or arbitrary tool execution without deterministic server-side validation.
6. **Authorization Outside the LLM:** Access control (`user_id`, `account_id`) is enforced deterministically in Python/SQL *before* data reaches the LLM and *before* any tool executes.
7. **Human Approval Gate for Outbound Email:** No AI workflow or MCP tool may call Gmail `messages.send`. Only `DraftService.send_approved_draft()` may send, and it must verify `status == "approved"` and `draft_version` match inside a row-locked transaction (`SELECT ... FOR UPDATE`).
8. **Strict Citation Grounding:** RAG assistant responses must reference only citation indices present in the retrieved `context_citations` map. `verify_citations` must strip or reject any fabricated `[n]` citation.

---

## 5. RAG Rules

1. **Mandatory Source Provenance:** Every `email_chunks` row must store `email_id`, `thread_id`, `account_id`, `user_id`, `chunk_index`, and `chunker_version`.
2. **Account-Level Isolation in SQL:** Hybrid retrieval (`pgvector` cosine search + PostgreSQL `tsvector` FTS) must filter `WHERE user_id = :user_id AND account_id = ANY(:account_ids)` inside both candidate subqueries before RRF fusion. Filtering *after* top-K retrieval is strictly prohibited.
3. **Embedding Version Consistency:** Queries must only compare vectors sharing the exact `embedding_model` and `embedding_version` used at query time.
4. **Chunking Reproducibility:** Changing `chunk_size`, `chunk_overlap`, or metadata prefix format requires incrementing `chunker_version` (e.g., `v1.1`).
5. **Continuous Retrieval Evaluation:** Changes to chunking, RRF weights, or reranking must be benchmarked against `tests/rag_eval/` (`Recall@5`, `Recall@10`, `MRR`) before merging.
6. **No Unsupported Claims:** System prompts for RAG Q&A must instruct the model to explicitly state *"I could not find evidence for that in your connected emails"* when retrieved context is insufficient.

---

## 6. MCP Rules

1. **Mandatory Client Authentication:** `/mcp` must reject unauthenticated requests with `401 Unauthorized`. Development bypass flags are forbidden in production configs.
2. **Strict Schema Validation:** Every MCP tool input must validate via Pydantic v2 with explicit length limits and UUID/type checks.
3. **User Authorization Propagation:** MCP handlers resolve `user_id` exclusively from the validated bearer token—never from a tool argument.
4. **Narrow Tool Scope:** Keep tools single-purpose (`search_emails`, `get_thread`, `ask_email_assistant`, `list_action_items`, `create_email_draft`). Do not expose generic SQL or arbitrary HTTP proxy tools.
5. **No Unapproved Sending:** MCP tools may create drafts in `pending_approval` state (`create_email_draft`), never send emails directly.
6. **Mandatory Audit Logging:** Every MCP tool invocation (including rejected/failed calls) must persist an `audit_logs` record with `actor_type="mcp_client"`, `action`, `resource_id`, and `correlation_id`.

---

## 7. Testing Rules

1. **Test Pyramid & Required Suites:**
   - `tests/unit/`: Pure service, chunker, normalizer, RRF, state-machine, encryption, and LangGraph routing tests (external APIs mocked).
   - `tests/integration/`: PostgreSQL + `pgvector` + Redis tests verifying SQLAlchemy repositories, `ON CONFLICT` sync idempotency, FTS + vector RRF queries, and cross-account isolation.
   - `tests/api/`: FastAPI endpoint tests covering auth, refresh rotation reuse detection, OAuth state CSRF checks, draft approval state conflicts, and uniform error responses.
   - `tests/rag_eval/`: Golden-dataset retrieval quality (`Recall@5`, `MRR`) and citation-faithfulness regression suite.
   - `tests/mcp/`: MCP auth enforcement, argument validation, ownership isolation, and audit log assertions.
   - `frontend/tests/`: Vitest component and store tests.
   - `tests/e2e/`: Playwright end-to-end flows for login, inbox triage, RAG chat citations, and Smart Compose approval.
2. **Branch Coverage Target:** Maintain **>= 90% branch coverage** (`pytest --cov=app --cov-branch`) across critical backend business logic (`app/modules/`, `app/mcp/`, `app/core/security.py`, `app/core/encryption.py`).
   - Any excluded lines (e.g., `if TYPE_CHECKING:` or `uvicorn` entrypoint) must be explicitly documented in `pyproject.toml` rather than masked by dummy tests.
3. **Mocking Discipline:** Unit tests must mock OpenAI, Anthropic, and Google Gmail HTTP endpoints deterministically via `respx` / provider fakes—never hit live paid APIs in CI.

---

## 8. Git Standards

1. **Conventional Commits (Enforced by CI):** Every PR title and commit message must follow `<type>(<optional-scope>): <imperative summary>`:
   - Allowed types: `feat`, `fix`, `docs`, `refactor`, `perf`, `test`, `chore`, `security`.
   - Examples: `feat(rag): implement hybrid vector and FTS RRF retriever`, `fix(gmail): handle expired historyId with bounded reconciliation`.
2. **Branch Naming:** Use descriptive prefixes: `feat/<topic>`, `fix/<topic>`, `docs/<topic>`, `refactor/<topic>`, `chore/<topic>`, `security/<topic>`. Direct pushes to `main` are disallowed once branch protection is enabled.
3. **Pull Request Requirements:**
   - Complete `.github/PULL_REQUEST_TEMPLATE.md` accurately.
   - All `ci.yml` and `pr-audit.yml` checks must pass.
   - Update `docs/TASKS.md` and `docs/MEMORY.md` whenever tasks are completed or architectural decisions change (and add an ADR in `MEMORY.md` for structural changes).
4. **Breaking Changes:** Must include `BREAKING CHANGE:` in PR description/commit footer, API version bump if REST contract breaks, and migration rollback notes for Alembic schema changes.
