# nebulus-conductor v1 — BDD Specifications

Revised: 2026-03-29 (post architectural review)

---

## Architectural Decisions (v1)

| # | Decision | Value |
|---|----------|-------|
| AD-01 | **Default stance** | Default-deny. Any tool call not explicitly permitted by an active policy rule is blocked. |
| AD-02 | **Fail behavior** | Fail-closed. If Conductor cannot evaluate a request (eval error, audit log unavailable, policy load failure), it blocks the request and logs the reason. Configurable per deployment via `policy.fail_behavior`. |
| AD-03 | **Transport** | HTTP on localhost (REST API). Agents submit requests via HTTP POST to the Conductor endpoint. No external network exposure in v1. |

### Supporting Decisions

- **Authority scope hierarchy:** `global > organization > domain > local`. A rule at a higher scope overrides a conflicting rule at a lower scope.
- **request_hash:** SHA-256 of the full request payload, excluding any content classified L3 (Restricted/Secret) prior to hashing. Used for tamper-evident audit records.
- **Hot-reload interval:** Configurable via `policy.reload_interval` (default: 30s). Conductor checks for policy file changes on this interval and reloads atomically.
- **Registration TTL:** Configurable via `registry.ttl` (default: 24h). Expired registrations are treated as unregistered.
- **Evaluation timeout:** Configurable via `conductor.eval_timeout_ms` (default: 500ms). Requests that exceed this timeout are treated as eval failures (fail-closed).

---

## Feature: Pre-Invocation Policy Gate

Conductor evaluates every tool call request before execution. The policy gate is the authoritative allow/block decision point.

> Policy state is read-only per evaluation. Concurrent evaluations do not mutate shared state.

```gherkin
Feature: Pre-Invocation Policy Gate

  Background:
    Given Conductor is running with a valid policy loaded
    And the policy default stance is "deny"

  Scenario: Allow a safe tool call
    Given a registered agent "agent-alpha" with scope "read:files"
    When the agent submits a tool call request for tool "file.read" with path "/home/user/docs/notes.txt"
    And the policy contains a rule permitting "agent-alpha" to call "file.read" within scope "read:files"
    Then Conductor returns decision "allow"
    And the tool call proceeds to execution

  Scenario: Block a credential-exfiltrating tool call
    Given a registered agent "agent-beta" with scope "read:files"
    When the agent submits a tool call request for tool "http.post" targeting "https://external.attacker.com"
    And the tool call payload contains L3-classified credential data
    Then Conductor returns decision "block"
    And the reason is "data-classification-violation"
    And no data is transmitted externally

  Scenario: Block a tool call that exceeds the agent's authorized scope
    Given a registered agent "agent-gamma" with scope "read:files"
    When the agent submits a tool call request for tool "shell.exec" with command "rm -rf /var/data"
    And the agent's policy scope does not include "exec:shell"
    Then Conductor returns decision "block"
    And the reason is "unauthorized-scope"

  Scenario: Policy evaluation failure triggers fail-closed behavior
    Given a registered agent "agent-delta" with scope "read:files"
    And the policy evaluation engine encounters an internal error during rule matching
    When the agent submits a tool call request for tool "file.read"
    Then Conductor blocks the request
    And Conductor returns decision "block"
    And the reason is "eval-error"
    And the error is logged to the audit trail with severity "error"

  Scenario: Malformed request is rejected
    Given a registered agent "agent-epsilon" with scope "read:files"
    When the agent submits a tool call request that is missing required fields (e.g., tool name or agent ID)
    Then Conductor returns HTTP 400
    And the error code is "MalformedRequest"
    And the request is not forwarded to the policy engine

  Scenario: No matching policy rule triggers default-deny
    Given a registered agent "agent-zeta" with scope "read:files"
    And the policy contains no rule that matches the submitted tool call
    When the agent submits a tool call request for tool "metrics.query"
    Then Conductor returns decision "block"
    And the reason is "no-matching-rule"
    And the block is logged to the audit trail

  Scenario: Concurrent tool calls from the same agent are each evaluated independently
    Given a registered agent "agent-eta" with scope "read:files, write:files"
    When the agent simultaneously submits two tool call requests:
      | Request | Tool        | Expected Decision |
      | R1      | file.read   | allow             |
      | R2      | file.write  | allow             |
    Then each request is evaluated independently against the policy
    And the policy state is not mutated during either evaluation
    And both decisions are returned without interference
```

