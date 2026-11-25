# MailMaster — Product Requirements Document (PRD)

**Document Version:** 1.0.0  
**Last Updated:** 2026-11-25  
**Author:** Anmol Vishvas (`anmol23vishvas@gmail.com`)  
**Target Repository:** https://github.com/anmolvishvas/MailMaster  
**Status:** Approved Baseline for Phase 1 Implementation

---

## 1. Product Overview

### 1.1 Product Vision

Email remains the primary system of record for engineering decisions, client agreements, vendor negotiations, project approvals, and operational escalations—yet modern inboxes operate as passive, chronological message dumps. **MailMaster** transforms email from a manual triage burden into an intelligent, verifiable, context-aware knowledge and communication workspace.

### 1.2 Product Positioning

MailMaster is a self-hostable, commercial-grade **AI-Powered Email Intelligence, Retrieval-Augmented Generation (RAG), and Workflow Automation Platform** built around Gmail, multi-provider LLMs (OpenAI and Anthropic), LangGraph orchestration, and the Model Context Protocol (MCP). Unlike superficial "AI email wrapper" extensions that summarize single messages in isolation, MailMaster indexes entire conversation histories per connected account into a hybrid vector + full-text retrieval store, enforces strict source-grounded citations, orchestrates intent-driven reasoning workflows, and exposes authenticated email capabilities to external AI clients via MCP—all under a strict **human-in-the-loop** execution guarantee.

### 1.3 One-Line Pitch

> An Intelligent AI-Powered Email Management, Knowledge Retrieval and Workflow Automation Platform.

### 1.4 Core Value Proposition

| Dimension | Traditional Email Clients | Generic AI Email Extensions | MailMaster |
|---|---|---|---|
| **Triage & Prioritization** | Manual folder/label rules and chronological sorting | Per-message generic tags | Structured LLM extraction of priority, intent, sentiment, category, and action items with thread-level synthesis |
| **Historical Knowledge Retrieval** | Exact-match keyword search; misses semantic intent | No cross-thread memory or relies on naive stuffing | Hybrid retrieval (pgvector HNSW cosine similarity + PostgreSQL `tsvector` BM25-style ranking via Reciprocal Rank Fusion + reranking) |
| **Trust & Grounding** | Manual reading required to verify claims | Frequent hallucinations without verifiable links | Source-grounded answers with clickable citations pointing to exact `email_id`, `thread_id`, sender, timestamp, and quoted excerpt |
| **Reply Drafting** | Blank-page composition | Generic tone rewrites without thread/historical context | Context-aware Smart Compose pulling relevant past threads via RAG and respecting user tone/length instructions |
| **Safety & External Actions** | Manual send | Risk of unreviewed autonomous actions or prompt injection | Mandatory human approval gate (`pending_approval` → `approved` → `sent`) before any Gmail send; untrusted email content isolation |
| **Ecosystem Interoperability** | Locked inside webmail UI | Proprietary plugins | Standards-compliant Streamable HTTP MCP server exposing authenticated tools, resources, and prompts |

### 1.5 Main User Journeys

1. **Onboarding & Account Connection:** A user registers with email/password, logs into MailMaster, initiates Google OAuth 2.0 consent with least-privilege Gmail scopes, and links one or more Gmail accounts. MailMaster encrypts OAuth tokens at rest and triggers an initial background synchronization.
2. **Automated Triage & Inbox Intelligence:** As emails ingest and threads are reconstructed, Celery workers normalize HTML/plain-text bodies, redact/isolate untrusted markup, and run structured LLM classification to populate priority (`urgent`, `high`, `normal`, `low`), category (`work`, `finance`, `engineering`, `scheduling`, `newsletter`, `personal`, `support`), sentiment, intent, summary, and actionable tasks.
3. **Contextual Knowledge Retrieval (AI Chat Assistant):** A user asks a natural-language question (e.g., *"What did the Acme team agree on regarding the Q3 SLA penalty?"*). LangGraph classifies intent, invokes the hybrid RAG pipeline scoped strictly to the user's connected account(s), reranks candidate chunks, and streams a grounded answer with inline `[1]`, `[2]` citations linking directly to the underlying email messages.
4. **Smart Compose & Human-Approved Sending:** On an active thread or standalone compose view, the user requests an AI-generated reply (optionally referencing historical context via RAG). MailMaster creates an `email_drafts` record in `draft` state. The user edits the body, explicitly clicks **Approve & Send**, transitioning the draft through `approved` → `sent` via the Gmail API with a full audit log entry.
5. **External AI Agent Integration via MCP:** A user connects an MCP-compatible client (e.g., Claude Desktop or an IDE agent) to MailMaster's `/mcp` Streamable HTTP endpoint using a scoped bearer token to search emails, inspect threads, ask RAG questions, or generate a draft—still blocked from sending without explicit approval in MailMaster.

