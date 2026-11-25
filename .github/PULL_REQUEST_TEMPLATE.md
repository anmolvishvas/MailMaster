## Summary

<!-- Provide a concise summary of what changed in this pull request. -->

## Motivation

<!-- Explain why this change is necessary and link any related tasks (e.g., MM-101) or issues. -->

## Type of Change

- [ ] `feat`: New feature or user-facing capability
- [ ] `fix`: Bug fix
- [ ] `refactor`: Code restructuring without behavior change
- [ ] `perf`: Performance improvement
- [ ] `docs`: Documentation updates
- [ ] `test`: Adding or updating tests
- [ ] `chore`: Build, tooling, dependency, or maintenance change
- [ ] `security`: Security hardening or vulnerability remediation

## Architectural Impact

<!-- Does this change affect module boundaries, database schema, API contracts, RAG/AI behavior, or existing ADRs? -->

- [ ] No architectural impact
- [ ] Affects module boundaries, schema, or ADRs (explain below and update `docs/ARCHITECTURE.md` / `docs/MEMORY.md`)

## Implementation Details

<!-- Describe key design choices, touched modules, Alembic migrations, or configuration changes. -->

## Testing

- [ ] Unit tests added or updated
- [ ] Integration tests added where applicable
- [ ] Frontend tests added where applicable
- [ ] Coverage requirements maintained (`>= 90%` branch coverage on critical business logic)
- [ ] No regressions identified

## Code Quality

- [ ] Ruff checks pass (`ruff check .`)
- [ ] Ruff formatting passes (`ruff format --check .`)
- [ ] Mypy strict checks pass (`mypy --strict app`)
- [ ] Frontend linting and type checks pass (`npm run lint && npx vue-tsc --noEmit`)
- [ ] No new warnings introduced
- [ ] No hardcoded secrets or credentials
- [ ] No unnecessary dependencies added

## Documentation

- [ ] Relevant documentation updated (`docs/PRD.md`, `docs/ARCHITECTURE.md`, `docs/DESIGN.md`, `docs/RULES.md`, `README.md`)
- [ ] `docs/TASKS.md` updated for verified completed tasks only
- [ ] `docs/MEMORY.md` updated if architecture or project state changed
- [ ] ADR added in `docs/MEMORY.md` if required

## Security

- [ ] Authentication and authorization reviewed
- [ ] Account-level data isolation (`user_id` / `account_id`) verified where applicable
- [ ] Untrusted email content sanitization and prompt-injection boundaries preserved
- [ ] External actions (Gmail send, OAuth link/unlink, MCP tools) remain auditable in `audit_logs`
- [ ] Sensitive data (OAuth tokens, JWT secrets, PII) is protected and never logged

## Breaking Changes

<!-- Explicitly document any breaking API, schema, configuration, or behavioral changes and required migration steps, or state "None". -->
None.
