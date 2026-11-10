# API Contract --- TRAIL-SOC

> **Status:** Proposed `/api/v1` contract. Examples are illustrative
> request/response fixtures and should become OpenAPI schemas and
> contract tests during implementation.

## API safety rule

The API is not an authority simply because a caller can reach it. Human
endpoints authenticate the analyst; gateway endpoints authenticate the
workload and resolve its work order server-side. The JSON body can
identify a requested resource, but it can never grant identity, scope,
approval or risk.

Base path: `/api/v1`\
Content type: `application/json`\
All timestamps: UTC RFC 3339.\
Authentication: analyst JWT/OIDC for human endpoints; workload
identity/service token for agent-to-gateway endpoints. Do not use a
shared hard-coded agent token in a deployed environment.

## 1. Conventions

-   IDs are opaque strings (UUID recommended).
-   Requests reject unknown fields where security-sensitive.
-   Pagination uses `limit` and `cursor`.
-   Errors use
    `{ "error": { "code": "...", "message": "...", "request_id": "..." } }`.
-   All mutating requests accept `Idempotency-Key` where appropriate.
-   API version changes are explicit; breaking changes require
    `/api/v2`.

## 2. Alert and investigation APIs

### `POST /alerts`

Create an alert.

Request:

``` json
{
  "source": "mock-siem",
  "external_id": "alert-001",
  "severity": "HIGH",
  "title": "Suspicious authentication activity",
  "description": "Multiple failed logins followed by a successful login",
  "observed_at": "2026-11-10T09:00:00Z",
  "asset_ids": ["device-123"],
  "evidence": [{"kind": "log", "content": {"event": "failed_login", "ip": "198.51.100.10"}}]
}
```

Response `201`:

``` json
{"id":"alert-uuid","status":"RECEIVED"}
```

### `POST /investigations`

Create investigation from alert.

Request:

``` json
{"alert_id":"alert-uuid","scenario":"suspicious_login"}
```

Response `202`:

``` json
{"id":"inv-uuid","status":"CREATED","status_url":"/api/v1/investigations/inv-uuid"}
```

### `GET /investigations`

List investigations. Query: `status`, `severity`, `limit`, `cursor`.

### `GET /investigations/{investigation_id}`

Return status, alert summary, agent states and timestamps.

### `POST /investigations/{id}/start`

Start orchestration. Returns `202`.

### `POST /investigations/{id}/stop`

Request a safe stop. Must prevent new tool calls; in-flight operations
are reconciled.

## 3. Work orders

### `GET /investigations/{id}/work-orders`

Return work orders and their current versions.

### `GET /work-orders/{work_order_id}`

Return objective, agent binding, allowed tool/action pairs, target
scope, budget, expiry and status. Do not expose secrets.

Work orders are created by the orchestrator, not arbitrary agents. A
future admin endpoint may create test work orders with privileged
authorization.

### Work-order response example

``` json
{
  "id": "wo-uuid",
  "version": 1,
  "agent_id": "endpoint_analyst",
  "objective": "Assess device-123 process activity",
  "allowed_actions": ["endpoint.get_device", "endpoint.get_process_tree",
                      "endpoint.simulate_isolate"],
  "allowed_targets": ["device-123"],
  "budgets": {"max_requests": 10, "max_executions": 4},
  "expires_at": "2026-11-10T10:00:00Z",
  "risk_ceiling": "R2",
  "status": "ACTIVE"
}
```

The UI may display this object, but the client cannot modify it by
resubmitting the response. A new capability requires a new work-order
version and a new audit event.

## 4. Agent gateway API

### `POST /gateway/tool-requests`

Submit a tool request. This endpoint is the only route to security
tools.

Headers:

``` http
Authorization: Bearer <workload-token>
Idempotency-Key: <unique-random-value>
```

Request:

``` json
{
  "work_order_id": "wo-uuid",
  "tool": "siem",
  "action": "query_events",
  "target": "tenant-demo",
  "arguments": {
    "start_time": "2026-11-10T08:00:00Z",
    "end_time": "2026-11-10T09:00:00Z",
    "filters": {"asset_id": "device-123"},
    "limit": 100
  },
  "reason": "Correlate authentication events for the alert",
  "evidence_refs": ["ev-alert-001"]
}
```

Allowed outcomes: - `200` --- request previously completed / idempotent
replay. - `202` --- approved request accepted for asynchronous execution
or approval pending. - `403` --- policy denied or scope violation. -
`409` --- idempotency conflict. - `422` --- schema/argument validation
failed. - `429` --- rate/budget limit. - `503` --- policy/audit
dependency unavailable; no execution.

Example denial:

``` json
{
  "request_id": "req-uuid",
  "status": "DENIED",
  "decision": "DENY",
  "reason_codes": ["TARGET_OUT_OF_SCOPE"],
  "executed": false
}
```

Example approval pending:

``` json
{
  "request_id": "req-uuid",
  "status": "PENDING_APPROVAL",
  "decision": "REQUIRE_APPROVAL",
  "approval_id": "approval-uuid",
  "executed": false
}
```

Example success:

``` json
{
  "request_id": "req-uuid",
  "status": "SUCCEEDED",
  "decision": "ALLOW",
  "result": {"events": []},
  "evidence_refs": ["ev-002"],
  "executed": true
}
```

### `GET /gateway/tool-requests/{request_id}`

Get request lifecycle and decision. Agent view is restricted to its own
work order.

## 5. Human approval APIs

### `GET /approvals`

Analyst queue. Supports `status`, `investigation_id`, pagination.

### `POST /approvals/{approval_id}/decision`

Request:

``` json
{
  "decision": "APPROVE",
  "comment": "Approved for this simulated incident only"
}
```

or `"REJECT"`. The server binds decision to the immutable request hash
and authenticated approver. The browser must not supply the hash as an
authority.

Response:

``` json
{"approval_id":"approval-uuid","status":"APPROVED","request_id":"req-uuid"}
```

### `POST /approvals/{approval_id}/execute`

Optional explicit execute step. It revalidates work order, request hash,
policy and approval. It must not directly invoke an adapter.

## 6. Evidence, reports and audit

### `GET /investigations/{id}/evidence`

List evidence metadata and redacted preview.

### `GET /evidence/{evidence_id}`

Fetch metadata. Raw object retrieval uses a short-lived authorized URL
or API stream.

### `GET /investigations/{id}/findings`

List structured findings and evidence links.

### `GET /investigations/{id}/report`

Return final report JSON or Markdown/PDF export.

### `GET /investigations/{id}/audit-events`

Paginated audit events; redact secrets and sensitive raw arguments.

### `POST /audit/verify`

Request:

``` json
{"investigation_id":"inv-uuid"}
```

Response:

``` json
{"valid":true,"events_checked":42,"head_hash":"...","first_invalid_sequence":null}
```

`valid: false` is an operational security signal, not a request to
repair the ledger. The verifier should identify the first invalid
sequence and the investigation should be marked as requiring review.

## 7. Operational endpoints

-   `GET /healthz` --- process liveness only.
-   `GET /readyz` --- dependencies required for safe operation (DB, OPA,
    audit writer).
-   `GET /metrics` --- optional Prometheus metrics; restrict exposure.

## 8. API security requirements

-   Enforce authentication and authorization server-side on every
    endpoint.
-   Separate human and workload principals.
-   Bind agent identity to server-issued workload identity; never trust
    `agent_id` from JSON alone.
-   Rate-limit all endpoints.
-   Apply request body and query size limits.
-   Validate target IDs and argument schemas.
-   Do not accept arbitrary shell, SQL, URL fetch or code execution
    inputs.
-   Do not return tool credentials or unrestricted raw evidence.
-   Use TLS outside local development.