---

## 2. Problem Statement

### 2.1 Information Overload in Conventional Email Clients

Knowledge workers routinely receive 80–200+ emails daily across fragmented threads, automated notifications, CC chains, and urgent escalations. Conventional clients treat a 25-message architectural debate and an automated receipt identically in a flat chronological list, forcing users to spend 20–30% of their workday context-switching, opening threads to assess urgency, and manually tracking buried action items.

### 2.2 Inadequacy of Keyword Search for Contextual Retrieval

Traditional email search relies on exact lexical matching (`from:alice "rate limit"`). Users frequently remember *concepts, decisions, or outcomes* rather than exact terminology (e.g., searching *"that database connection pool incident in October"* when the email actually discussed *"PgBouncer saturation and Hikari timeout errors"*). Pure lexical search returns zero or noisy results, while naive full-thread scanning fails to surface the exact paragraph containing the answer.

### 2.3 Inaccessible Historical Email Knowledge

Critical institutional and personal context—pricing quotes, contract amendments, technical design trade-offs, interview feedback, travel itineraries—is trapped inside years of email threads. Extracting this knowledge requires opening dozens of threads and mentally reconstructing timelines across forwarded replies and quoted text blocks.

### 2.4 Limitations of Existing AI Email Assistants

Most existing AI email tools suffer from three architectural flaws:
1. **Context Blindness:** They pass only the current email body into an LLM prompt, producing generic, factually disconnected replies.
2. **Unverified Hallucinations:** They answer questions about inbox history without deterministic retrieval grounding or verifiable message citations.
3. **Unsafe Execution & Vendor Lock-In:** They couple directly to a single LLM vendor, lack structured output validation, and fail to isolate untrusted email text from tool execution—creating severe prompt-injection risks.

### 2.5 How MailMaster Solves These Problems

MailMaster resolves each gap through deliberate systems engineering:
- **Deterministic Ingestion & Normalization:** Strips HTML boilerplate, deduplicates quoted reply chains, reconstructs Gmail threads, and persists clean message bodies with idempotent incremental history sync.
- **Hybrid RAG Pipeline:** Combines `pgvector` dense semantic search with PostgreSQL `tsvector` full-text search via Reciprocal Rank Fusion (RRF), optional cross-encoder/LLM reranking, and strict `account_id` isolation.
- **Structured AI Intelligence:** Validates every classification, summary, and action item against strict Pydantic v2 schemas via a provider-independent LLM Gateway (OpenAI & Anthropic).
- **LangGraph + Human-in-the-Loop + MCP:** Uses explicit state graphs for multi-step assistant reasoning, enforces human approval prior to any outbound email send, and exposes safe, audited capabilities over the official Python MCP SDK.

---

## 3. Goals and Non-Goals

### 3.1 MVP Goals (Release v1.0)

