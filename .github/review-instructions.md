# Code Review Guidelines

These instructions define what the automated reviewer must evaluate on every pull request, regardless of language, framework, or file type.

## Project context
- This repository contains nightly batch jobs and reporting rollups, not OLTP queries.
- DO NOT flag: high statement_timeout values, batch DDL patterns (DROP/CREATE,
  TRUNCATE/INSERT), index overhead on tables that are dropped and recreated in
  the same script, or ANALYZE timing.
- DO flag: incorrect arithmetic, missing transaction wrapping where atomicity
  matters, schema mismatches, missing error handling on data that may be NULL
  or zero in upstream sources.

## Quick Reference

| Priority | Category | Action |
|----------|----------|--------|
| 1 | Security | Flag injection, auth gaps, leaked secrets, weak crypto, unsafe dependencies |
| 2 | Correctness | Flag null derefs, swallowed errors, races, resource leaks, boundary bugs |
| 3 | Performance | Flag N+1 queries, unbounded fetches, hot-path waste, missing caching |
| 4 | Maintainability | Flag dead code, high complexity, misleading names, copy-paste duplication |
| 5 | Observability | Flag missing logging, monitoring gaps, absent correlation IDs, no retry logic |

---

## Priority 1 — Security

Flag any change that introduces or fails to mitigate a security risk.

- **Injection**: SQL injection, command injection, LDAP injection, template injection, XSS (reflected/stored/DOM). Verify all user-controlled inputs are parameterized or escaped before reaching sinks.
- **Authentication & Authorization**: Missing or bypassed auth checks, hardcoded credentials, secrets in source, tokens without expiry, privilege escalation paths.
- **Secrets & Sensitive Data**: API keys, connection strings, passwords, PII, or tokens committed in code, config, logs, or error messages. Ensure secrets come from environment variables, vaults, or managed identity — never literals.
- **Cryptography**: Use of deprecated algorithms (MD5, SHA-1 for integrity, DES, RC4), insufficient key lengths, missing salt/IV, deterministic encryption where randomness is required.
- **Dependency & Supply Chain**: New dependencies without pinned versions, known-vulnerable packages, importing from untrusted registries.
- **Path Traversal & File Access**: Unsanitized user input used in file paths, directory traversal (`../`), unrestricted file uploads.
- **Deserialization**: Untrusted data deserialized without schema validation or allow-listing of types.
- **SSRF / Open Redirect**: User-controlled URLs passed to HTTP clients or redirect targets without validation against an allow-list.

---

## Priority 2 — Correctness & Reliability

Flag logic that will cause incorrect behavior, data loss, or runtime failures.

- **Null / Undefined Handling**: Dereferencing potentially null values, missing null checks on API responses or optional fields.
- **Error Handling**: Swallowed exceptions, overly broad catch blocks, missing cleanup in error paths.
- **Race Conditions & Concurrency**: Shared mutable state without synchronization, non-atomic check-then-act.
- **Resource Leaks**: Connections, handles, streams, or transactions not closed in all paths (including error paths).
- **Boundary Conditions**: Off-by-one, overflow/underflow, division by zero, empty collection access.
- **Data Integrity**: Schema mismatches, missing constraints, writes without validation.
- **Contract Violations**: Breaking public API changes without version bumps.

---

## Priority 3 — Performance & Cost

Flag changes that introduce unnecessary cost, latency, or resource consumption in production.

- **N+1 Queries / Chatty I/O**: Database or API calls inside loops that should be batched or joined.
- **Missing Pagination / Limits**: Unbounded queries that will degrade as data grows.
- **Hot-Path Waste**: Expensive operations repeated in tight loops that could be hoisted.
- **Memory**: Loading full datasets into memory when streaming is feasible.
- **Caching**: Missing cache for expensive idempotent lookups, or cache without TTL/invalidation.
- **Cloud Cost**: Over-provisioned IaC resources, missing auto-scale, always-on resources that could be serverless.

---

## Priority 4 — Maintainability & Clarity

Only flag when the issue will cause real confusion or maintenance burden — do NOT flag minor style preferences.

- **Dead Code**: Unreachable branches, commented-out blocks, unused imports/variables that add noise.
- **Complexity**: Deeply nested conditionals (>3 levels), functions exceeding ~50 lines of logic, God objects or methods doing too many things.
- **Misleading Names**: Variables or functions whose name contradicts their behavior (e.g., `isValid` that also mutates state).
- **Missing Context for Future Maintainers**: Non-obvious business logic without a brief comment, magic numbers without named constants, undocumented public API contracts.
- **Copy-Paste Duplication**: Identical logic repeated across the diff that should be extracted to a shared function.

---

## Priority 5 — Observability & Operability

Flag gaps that would make production incidents harder to detect or diagnose.

- **Logging**: Missing log statements around error paths or critical state transitions, logging sensitive data (PII, tokens), inconsistent log levels.
- **Monitoring**: New features without corresponding metrics, health-check endpoints, or alerting hooks.
- **Tracing / Correlation**: Async operations or cross-service calls without propagating correlation IDs or trace context.
- **Failure Modes**: Missing retry/backoff for transient failures, no circuit breaker for external dependencies, no graceful degradation path.

---

## What NOT to flag

Do not comment on any of the following — these are out of scope for this reviewer:

- Code formatting, whitespace, indentation, or line length (handled by formatters/linters).
- Naming style preferences (camelCase vs. snake_case) unless the name is actively misleading.
- Missing documentation or docstrings on internal/private code.
- Test coverage gaps (unless the test itself has a correctness bug).
- Subjective architectural preferences when the existing pattern is consistent.
- TODOs or FIXMEs — acceptable unless they replace actual error handling or hide a known defect that will reach production without being addressed.

---

## Behavioral Rules

- **Be precise**: Every finding must reference specific lines and explain *why* it's a problem, not just *what* the code does.
- **Be actionable**: If you can propose a concrete fix, do so. If the fix is non-trivial, describe the approach.
- **Assume production traffic**: Evaluate the code as if it will serve real users at scale immediately after merge.
- **Respect existing patterns**: If the codebase already uses a particular pattern consistently, don't flag new code for following that same pattern.
- **State uncertainty explicitly**: If you're unsure whether something is a bug, still comment on it but clearly state your uncertainty rather than presenting it as fact or staying silent.
- **One finding per issue**: Don't combine unrelated problems into a single comment.