---

## Feature: Data Classification

Conductor enforces data classification rules on all content flowing through tool calls. Classification levels: L1 (Internal), L2 (Sensitive), L3 (Restricted/Secret).

```gherkin
Feature: Data Classification

  Background:
    Given Conductor is running with data classification enforcement enabled
    And the classification policy defines L1, L2, and L3 content rules

  Scenario: Block L3 data egress via HTTP response
    Given a tool call response contains L3-classified content (e.g., API keys, credentials, secrets)
    When Conductor evaluates the outbound HTTP response
    Then Conductor blocks the response
    And the reason is "l3-egress-blocked"
    And the L3 content is not included in the HTTP response body

  Scenario: Block L3 data egress via message channel
    Given a tool call would write L3-classified content to a message channel (e.g., Telegram, Discord, Slack)
    When Conductor evaluates the outbound message channel write
    Then Conductor blocks the write
    And the reason is "l3-egress-blocked"
    And the L3 content is not sent to the message channel

  Scenario: Block L3 data egress via file write
    Given a tool call would write L3-classified content to a file on disk
    When Conductor evaluates the outbound file write
    Then Conductor blocks the write
    And the reason is "l3-egress-blocked"
    And no file is created or modified with the L3 content

  Scenario: Allow L1 data in internal tool call flow
    Given a tool call carries only L1-classified content (e.g., non-sensitive operational data)
    When Conductor evaluates the tool call
    Then Conductor does not block the request on classification grounds
    And the L1 content flows through to the tool execution

  Scenario: L2 data allowed internally, blocked externally, redacted in audit log
    Given a tool call carries L2-classified content (e.g., system paths, config values, log entries)
    When the tool call targets an internal destination (same host, localhost-bound service)
    Then Conductor allows the tool call
    And the audit log entry for this call has L2 content redacted (replaced with "[L2-REDACTED]")

    When the tool call targets an external destination (off-host HTTP endpoint, external message channel, remote file system)
    Then Conductor blocks the tool call
    And the reason is "l2-egress-blocked"
    And the audit log entry has L2 content redacted

  Scenario: False positive override — agent flags block for human review
    Given a tool call has been blocked by Conductor
    And the agent believes the block is a false positive
    When the agent submits an override request referencing the blocked request ID
    Then Conductor logs the override request with "override_requested=true"
    And Conductor does NOT automatically allow the blocked tool call
    And a human operator must review and explicitly approve the override before the tool call can proceed
```

---

## Feature: Agent Registration

Conductor maintains a registry of authorized agents. Only registered, non-expired, non-revoked agents may submit tool call requests.