1. Deliver a secure, modular-monolith FastAPI backend and Vue 3 TypeScript frontend deployable via a single `docker compose up` command.
2. Provide user registration, JWT authentication with rotating refresh tokens, and encrypted Google OAuth 2.0 Gmail account linking.
3. Implement idempotent initial and incremental Gmail synchronization using Gmail History API (`historyId`) via Celery + Redis.
4. Provide automated LLM-driven classification, priority scoring, sentiment/intent detection, summarization, and action-item extraction.
5. Build a custom, independently evaluable hybrid RAG pipeline (`pgvector` + PostgreSQL FTS + RRF + reranking) with mandatory source citations.
6. Implement a multi-provider LLM Gateway supporting OpenAI and Anthropic with token/cost logging, retries, and configurable provider fallback.
7. Orchestrate conversational Q&A and drafting workflows through LangGraph with explicit conditional routing.
8. Expose authenticated email search, thread reading, RAG Q&A, and draft creation via a custom MCP server over Streamable HTTP transport.
9. Implement Smart Compose with inline draft editing and mandatory human approval before Gmail send.
10. Enforce ≥90% branch coverage on critical backend business logic, strict Ruff/Mypy compliance, and automated CI checks.

### 3.2 Explicit MVP Non-Goals

The following capabilities are **out of scope for MVP v1.0** and must not be implemented in the initial release, though module boundaries must accommodate them cleanly in future phases:

- Microsoft Outlook / Exchange / IMAP integration.
- Enterprise multi-tenant organization management (RBAC across teams, SSO/SAML, shared team inboxes).
- Autonomous email sending without explicit human approval.
- Drag-and-drop visual workflow builder UI.
- Slack, Microsoft Teams, or Discord integrations.
- Google Calendar / scheduling automation.
- Kubernetes manifests, Helm charts, or multi-region orchestration.
- Complex entity-relationship knowledge graphs (e.g., Neo4j).
- Decomposing the modular monolith into independently deployed microservices.

---

## 4. Target Users and Personas

### 4.1 Persona 1: Individual Knowledge Worker ("Maya — Senior Product Manager")
- **Volume:** 120+ emails/day across stakeholders, customers, and vendors.
- **Pain Points:** Loses track of commitments buried in long CC chains; spends mornings triaging newsletters vs. urgent escalations.
- **Primary MailMaster Value:** Morning priority triage view, thread summaries with extracted action items, and natural-language search for past stakeholder commitments.

### 4.2 Persona 2: Software Engineer & Technical Architect ("Arjun — Staff Engineer")
- **Volume:** 60–100 emails/day including incident reports, RFC discussions, security advisories, and external vendor threads; uses AI coding/productivity tools daily.
- **Pain Points:** Needs fast retrieval of historical technical decisions and wants his IDE/desktop AI assistant to inspect relevant email context without copy-pasting.
- **Primary MailMaster Value:** Hybrid semantic+keyword retrieval for technical terms, verifiable email citations, and MCP integration into external AI workflows.

### 4.3 Persona 3: Engineering / Operations Manager ("Elena — Engineering Manager")
- **Volume:** 150+ emails/day spanning hiring, vendor procurement, executive updates, and escalations.
- **Pain Points:** Needs instant catch-up on 15-message threads before meetings and fast, context-accurate draft replies that preserve exact dates/numbers.
- **Primary MailMaster Value:** On-demand thread summarization, Smart Compose grounded in historical threads, and human-approved sending safety.

### 4.4 Persona 4: Freelancer & Independent Consultant ("Marcus — Cloud Consultant")
- **Volume:** Multiple client conversations, SOWs, invoices, and scope change requests across one or two Gmail accounts.
- **Pain Points:** Must never confuse details between Client A and Client B, and needs accurate recall of quoted pricing and deadlines.
- **Primary MailMaster Value:** Strict account-isolated RAG retrieval, citation-backed contract/quote lookup, and action-item extraction.

---

## 5. Functional Requirements

### 5.1 Authentication & User Management (`auth`)
- **FR-AUTH-01:** Users can register with email, password (Argon2id hashed), and full name; duplicate emails return `409 Conflict`.
- **FR-AUTH-02:** Users can log in to receive a short-lived JWT access token (15 minutes) and an HTTP-only/secure rotating refresh token (7 days).
- **FR-AUTH-03:** Refresh token rotation invalidates the prior token and detects token reuse, revoking the entire token family upon suspected theft.
- **FR-AUTH-04:** Users can log out to revoke active refresh tokens and inspect their authenticated profile (`GET /api/v1/auth/me`).

