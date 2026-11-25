# MailMaster — Living Project Journal (`MEMORY.md`)

**Source of Truth:** This document tracks verified project state, architectural decisions, active work, and handoff instructions across development sessions. Update it after verified implementation changes; never delete historical ADRs.

---

## 1. Project Status

- **Project Name:** MailMaster
- **Current Version:** `0.1.0-docs` (Pre-code Documentation Baseline)
- **Current Phase:** Phase 0 -> Phase 1 Transition (Documentation & CI Governance Complete; Code Scaffolding Pending)
- **Overall Progress:** Documentation & CI/PR governance established (`docs/`, `.github/`, `README.md`). Application code (`backend/`, `frontend/`, `infrastructure/`) not yet scaffolded.
- **Last Updated:** 2026-11-25
- **Active Development Milestone:** Initial Documentation & Architecture Baseline PR (PR #1)

---

## 2. Completed Work Tracker

### Phase 0 — Architecture, Specifications & CI Governance (PR #1)
- [x] `README.md` — Project overview, stack, documentation links, quick start, testing, and security posture (clearly distinguishing planned vs. implemented status).
- [x] `docs/PRD.md` — Product requirements, personas, functional/non-functional specs, MVP acceptance criteria (`AC-01`..`AC-14`), user stories, and post-MVP roadmap.
- [x] `docs/ARCHITECTURE.md` — Modular monolith architecture, Mermaid system & ER diagrams, 15-table schema specifications, sync/AI/RAG/MCP/security/infrastructure designs, and canonical directory tree.
- [x] `docs/RULES.md` — Enforceable Python 3.12+, async, AI, RAG, MCP, testing (`>= 90%` branch coverage target), and Git standards.
- [x] `docs/DESIGN.md` — Dark-first UI design system, hex tokens, typography, 18 component specs, screen layouts, and responsive rules.
- [x] `docs/TASKS.md` — 7-phase dependency-ordered implementation roadmap (`MM-101`..`MM-705`).
- [x] `docs/MEMORY.md` — Living journal and initial ADR log (`ADR-001`..`ADR-006`).
- [x] `.github/PULL_REQUEST_TEMPLATE.md` — PR quality, testing, documentation, and security checklist.
- [x] `.github/workflows/ci.yml` — Lint, strict Mypy, frontend quality, Python 3.12/3.13 test matrix, security audit (`Bandit` + `pip-audit`), and Docker validation pipeline.
- [x] `.github/workflows/pr-audit.yml` — Conventional Commit PR title validation and automatic path-based labeling.

### Phase 1–7 Implementation Tasks (`docs/TASKS.md`)
- [ ] `MM-101` through `MM-705`: Pending implementation (see [TASKS.md](./TASKS.md)).

---

## 3. Active Work

- **Current Focus:** Delivering PR #1 (documentation-only foundation + GitHub Actions governance) on `anmolvishvas/MailMaster`.
- **Current Blockers:** None.
- **Next Scheduled Milestone:** Phase 1 code scaffolding (`MM-101` through `MM-108`).

---

## 4. Architectural Decision Records (ADRs)

### ADR-001: Modular Monolith Over Microservices for MVP
- **Status:** Accepted
- **Date:** 2026-11-25
- **Context:** MailMaster requires ACID consistency across synced emails, threads, drafts, and audit logs, plus low-latency in-process sharing between REST API routes and the MCP server.
- **Decision:** Implement MailMaster as a domain-oriented modular monolith (`backend/app/modules/*`) paired with Celery background workers (`backend/app/workers/`) rather than separate microservices.
- **Consequences:** Simplifies deployment (`docker compose up`), preserves transactional integrity, avoids inter-service RPC overhead, and keeps future service extraction feasible via strict `service.py` boundaries.

### ADR-002: PostgreSQL 17 + pgvector + FTS for Hybrid RAG (No Separate Vector DB)
- **Status:** Accepted
- **Date:** 2026-11-25
- **Context:** Email RAG requires both semantic similarity and exact keyword/identifier matching, alongside strict per-account ownership filtering (`user_id`, `account_id`) and transactional consistency with deleted/disconnected accounts.
- **Decision:** Store `email_chunks` (with `GIN`-indexed `tsvector`) and `email_embeddings` (`vector(1536)` with HNSW index) inside PostgreSQL 17 + `pgvector`, combining results via Reciprocal Rank Fusion (`k = 60`).
- **Consequences:** Eliminates dual-write synchronization drift with external vector databases, guarantees transactional cascade deletion on account disconnect, and enforces account isolation inside SQL CTEs before fusion.

### ADR-003: Multi-Provider LLM Gateway With Strict Pydantic Structured Outputs
- **Status:** Accepted
- **Date:** 2026-11-25
- **Context:** Depending directly on OpenAI or Anthropic SDKs inside business logic creates vendor lock-in and scattered retry/cost-tracking logic.
- **Decision:** Introduce `LLMGateway` (`app/modules/ai/gateway/`) exposing `BaseLLMProvider` implemented by `OpenAIProvider` and `AnthropicProvider`, enforcing Pydantic v2 validation, versioned prompts, token/cost logging (`llm_usage_logs`), and fallback routing.
- **Consequences:** Minor upfront adapter boilerplate, repaid by uniform observability, testability with fake providers, and resilience to single-vendor outages.

### ADR-004: Mandatory Human-in-the-Loop Approval Gate for Email Sending
- **Status:** Accepted
- **Date:** 2026-11-25
- **Context:** Email bodies are untrusted external input susceptible to prompt injection; autonomous sending creates unacceptable reputational and security risk.
- **Decision:** Enforce an explicit `email_drafts` state machine (`draft` -> `pending_approval` -> `approved` -> `sent`) with optimistic `draft_version` locking; any edit resets approval, and MCP tools may only create `pending_approval` drafts (never send).
- **Consequences:** Adds one explicit user confirmation click before sending; completely prevents unauthorized AI/MCP outbound emails.

### ADR-005: Streamable HTTP Transport for Embedded Custom MCP Server
- **Status:** Accepted
- **Date:** 2026-11-25
- **Context:** External AI clients need authenticated, network-accessible integration with MailMaster without maintaining a separate local stdio binary or duplicating database logic.
- **Decision:** Mount an official Python `mcp` SDK server using Streamable HTTP transport at `/mcp` inside the FastAPI application, sharing bearer token authentication and domain services.
- **Consequences:** Zero duplicated business logic, centralized `audit_logs` tracking, and seamless deployment behind Nginx.

### ADR-006: Path-Guarded CI Execution Until Phase 1 Code Scaffolding Lands
- **Status:** Accepted
- **Date:** 2026-11-25
- **Context:** PR #1 introduces documentation and GitHub Actions workflows before `backend/pyproject.toml`, `frontend/package.json`, and `docker-compose.yml` are scaffolded in `MM-101`–`MM-107`.
- **Decision:** Configure `ci.yml` jobs to detect project manifest presence (`backend/pyproject.toml`, `frontend/package.json`, `docker-compose.yml`) and emit a clear skip notice when absent, while strictly enforcing every check as soon as those files exist.
- **Consequences:** Documentation-only PR #1 can pass CI cleanly without disabling branch checks or weakening enforcement once Phase 1 code is committed.

---

## 5. Important Engineering Context

- **Canonical Stack:** Python 3.12+, FastAPI, Pydantic v2, SQLAlchemy 2.0 (async + `asyncpg`), Alembic, PostgreSQL 17+, `pgvector` (`vector(1536)`), Redis, Celery, LangGraph, OpenAI + Anthropic, Official Python MCP SDK, Vue 3 + TypeScript + Vite + Tailwind + `shadcn-vue` + Pinia + TanStack Query, Docker Compose + Nginx.
- **Implemented Modules:** None yet (documentation baseline only).
- **Known Limitations (Current State):** Application runtime, migrations, tests, and Dockerfiles do not exist yet and will be built sequentially starting at `MM-101`.
- **Critical Technical Constraints:**
  - `email_embeddings` dimension (`1536`) must stay synchronized with `EMBEDDING_DIMENSIONS` and `text-embedding-3-small` unless a versioned re-indexing migration is added.
  - All user/account queries must bind `user_id` (and `account_id`) in repositories and RAG SQL CTEs.
  - Email HTML must be sanitized via `nh3` on backend and `DOMPurify` on frontend.

---

## 6. Immediate Next Actions

Execute Phase 1 sequentially per `docs/TASKS.md`:
1. **`MM-101`:** Scaffold `backend/`, `frontend/`, `infrastructure/`, `scripts/`, `tests/`, `.gitignore`, `.env.example`, and `LICENSE`.
2. **`MM-102`:** Implement `app/core/config.py`, `app/core/exceptions.py`, and `app/core/logging.py`.
3. **`MM-103`:** Initialize FastAPI app (`app/main.py`) and `/api/v1/health/{live,ready}` endpoints.
4. **`MM-104`:** Configure async SQLAlchemy 2.0, `pgvector` init SQL, and Alembic migrations.
5. **`MM-105`:** Configure Redis client and Celery worker app.

---

## 7. AI Agent Handoff Instructions

Every future AI coding agent or contributor starting a session on MailMaster **MUST** read the following files in order before proposing or writing code:

1. `README.md`
2. `docs/PRD.md`
3. `docs/ARCHITECTURE.md`
4. `docs/RULES.md`
5. `docs/TASKS.md`
6. `docs/MEMORY.md`

### Session Execution Protocol
- Verify which tasks in `docs/TASKS.md` are currently `[x]` vs `[ ]` and inspect existing source files rather than assuming architecture is already coded.
- Follow `docs/RULES.md` strictly (thin routers, `LLMGateway` abstraction, account isolation in SQL, `>= 90%` branch coverage on critical logic, Ruff + Mypy strict compliance).
- After completing and verifying implementation work (tests + linters passing), update `docs/TASKS.md` checkboxes and `docs/MEMORY.md` (`Completed Work Tracker`, `Active Work`, `Immediate Next Actions`, and new `ADR-xxx` if structural decisions were made).
- **Never** overwrite historical ADRs or mark tasks `[x]` based on unverified or partial changes.