```gherkin
Feature: Agent Registration

  Background:
    Given Conductor is running with the agent registry active
    And the registry TTL is set to 24h (configurable via registry.ttl)

  Scenario: Reject tool call from unregistered agent
    Given no agent with ID "agent-unknown" is registered in the registry
    When "agent-unknown" submits a tool call request
    Then Conductor returns decision "block"
    And the reason is "unregistered-agent"

  Scenario: Register an external OpenClaw bridge agent
    Given an external OpenClaw bridge agent "bridge-openclaw-01" wishes to register
    And the sponsoring agent "sponsor-admin" is in the approved sponsor list
    When "bridge-openclaw-01" submits a registration request sponsored by "sponsor-admin"
    Then Conductor adds "bridge-openclaw-01" to the registry
    And the registration is assigned a TTL of 24h (default)
    And "bridge-openclaw-01" may subsequently submit tool call requests

  Scenario: Reject registration with unapproved sponsor
    Given an external agent "bridge-rogue" wishes to register
    And the sponsoring agent "sponsor-unknown" is NOT in the approved sponsor list
    When "bridge-rogue" submits a registration request sponsored by "sponsor-unknown"
    Then Conductor rejects the registration
    And the reason is "unapproved-sponsor"

  Scenario: Duplicate registration is rejected
    Given an agent "agent-alpha" is already registered in the registry
    When "agent-alpha" submits a second registration request
    Then Conductor rejects the registration
    And the error code is "DuplicateAgent"
    And the existing registration is not modified

  Scenario: Agent deregistration
    Given a registered agent "agent-beta" has active in-flight requests R1 and R2
    When an admin submits a deregistration request for "agent-beta"
    Then Conductor marks "agent-beta" as deregistering
    And in-flight requests R1 and R2 are allowed to complete
    And "agent-beta" is removed from the registry once all in-flight requests complete
    And any subsequent requests from "agent-beta" are rejected with reason "unregistered-agent"

  Scenario: Expired registration is treated as unregistered
    Given an agent "agent-gamma" was registered with a TTL of 24h
    And 24h have elapsed since "agent-gamma" registered
    When "agent-gamma" submits a tool call request
    Then Conductor treats "agent-gamma" as unregistered
    And returns decision "block" with reason "registration-expired"

  Scenario: Admin revokes an agent immediately
    Given a registered agent "agent-compromised" is active
    When an admin issues a revocation command for "agent-compromised"
    Then Conductor immediately marks "agent-compromised" as revoked
    And all subsequent requests from "agent-compromised" are rejected
    And the reason is "agent-revoked"
    And in-flight requests are blocked at next policy checkpoint
```

---

## Feature: Audit Trail

Conductor maintains a tamper-evident, append-only audit log for all policy decisions and tool execution outcomes.

> `request_hash`: SHA-256 of the full request payload, excluding any content classified as L3 prior to hashing.

```gherkin
Feature: Audit Trail

  Background:
    Given Conductor is running with audit logging enabled and the audit log is available

  Scenario: Log allowed tool call with execution status
    Given a registered agent "agent-alpha" submits a tool call that is allowed by policy
    When the tool call executes and completes successfully
    Then Conductor writes an audit log entry AFTER execution completes
    And the audit entry includes: agent_id, tool_name, decision="allow", execution_status="success", timestamp, request_hash
    # request_hash = SHA-256(full request payload minus L3 content)

  Scenario: Log blocked tool call
    Given a registered agent "agent-beta" submits a tool call that is blocked by policy
    When Conductor evaluates and blocks the request
    Then Conductor writes an audit log entry with: agent_id, tool_name, decision="block", reason, timestamp, request_hash
    And the tool call does not execute

  Scenario: Audit log unavailable triggers fail-closed
    Given the audit log is unavailable (e.g., disk full, write failure, log service down)
    When any agent submits a tool call request
    Then Conductor blocks all tool calls
    And Conductor logs the error to stderr
    And Conductor alerts the operator (via configured alert channel)
    And no tool calls are allowed until audit logging is restored

  Scenario: Tool allowed but execution fails — separate audit event
    Given a registered agent "agent-gamma" submits a tool call that is allowed by policy
    When the tool call executes but the tool itself returns an error or exception
    Then Conductor writes a separate audit log entry for the execution failure
    And the audit entry includes: agent_id, tool_name, decision="allow", execution_status="execution_failed", error_detail, timestamp, request_hash

  Scenario: Audit records are tamper-evident
    Given audit log records have been written by Conductor
    When any audit record is modified after being written
    Then any modification to an audit record is detectable
    # Implementation note: Conductor uses an integrity mechanism (e.g., chained hashes, WORM storage,
    # or signed records) — the specific mechanism is an implementation detail, not a BDD concern.

  Scenario: Audit log entry includes request_hash for traceability
    Given a registered agent "agent-delta" submits a tool call request
    And the request payload contains a mix of L1, L2, and L3 content
    When Conductor evaluates the request (allow or block)
    Then the audit log entry includes a request_hash field
    And the request_hash is SHA-256 of the full request payload with all L3 content removed before hashing
```