### 5.2 Gmail Integration (`gmail`)
- **FR-GMAIL-01:** Users initiate Google OAuth 2.0 authorization with CSRF-protected `state` (stored in Redis with TTL) and PKCE.
- **FR-GMAIL-02:** MailMaster requests least-privilege Gmail scopes (`gmail.readonly`, `gmail.compose`, `gmail.send`, `userinfo.email`).
- **FR-GMAIL-03:** OAuth access and refresh tokens are encrypted at rest using envelope/Fernet AES-256-GCM-authenticated encryption before persistence in `email_accounts`.
- **FR-GMAIL-04:** Users can view connected Gmail accounts, inspect sync health, and disconnect an account (which revokes the Google token where possible and deletes/purges account credentials and isolated data per retention rules).

### 5.3 Email Synchronization & Management (`gmail` + `emails`)
- **FR-SYNC-01:** Connecting a Gmail account automatically triggers an initial bounded background sync (configurable window, default last 30 days or up to 500 threads) via Celery.
- **FR-SYNC-02:** Incremental sync uses Gmail `users.history.list` starting from the persisted `sync_states.last_history_id`.
- **FR-SYNC-03:** If Gmail returns `404 HistoryId expired`, MailMaster gracefully falls back to a controlled reconciliation resync without duplicating existing messages.
- **FR-SYNC-04:** Email ingestion normalizes MIME parts, extracts clean plain text and sanitized HTML, preserves `Message-ID`, `In-Reply-To`, `References`, and `gmail_thread_id`, and strips/marks quoted reply history for cleaner chunking.
- **FR-SYNC-05:** Users can browse paginated threads, filter by category/priority/unread/label, view full thread timelines, and toggle read status.

### 5.4 Email Intelligence (`ai`)
- **FR-AI-01:** Newly ingested emails trigger asynchronous AI analysis producing a validated `EmailAnalysis` record containing:
  - `category` (`work`, `engineering`, `finance`, `scheduling`, `support`, `newsletter`, `personal`, `other`)
  - `priority` (`urgent`, `high`, `normal`, `low`) with short rationale
  - `sentiment` (`positive`, `neutral`, `negative`, `urgent_escalation`)
  - `intent` (`action_required`, `fyi`, `question`, `approval_request`, `scheduling`, `billing`, `automated_notification`)
  - `summary` (1–3 concise sentences)
  - `action_items` (JSON list of `{title, owner, due_date, confidence}`)
- **FR-AI-02:** Users can request on-demand thread-level summarization synthesizing key decisions, open questions, and next steps across all messages in a thread.
- **FR-AI-03:** Users can inspect token consumption and estimated USD cost per model/provider (`GET /api/v1/ai/usage`).

### 5.5 RAG Engine (`rag`)
- **FR-RAG-01:** Normalized emails are deterministically segmented into overlapping token-bounded chunks (`email_chunks`, default 400 tokens, 60-token overlap) prefixed with structured metadata headers (subject, sender, date).
- **FR-RAG-02:** Chunks are embedded asynchronously using the configured embedding model (`text-embedding-3-small`, 1536 dims) and stored in `email_embeddings` with `embedding_model` and `embedding_version`.
- **FR-RAG-03:** Search and Q&A queries execute hybrid retrieval scoped strictly by `user_id` and authorized `account_id`(s):
  - Dense cosine vector search via `pgvector` HNSW index (top-K = 30).
  - PostgreSQL full-text search via `ts_rank_cd` over `email_chunks.search_tsv` (top-K = 30).
  - Fusion via Reciprocal Rank Fusion (`RRF_k = 60`).
  - Optional reranking step to select top-N (`N = 8`) chunks within token budget.
