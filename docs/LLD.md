# Low-Level Design (LLD) --- TRAIL-SOC

> **Status:** Implementation-ready design baseline. Code examples are
> contracts and algorithms to implement, not executable code already
> present in this repository.

## Implementation priorities

Implement the gateway and mock adapters before adding live LLM calls.
The security properties must be testable with deterministic fixtures,
so agent quality or provider availability cannot hide an authorization
regression.

## 1. Module layout

``` text
apps/api/app/
  main.py
  core/                 # settings, auth, errors, logging
  alerts/
  investigations/
  workorders/
  agents/
  gateway/
    service.py
    schemas.py
    canonicalize.py
    idempotency.py
    budgets.py
    policy_client.py
    approvals.py
  tools/
    registry.py
    adapters/
      siem_mock.py
      threat_intel_mock.py
      endpoint_mock.py
  audit/
    writer.py
    verifier.py
  evidence/
  reports/
  db/
    models.py
    session.py
    migrations/
policies/
  main.rego
  main_test.rego
```

Keep module interfaces explicit. The gateway must call adapters through
a registry; agents must not import adapter modules.

## 2. Core enums

``` python
class InvestigationStatus(str, Enum):
    CREATED = "CREATED"
    RUNNING = "RUNNING"
    WAITING_APPROVAL = "WAITING_APPROVAL"
    COMPLETED = "COMPLETED"
    FAILED = "FAILED"
    STOPPED = "STOPPED"

class WorkOrderStatus(str, Enum):
    ACTIVE = "ACTIVE"
    PAUSED = "PAUSED"
    EXHAUSTED = "EXHAUSTED"
    EXPIRED = "EXPIRED"
    COMPLETED = "COMPLETED"
    REVOKED = "REVOKED"

class Decision(str, Enum):
    ALLOW = "ALLOW"
    DENY = "DENY"
    REQUIRE_APPROVAL = "REQUIRE_APPROVAL"

class RequestStatus(str, Enum):
    RECEIVED = "RECEIVED"
    DENIED = "DENIED"
    PENDING_APPROVAL = "PENDING_APPROVAL"
    APPROVED = "APPROVED"
    EXECUTING = "EXECUTING"
    SUCCEEDED = "SUCCEEDED"
    FAILED = "FAILED"
    TIMED_OUT = "TIMED_OUT"
    DUPLICATE = "DUPLICATE"
```

## 3. Tool registry

Use an explicit registry, not dynamic import based on agent-provided
strings.

  Tool             Action               Risk   Arguments
  ---------------- -------------------- ------ ----------------------------
  `siem`           `query_events`       R0     time range, filters, limit
  `siem`           `get_alert`          R0     alert_id
  `threat_intel`   `lookup_indicator`   R0     indicator, indicator_type
  `endpoint`       `get_device`         R0     device_id
  `endpoint`       `get_process_tree`   R0     device_id, process_id
  `endpoint`       `simulate_isolate`   R2     device_id, reason

The MVP registry must not include real `disable_protection`,
`delete_logs`, `wipe`, shell, arbitrary HTTP, or arbitrary SQL actions.

Each adapter implements:

``` python
class ToolAdapter(Protocol):
    async def execute(
        self, action: str, arguments: dict, idempotency_key: str
    ) -> ToolResult: ...
```

Adapters validate again at their boundary and apply timeouts. Gateway
validation is not a reason to omit adapter validation.

## 4. Gateway service algorithm

``` python
async def submit_tool_request(req, caller):
    identity = authenticate_workload(caller)
    work_order = load_active_work_order(req.work_order_id)

    assert_agent_bound(identity, work_order)
    assert_work_order_active_and_not_expired(work_order)
    spec = registry.get(req.tool, req.action)
    validate_target_scope(req.target, work_order)
    validated_args = spec.argument_model.model_validate(req.arguments)
    canonical = canonicalize(req, validated_args)
    request_hash = sha256(canonical)

    prior = find_idempotency(req.idempotency_key)
    if prior:
        return same_request_result_or_conflict(prior, request_hash)

    reserve_budget_atomically(work_order, spec)
    policy_input = build_policy_input(identity, work_order, spec, req)
    decision = await opa.evaluate(policy_input)

    persist_decision_and_audit(decision, request_hash)
    if decision.kind == "DENY":
        return denied_response()
    if decision.kind == "REQUIRE_APPROVAL":
        create_pending_approval_bound_to_hash(...)
        return pending_approval_response()

    persist_execution_intent()
    result = await adapter.execute(...)
    persist_result_evidence_and_audit(result)
    return result
```

Production implementation must use database transactions/locking to
prevent concurrent requests from exceeding budgets or executing the same
idempotency key. Do not use Python in-memory dictionaries as the source
of truth.

