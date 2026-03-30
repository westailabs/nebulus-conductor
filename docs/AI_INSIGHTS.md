# Project AI Insights (Long-Term Memory)

## Purpose
This document serves as the **Long-Term Memory** for AI agents working on **nebulus-conductor**. It captures project-specific behavioral nuances, recurring pitfalls, and architectural decisions that are not strictly "rules" (in `AI_DIRECTIVES.md`) but are critical for maintaining continuity.

## 1. Architectural Patterns

*   **Artifacts**: Always update `task.md` before tool calls when starting a new phase.

## 2. Recurring Pitfalls
*   **Testing**: Do not assume tests pass; always checking logs.
*   **Dependencies**: Check `requirements.txt` before adding new libraries.

## 3. Workflow Nuances
*   **Verification**: Trust the test runner (`pytest` or `verify.yml`) over your assumptions.

---

## 4. BDD Specifications — v1 (2026-03-29)

**File written:** `docs/specs/v1-bdd-specs.md`
**Committed:** `feat: v1 BDD specifications — revised post architectural review` (initial commit on new repo)

### What was written

Full Gherkin BDD specification for nebulus-conductor v1. Covers 6 features with 34 scenarios total:

| Feature | Scenarios |
|---------|-----------|
| Pre-Invocation Policy Gate | 7 (including concurrent eval, malformed request, no-match default-deny, fail-closed) |
| Data Classification | 5 (L1/L2/L3 classification, false-positive override flow) |
| Agent Registration | 7 (duplicate, deregistration, expiry, revocation, unapproved sponsor) |
| Audit Trail | 6 (log-after-exec, log-on-block, unavailable=fail-closed, execution failure event, tamper-evident) |
| Policy Management | 6 (startup errors, empty policy, hot-reload, invalid hot-reload) |
| HTTP Transport | 4 (POST, timeout, 503 on fail-closed, 400 on malformed) |

### Key architectural decisions encoded

1. **Default-deny** — anything not explicitly permitted is blocked; no permissive fallback
2. **Fail-closed** — any eval error, audit log failure, or policy load failure results in all traffic blocked (configurable via `policy.fail_behavior`)
3. **HTTP on localhost only** — REST API transport, no external network binding in v1
4. **Authority scope hierarchy** — `global > organization > domain > local`
5. **request_hash** — SHA-256 of full request payload minus L3 content (defined in spec as a comment)
6. **Hot-reload interval** — configurable via `policy.reload_interval`, default 30s

### Notable design choices in the spec

- L2 data is allowed internal, blocked external, and **redacted** in audit logs (not blocked internal)
- False positive overrides are **human-gated** — Conductor logs `override_requested=true` but does NOT auto-allow
- Audit entries are written **after** tool execution, not before, and include `execution_status`
- Audit log unavailability is a hard fail-closed condition (not a degraded mode)
- Agent deregistration is graceful: in-flight requests complete before removal
- Registration expiry and admin revocation are separate mechanisms with different semantics
- Empty policy file (zero rules) is a valid startup state (default-deny with warning), unlike missing/invalid policy file (refuses to start)
- `request_hash` excludes L3 content before hashing to prevent secrets from appearing in audit trail even in hashed form