---

## Feature: Policy Management

Conductor loads, validates, and manages the lifecycle of policy files. Invalid or missing policies result in a safe failure state.

```gherkin
Feature: Policy Management

  Scenario: Load policy on startup
    Given a valid policy file exists at the configured policy path
    When Conductor starts
    Then Conductor loads and activates the policy
    And begins accepting agent requests

  Scenario: Startup with invalid policy file
    Given a policy file exists at the configured policy path
    But the policy file contains a syntax or semantic error
    When Conductor starts
    Then Conductor refuses to start
    And logs a "PolicyLoadError" with the line number and reason for the error
    And no agent requests are accepted

  Scenario: Startup with no policy file
    Given no policy file exists at the configured policy path
    When Conductor starts
    Then Conductor refuses to start
    And logs a "MissingPolicyError"
    And no agent requests are accepted

  Scenario: Startup with empty policy (zero rules)
    Given a policy file exists at the configured policy path
    And the policy file is syntactically valid but contains zero rules
    When Conductor starts
    Then Conductor starts successfully in default-deny mode
    And logs a warning: "Policy contains zero rules; all requests will be denied"
    And all tool call requests are blocked until a non-empty policy is loaded

  Scenario: Hot-reload updated policy
    Given Conductor is running with a valid policy loaded
    And the policy file has been updated on disk
    When the configured reload_interval elapses (default: 30s)
    Then Conductor detects the policy file change
    And atomically reloads the updated policy
    And new policy takes effect for all subsequent requests
    And in-flight evaluations using the prior policy complete without interruption

  Scenario: Hot-reload rejects invalid updated policy
    Given Conductor is running with a valid policy loaded
    And the policy file on disk has been updated with an invalid configuration
    When the configured reload_interval elapses
    Then Conductor detects the policy file change
    And attempts to load the updated policy
    And rejects the updated policy due to the validation error
    And logs the validation error with line number and reason
    And continues running with the previously valid policy unchanged
```

---

## Feature: HTTP Transport

Conductor exposes a localhost-only HTTP REST API for agent communication. No external network binding in v1.

```gherkin
Feature: HTTP Transport

  Background:
    Given Conductor is running and listening on localhost only
    And the evaluation timeout is set to 500ms (configurable via conductor.eval_timeout_ms)

  Scenario: Agent submits a tool call request via HTTP POST
    Given a registered agent "agent-alpha"
    When the agent sends an HTTP POST to the Conductor endpoint with a valid tool call request body
    Then Conductor receives and evaluates the request
    And returns an HTTP 200 response with a JSON body containing the allow/block decision

  Scenario: Conductor returns decision within configured timeout
    Given a registered agent "agent-beta"
    When the agent sends a valid tool call request
    Then Conductor returns the allow/block decision within 500ms (the configured eval_timeout_ms)
    And the response body includes: decision, reason (if blocked), request_id, timestamp

  Scenario: Conductor returns 503 when fail-closed is triggered
    Given Conductor has entered fail-closed state (e.g., audit log unavailable, eval engine error)
    When any agent sends a tool call request
    Then Conductor returns HTTP 503
    And the response body includes error code "ServiceUnavailable" and reason
    And no tool calls are processed until fail-closed condition is resolved

  Scenario: Request missing required fields returns 400
    Given an agent sends an HTTP POST to the Conductor endpoint
    And the request body is missing one or more required fields (agent_id, tool_name, or request payload)
    Then Conductor returns HTTP 400
    And the response body includes error code "MalformedRequest"
    And the missing field names are included in the error detail
    And the request is not evaluated by the policy engine
```

---

_End of v1 BDD Specifications_