- **FR-RAG-04:** Context builder wraps retrieved email excerpts inside explicit untrusted-data XML delimiters (`<retrieved_email_context>`) and attaches deterministic citation IDs (`[1]`, `[2]`, ...) mapped to real `email_id`, `thread_id`, `subject`, `sender`, and `received_at`.

### 5.6 Multi-Provider LLM Gateway (`ai/gateway`)
- **FR-LLM-01:** A unified `LLMGateway` interface abstracts chat completion, structured JSON output (via provider native structured outputs / tool schemas + Pydantic v2 validation), streaming, and embedding generation.
- **FR-LLM-02:** MVP implements `OpenAIProvider` and `AnthropicProvider` behind a common `BaseLLMProvider` contract.
- **FR-LLM-03:** Automatic exponential backoff retries handle transient `429`/`5xx` errors; configurable fallback routes requests from primary provider to secondary provider when primary fails or circuit-breaks.
- **FR-LLM-04:** Every LLM call writes an immutable `llm_usage_logs` record capturing `user_id`, `task_type`, `provider`, `model`, `prompt_version`, `input_tokens`, `output_tokens`, `latency_ms`, `estimated_cost_usd`, and `status`.

### 5.7 LangGraph AI Workflow (`ai/workflows`)
- **FR-GRAPH-01:** Conversational assistant requests execute through a compiled `StateGraph` with typed `AssistantState`.
- **FR-GRAPH-02:** Graph nodes include: `classify_intent` → conditional routing (`direct_answer` | `rag_search` | `thread_summary` | `draft_reply`) → `retrieve_context` → `rerank_and_build_context` → `generate_response` → `verify_citations`.
- **FR-GRAPH-03:** Citation verification checks that every `[n]` marker emitted in the assistant response corresponds to an actual retrieved chunk ID; invalid references are stripped or regenerated before returning to the user.

### 5.8 MCP Integration (`mcp`)
- **FR-MCP-01:** MailMaster exposes a Streamable HTTP MCP server at `/mcp` built on the official Python `mcp` SDK.
- **FR-MCP-02:** Every MCP connection/request authenticates a MailMaster JWT or scoped personal access token and resolves the owning `user_id`.
- **FR-MCP-03:** MCP exposes narrowly scoped tools (`search_emails`, `get_thread`, `ask_email_assistant`, `create_email_draft`, `list_action_items`), read-only resources (`mailmaster://threads/{thread_id}`, `mailmaster://accounts`), and reusable prompts (`triage_inbox`, `draft_executive_reply`).
- **FR-MCP-04:** MCP tools reuse internal application services and **never** expose an unapproved `send_email` tool; `create_email_draft` always produces a `pending_approval` draft requiring UI confirmation. Every MCP tool call is written to `audit_logs`.

### 5.9 Smart Compose & Human-in-the-Loop Sending (`drafts`)
- **FR-DRAFT-01:** Users can generate an AI reply draft for an existing thread or a new email by specifying intent, tone (`professional`, `concise`, `friendly`, `firm`), and optional RAG grounding toggle.
- **FR-DRAFT-02:** Drafts are persisted in `email_drafts` with `status = 'draft'` (or `'pending_approval'`), `source_citation_ids`, and `ai_generated = true`.
- **FR-DRAFT-03:** Users can edit recipient(s), subject, and body freely. Editing an approved draft automatically resets `status` back to `draft` to prevent TOCTOU (time-of-check to time-of-use) tampering.
- **FR-DRAFT-04:** Sending requires explicit user invocation of `POST /api/v1/drafts/{draft_id}/approve` followed by `send` (or combined UI confirmation modal calling approve then send with `draft_version` optimistic lock). Gmail send creates a corresponding `audit_logs` record.

### 5.10 AI Chat Assistant (`chat`)
- **FR-CHAT-01:** Users can create, list, inspect, and delete `chat_sessions` optionally pinned to a specific `email_account_id` or `thread_id`.
- **FR-CHAT-02:** Users can send messages and receive streamed (SSE) or standard responses containing the answer text, structured `citations[]`, and `usage` metadata.
- **FR-CHAT-03:** Multi-turn conversation history is persisted in `chat_messages` and windowed safely within the model's context budget.

