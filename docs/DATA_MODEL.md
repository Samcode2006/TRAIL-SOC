# Data Model --- TRAIL-SOC

> **Status:** Proposed PostgreSQL model. Table definitions are logical
> contracts and still require migrations, indexes and repository tests.

## Data ownership rule

PostgreSQL owns transactional state and relationships. Evidence objects
may be stored in a filesystem or object store, but their metadata,
content digest, sensitivity and investigation linkage remain in
PostgreSQL. Agents receive references and redacted representations, not
unrestricted storage access.

PostgreSQL is the source of truth for transactional state. Evidence
blobs may live in filesystem/MinIO; PostgreSQL stores metadata and
digests.

## 1. Entity relationships

``` text
Alert 1 --- * Investigation
Investigation 1 --- * WorkOrder
Investigation 1 --- * AgentRun
WorkOrder 1 --- * ToolRequest
ToolRequest 1 --- 0..1 Approval
ToolRequest 1 --- * PolicyDecision
ToolRequest 1 --- 0..* ToolExecution
Investigation 1 --- * Evidence
Investigation 1 --- * Finding
Investigation 1 --- * AuditEvent
Finding * --- * Evidence (via FindingEvidence)
```

## 2. Tables

### `alerts`

-   `id` UUID PK
-   `source` varchar
-   `external_id` varchar
-   `severity` enum
-   `title`, `description`
-   `observed_at` timestamptz
-   `raw_payload_uri` nullable
-   `created_at`
-   Unique `(source, external_id)`

### `investigations`

-   `id` UUID PK
-   `alert_id` FK
-   `status`
-   `scenario`
-   `started_at`, `completed_at`
-   `created_by`
-   `created_at`, `updated_at`

### `work_orders`

-   `id` UUID PK
-   `investigation_id` FK
-   `agent_id`
-   `version` integer
-   `objective` text
-   `allowed_actions` JSONB
-   `allowed_targets` JSONB
-   `budgets` JSONB
-   `stop_conditions` JSONB
-   `status`
-   `expires_at`
-   `policy_version`
-   `created_at`
-   Unique `(investigation_id, agent_id, version)`

### `tool_requests`

-   `id` UUID PK
-   `work_order_id` FK
-   `investigation_id` FK
-   `agent_principal`
-   `tool`, `action`, `target`
-   `arguments_redacted` JSONB
-   `canonical_request_hash`
-   `idempotency_key`
-   `status`
-   `created_at`, `updated_at`
-   Unique `(work_order_id, idempotency_key)`

### `policy_decisions`

-   `id` UUID PK
-   `tool_request_id` FK
-   `decision`
-   `reason_codes` JSONB
-   `policy_id`, `policy_version`
-   `input_digest`
-   `evaluated_at`

### `approvals`

-   `id` UUID PK
-   `tool_request_id` FK UNIQUE
-   `request_hash`
-   `status`
-   `approver_id`
-   `comment`
-   `expires_at`
-   `decided_at`
-   `consumed_at`

### `tool_executions`

-   `id` UUID PK
-   `tool_request_id` FK
-   `adapter_name`
-   `status`
-   `started_at`, `finished_at`
-   `result_digest`
-   `error_code`
-   `retry_count`

### `evidence`

-   `id` UUID PK
-   `investigation_id` FK
-   `source_type`, `source_ref`
-   `object_uri`
-   `content_sha256`
-   `collected_at`
-   `sensitivity`
-   `metadata` JSONB

### `findings`

-   `id` UUID PK
-   `investigation_id` FK
-   `agent_id`
-   `title`, `claim`
-   `confidence` numeric (0..1)
-   `assumptions` JSONB
-   `limitations` JSONB
-   `created_at`

### `finding_evidence`

-   `finding_id` FK
-   `evidence_id` FK
-   `relationship` enum: SUPPORTS / CONTRADICTS / CONTEXT
-   Composite PK `(finding_id, evidence_id, relationship)`

### `audit_events`

-   `sequence` bigserial
-   `id` UUID UNIQUE
-   `investigation_id` FK nullable
-   `actor_type`, `actor_id`
-   `event_type`
-   `payload` JSONB
-   `payload_digest`
-   `previous_hash`
-   `event_hash`
-   `created_at`
-   Append-only application permissions; no update/delete endpoint.

### `agent_runs`

-   `id` UUID PK
-   `investigation_id` FK
-   `work_order_id` FK
-   `agent_id`
-   `status`
-   `started_at`, `finished_at`
-   `steps_used`
-   `error_code`

## 3. Constraints and indexes

-   Foreign keys on all relationships.
-   Unique alert source/external ID.
-   Unique work-order version.
-   Unique idempotency key per work order.
-   Unique approval per tool request.
-   Check `confidence BETWEEN 0 AND 1`.
-   Index investigation status/created time.
-   Index tool request status/work order.
-   Index audit events `(investigation_id, sequence)`.
-   Index evidence investigation/source.
-   JSONB fields are validated by application schemas; do not rely on
    JSONB alone for security validation.

## 4. Retention and deletion

For the prototype, retain all scenario data until manually reset. In a
future pilot, define retention by evidence sensitivity, legal
requirements and incident policy. Audit records should not be casually
deleted; support controlled archival and documented retention workflows.
Do not claim immutable storage unless the storage system actually
enforces it.