If a timeout occurs after an adapter may have performed a side effect,
mark the outcome `UNKNOWN`/`TIMED_OUT` and reconcile before retrying. Do
not assume timeout means no execution.

## 5. Policy input and output

Example policy input:

``` json
{
  "subject": {"agent_id": "endpoint_analyst", "role": "endpoint_analyst"},
  "work_order": {
    "id": "wo-123",
    "status": "ACTIVE",
    "allowed_actions": ["endpoint.get_device", "endpoint.simulate_isolate"],
    "allowed_targets": ["device-123"],
    "max_calls": 10,
    "calls_used": 2
  },
  "request": {
    "tool": "endpoint",
    "action": "simulate_isolate",
    "target": "device-123",
    "risk": "R2",
    "request_hash": "..."
  },
  "approval": {"valid": false}
}
```

Example output:

``` json
{
  "decision": "REQUIRE_APPROVAL",
  "reason_codes": ["RISK_TIER_REQUIRES_HUMAN_APPROVAL"],
  "policy_id": "endpoint_isolation_approval",
  "policy_version": "v1"
}
```

Policy engine must never accept an LLM-generated `risk` value as
authoritative. Risk comes from the trusted tool registry.

## 6. Approval design

An approval record binds: - request ID - canonical request hash -
approver identity - decision - timestamp - expiry - optional comment

Approval endpoint re-fetches the request and verifies its hash has not
changed, work order is still active, approval has not expired, approver
is authorized and action remains permitted. Approval is single-use. The
approved operation then re-enters gateway enforcement; it does not call
the adapter directly from the UI.

## 7. Idempotency

-   Client sends a cryptographically random idempotency key per logical
    operation.
-   Database unique constraint on `(work_order_id, idempotency_key)`.
-   Store canonical request hash and resulting status.
-   Same key + same hash returns existing request/result.
-   Same key + different hash returns `409 IDEMPOTENCY_CONFLICT`.
-   For semantic duplicate detection, use a key derived from
    investigation + tool + action + normalized target + short time
    bucket for configured actions.
-   A semantic duplicate should be denied or return prior result; never
    silently execute again.

## 8. Budget accounting

Store counters in database: - tool_calls_used - tool_calls_max -
started_at / expires_at - optional token_budget / token_usage -
action-specific counts

Reserve a call atomically before policy/adapter execution. Define
whether denied requests consume a call budget; recommended: count all
submitted requests against a separate request-attempt budget, while only
approved adapter calls consume execution budget. This prevents denial
spam without punishing harmless policy checks.

## 9. Evidence and findings schemas

Evidence:

``` json
{
  "id": "ev-001",
  "investigation_id": "inv-001",
  "source_type": "SIEM",
  "source_ref": "alert-001",
  "collected_at": "2026-11-10T09:00:00Z",
  "content_sha256": "...",
  "object_uri": "evidence://inv-001/ev-001",
  "sensitivity": "INTERNAL"
}
```

Finding:

``` json
{
  "id": "finding-001",
  "title": "Repeated failed logins followed by suspicious process",
  "claim": "The account may have been compromised",
  "evidence_for": ["ev-001", "ev-004"],
  "evidence_against": ["ev-007"],
  "assumptions": ["Endpoint clock is synchronized"],
  "confidence": 0.72,
  "limitations": ["No identity-provider sign-in risk data available"]
}
```

Confidence is a model/agent estimate. UI must label it as such and must
not present it as statistically calibrated unless calibration testing is
performed.

## 10. Audit event hashing

Canonicalization: - UTF-8 JSON. - Sorted object keys. - No
NaN/Infinity. - UTC timestamps in RFC 3339. - Stable representation of
decimals. - Exclude `event_hash` and signature fields from hashed body.

`event_hash = SHA256(previous_hash_bytes || canonical_event_bytes)`

The first event uses a documented genesis hash. Sequence number and
previous hash are checked by verifier. Add signed periodic checkpoints
in hardening phase.

## 11. Error handling

Use stable error codes: - `AUTHENTICATION_FAILED` -
`WORK_ORDER_NOT_FOUND` - `WORK_ORDER_INACTIVE` - `AGENT_NOT_BOUND` -
`ACTION_NOT_ALLOWED` - `TARGET_OUT_OF_SCOPE` -
`ARGUMENT_VALIDATION_FAILED` - `POLICY_ENGINE_UNAVAILABLE` -
`BUDGET_EXHAUSTED` - `RATE_LIMITED` - `IDEMPOTENCY_CONFLICT` -
`APPROVAL_REQUIRED` - `APPROVAL_EXPIRED` - `APPROVAL_HASH_MISMATCH` -
`TOOL_TIMEOUT` - `AUDIT_WRITE_FAILED`

Never expose secrets, stack traces, raw policy internals or sensitive
evidence in client-facing errors.