### 5.11 Frontend Dashboard (`frontend`)
- **FR-UI-01:** Vue 3 SPA using Composition API, Pinia, Vue Router, Tailwind CSS, shadcn-vue, and TanStack Query.
- **FR-UI-02:** Provides dedicated views for Auth (Login/Register), Inbox Dashboard (3-pane or adaptive triage list + preview + AI insights), Thread Detail + Smart Compose drawer, AI Chat Assistant with interactive source citation cards, and Settings (Gmail accounts, AI provider preferences, usage metrics).

---

## 6. Non-Functional Requirements

### 6.1 Security
- Passwords hashed with Argon2id; JWT signed with HS256/RS256 using rotated/environment-managed keys (`>= 256-bit` entropy).
- Gmail OAuth refresh/access tokens encrypted at rest using authenticated AES-256 (Fernet/AES-GCM) with key versioning support.
- Strict account-level data isolation: every SQL query on user/account-scoped tables includes `user_id` ownership verification.
- Untrusted email HTML sanitized before rendering (DOMPurify on frontend; bleach/nh3 stripping on backend); email text delimited as untrusted data in all LLM prompts to mitigate prompt injection.
- Rate limiting via Redis sliding window on auth endpoints (`10 req/min/IP`), AI chat/draft endpoints (`30 req/min/user`), and sync triggers (`6 req/min/user`).

### 6.2 Performance
- Inbox list (`GET /api/v1/threads`) p95 latency ≤ 150 ms for accounts with up to 50,000 synced messages (keyset/cursor or indexed pagination).
- Hybrid RAG retrieval (vector + FTS + RRF fusion, excluding LLM generation) p95 latency ≤ 350 ms for up to 100,000 chunks per account.
- First SSE token for AI chat assistant p95 ≤ 2.0 s (excluding external LLM provider cold spikes).
- Background incremental sync completes typical delta (< 50 messages) in ≤ 10 s end-to-end.

### 6.3 Reliability & Idempotency
- Email ingestion is strictly idempotent on `(account_id, gmail_message_id)`.
- Celery tasks use `acks_late=True`, bounded exponential backoff retries with jitter, dead-letter logging on terminal failure, and per-account distributed sync locks in Redis to avoid concurrent duplicate syncs.

### 6.4 Scalability
- Stateless FastAPI API replicas and horizontally scalable Celery worker pools behind Nginx and Redis.
- Heavy I/O (Gmail API, LLM API, embedding generation) never blocks the FastAPI event loop; batch tasks run asynchronously in Celery workers.

### 6.5 Maintainability
- Modular monolith with strict dependency direction (`api` → `modules/*/service.py` → `repository.py` + `gateway`).
- Ruff linting/formatting, strict Mypy type checking on backend, ESLint + `vue-tsc` on frontend, and ≥ 90% branch coverage on core business logic.

### 6.6 Availability & Graceful Degradation
- Health probes distinguish liveness (`/api/v1/health/live`) from dependency readiness (`/api/v1/health/ready` verifying PostgreSQL and Redis).
- If an LLM provider experiences an outage, core email reading, keyword search, and Gmail sync remain fully operational, and the LLM gateway automatically falls back to the secondary provider if configured.

### 6.7 Observability
- Structured JSON logging with correlation `request_id`, `user_id`, and Celery `task_id` across API and workers.
- OpenTelemetry tracing instrumentation across FastAPI, SQLAlchemy, httpx, and Celery, plus persisted `llm_usage_logs` and `audit_logs`.

### 6.8 Data Privacy
- Users can disconnect a Gmail account and purge all synced emails, embeddings, analyses, and drafts for that account.
- LLM API calls explicitly avoid sending PII in logs; email content is never logged at `INFO` level in application logs.

---

## 7. MVP Acceptance Criteria

| ID | Feature | Acceptance Criteria | Priority |
|---|---|---|---|
| **AC-01** | User Registration & Login | `POST /api/v1/auth/register` hashes password with Argon2id and returns `201`; `POST /api/v1/auth/login` issues JWT access token (15m) + refresh token (7d). Invalid credentials return `401` without leaking user existence. | P0 |
| **AC-02** | Refresh Token Rotation | `POST /api/v1/auth/refresh` issues a new refresh token and marks the previous token rotated. Reusing a rotated token revokes the entire token family and returns `401`. | P0 |
| **AC-03** | Google OAuth 2.0 Connection | `GET /api/v1/gmail/oauth/authorize` sets CSRF state in Redis (10m TTL); callback validates state, exchanges code for tokens, encrypts tokens at rest in `email_accounts`, and dispatches initial sync task. | P0 |
| **AC-04** | Initial & Incremental Sync | Initial sync imports threads/messages idempotently; incremental sync uses `sync_states.last_history_id` and handles `404` history expiration via safe reconciliation. Re-running sync on identical Gmail state produces 0 duplicate rows. | P0 |
| **AC-05** | Email Normalization & Threading | MIME multipart emails resolve clean plain text + sanitized HTML (`nh3`); emails link accurately to `email_threads` by `gmail_thread_id`; quoted replies are separated from new message content. | P0 |
| **AC-06** | Structured AI Intelligence | Celery analysis worker produces schema-validated `email_analyses` (priority, category, sentiment, intent, summary, action_items) via LLM Gateway; malformed LLM output retries and falls back cleanly without crashing the worker. | P0 |
| **AC-07** | Multi-Provider LLM Gateway | Gateway supports both `OpenAIProvider` and `AnthropicProvider` behind `BaseLLMProvider`; switching providers in user settings routes subsequent requests without code changes; every call logs tokens and cost to `llm_usage_logs`. | P0 |
| **AC-08** | Hybrid RAG Retrieval | Queries execute both `pgvector` cosine search and PostgreSQL `tsvector` FTS filtered strictly by `user_id` and `account_id`, fused via RRF (`k=60`) and reranked. Cross-account retrieval returns 0 results from other users/accounts in automated isolation tests. | P0 |
| **AC-09** | Grounded AI Chat & Citations | LangGraph assistant executes intent routing and returns answers with `[n]` citations mapped to real `email_id` + `thread_id` + excerpt. Citation verifier rejects/strips any fabricated citation ID. | P0 |
| **AC-10** | Smart Compose & Approval Gate | `POST /api/v1/drafts/generate` creates a draft (`status='draft'`). Calling `POST /api/v1/drafts/{id}/send` on an unapproved draft returns `409 Conflict`. Only explicitly approved, unmodified drafts send via Gmail API and emit an `audit_logs` entry. | P0 |
| **AC-11** | Custom MCP Server | Streamable HTTP server at `/mcp` rejects unauthenticated requests (`401`), enforces user ownership on all tools/resources, logs every invocation in `audit_logs`, and provides no unapproved send capability. | P1 |
| **AC-12** | Vue 3 Dashboard & UX | Users can sign in, connect Gmail, trigger sync, browse triaged inbox, view thread summaries/action items, chat with citations, and approve/send Smart Compose drafts on desktop and tablet viewports without console errors. | P0 |
| **AC-13** | Containerized Deployment | `docker compose up --build` starts PostgreSQL (with `pgvector`), Redis, FastAPI API, Celery worker, Vue frontend, and Nginx reverse proxy; health endpoints report `200 OK` within 60s. | P0 |
| **AC-14** | CI Quality & Coverage Gate | GitHub Actions passes Ruff, strict Mypy, ESLint, `vue-tsc`, Vitest, Pytest (Python 3.12 & 3.13) with ≥ 90% critical business logic branch coverage, Bandit, `pip-audit`, and Docker build checks. | P0 |

---

## 8. User Stories

### Story 1: Connecting Gmail Safely
**As a** professional user,  
**I want** to connect my Gmail account via Google OAuth 2.0,  
**So that** MailMaster can sync and analyze my inbox without me sharing my Google password.

- **Given** I am logged into MailMaster and navigate to **Settings → Integrations**,  
  **When** I click **Connect Gmail Account** and complete Google consent,  
  **Then** MailMaster stores my encrypted OAuth tokens, displays my connected email address with status `Syncing`, and begins importing recent threads in the background.

### Story 2: Automated Inbox Triage
**As a** busy manager,  
**I want** incoming emails automatically categorized and prioritized with extracted action items,  
**So that** I can immediately focus on urgent escalations and commitments.

- **Given** a new customer escalation email arrives and is synced,  
  **When** the background intelligence pipeline completes,  
  **Then** the inbox list displays an `Urgent` badge, `Support` category tag, a 2-sentence summary in the preview panel, and the extracted action item with due date.

### Story 3: Citation-Grounded Historical Q&A
**As a** software engineer,  
**I want** to ask the AI assistant about past technical agreements in my inbox,  
**So that** I can verify exact commitments without manually searching 20 threads.

- **Given** I have synced emails containing vendor SLA discussions across multiple threads,  
  **When** I ask *"What uptime SLA did VendorX commit to in Q2?"* in the AI Assistant,  
  **Then** the assistant retrieves the relevant email chunks via hybrid RAG, returns the exact SLA percentage, and renders clickable citation pills linking to the specific source emails.

### Story 4: Context-Aware Reply with Human Approval
**As a** consultant,  
**I want** MailMaster to draft a reply using relevant past emails and wait for my approval before sending,  
**So that** I save drafting time without risking an inaccurate or unauthorized email being sent to a client.

- **Given** I am viewing an email thread asking for updated project milestones,  
  **When** I click **Draft AI Reply** with tone set to `Professional` and RAG context enabled,  
  **Then** Smart Compose generates an editable draft in `draft` state; if I edit the body and click **Approve & Send**, MailMaster transitions the draft to `approved`, sends it via Gmail, records an audit log, and displays confirmation.

### Story 5: External AI Client via MCP
**As a** developer using an MCP-compatible AI tool,  
**I want** to connect my client to MailMaster's MCP server,  
**So that** my assistant can search my emails and prepare drafts safely inside my workflow.

- **Given** I have configured my MCP client with MailMaster's `/mcp` URL and my authenticated bearer token,  
  **When** the client calls the `create_email_draft` tool,  
  **Then** MailMaster validates ownership, creates the draft in `pending_approval` state, logs the MCP execution in `audit_logs`, and instructs the user to review and send it inside MailMaster.

---

## 9. Future Roadmap (Post-MVP)

The following capabilities are planned for post-MVP releases (`v1.1+`) and intentionally excluded from `v1.0`:

| Target Phase | Planned Capability | Architectural Readiness in v1.0 |
|---|---|---|
| **v1.1** | Microsoft Outlook / Microsoft Graph API provider | `email_accounts.provider` enum and adapter pattern allow adding an `OutlookSyncProvider` alongside `GmailSyncProvider`. |
| **v1.2** | Calendar & meeting scheduling intelligence | Modular `modules/` layout supports adding a `calendar` domain module and LangGraph scheduling tools. |
| **v1.3** | Attachment parsing (PDF, DOCX, CSV indexing in RAG) | `email_chunks` can be extended with `source_type` (`body` vs `attachment`) without altering retrieval interfaces. |
| **v1.4** | Team workspaces, shared inboxes, and enterprise RBAC/SSO | Introducing an `organizations` entity and `org_id` scoping layer on top of existing `user_id` isolation. |
| **v1.5** | Visual rule-based automation & Slack digest notifications | Event hooks emitted by `sync_service` and `intelligence_service` can drive rule triggers and outbound webhook adapters. |
| **v2.0** | Kubernetes Helm deployment & horizontal read replicas | Stateless API/worker containers and externalized PostgreSQL/Redis configuration already follow 12-factor principles. |
