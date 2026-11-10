# Architecture --- TRAIL-SOC

> **Status:** Proposed MVP architecture. The diagrams are logical
> deployment targets for the implementation plan, not currently running
> services.

## Reading the dependency rules

Arrows describe permitted calls, not merely convenient imports. In
particular, an agent may call the gateway client but may not import an
adapter, resolve an adapter URL, or load a tool credential. A modular
monolith is acceptable only if these boundaries remain explicit and are
covered by tests.

## 1. Architectural style

Use a **modular monolith with isolated execution boundaries** for the
MVP. The domain modules can live in one FastAPI deployment to reduce
complexity, while the policy gateway and tool runtime retain strict
interfaces and separate credentials/network access. Split into services
only when scaling or security isolation requires it.

## 2. Component responsibilities and allowed dependencies

``` text
React Analyst Console
        |
        v
FastAPI API / Auth
  |        |          |
  |        |          +--> Report & Evidence modules
  |        +-------------> Investigation / Approval modules
  +----------------------> Orchestrator (LangGraph)
                                  |
                         Agent runtime (untrusted)
                                  |
                           ToolRequest API
                                  v
                         POLICY GATEWAY
                     /       |        \
                   OPA    PostgreSQL   Audit
                            |
                     approved only
                            v
                       Tool Runtime
                   /        |         \
                 SIEM      TI       Endpoint
               (mock)    (mock)       (mock)
```

The diagram is logical. In deployment, network segmentation must ensure
that the agent runtime cannot connect directly to the tool runtime.

## 3. Agent model

Three agents are sufficient: - `log_analyst` - `threat_intel` -
`endpoint_analyst`

Agents should not receive general-purpose tools. They receive a small
client function that submits a typed `ToolRequest` to the gateway. The
client must not include adapter URLs, credentials or vendor SDKs.

### Agent output contract

Each agent returns: - `summary` - `claims[]` - `evidence_refs[]` -
`contradictory_evidence_refs[]` - `assumptions[]` - `confidence` (0--1,
agent estimate---not a calibrated probability) -
`recommended_next_steps[]` - `limitations[]`

The orchestrator must distinguish observations from hypotheses and
recommendations.

## 4. Work-order capability model

A work order is a capability envelope. It includes: - Work-order ID and
version. - Investigation ID. - Agent subject / service identity. -
Objective. - Allowed tool/action pairs. - Allowed target selectors
(e.g. asset IDs or tenant/environment labels). - Argument constraints
where relevant. - Maximum calls, elapsed time, and optional token/cost
budget. - Per-action rate limits. - Expiry and stop conditions. - Risk
classification. - Policy bundle/version.

The gateway retrieves the persisted work order; it must not trust a
work-order object supplied solely by the agent. A signed work-order
token can be added later, but database-backed lookup is adequate for a
single-service MVP.

## 5. Policy decision pipeline

Order matters. Apply cheap hard checks before OPA: 1. Authenticate
calling workload. 2. Resolve active work order and agent binding. 3.
Check work-order status and expiry. 4. Validate tool/action exists. 5.
Validate target is in scope. 6. Validate typed arguments and reject
unknown keys. 7. Canonicalise request and compute hash. 8. Check
idempotency and duplicate window. 9. Check call/time/rate budgets. 10.
Evaluate Rego policy. 11. Enforce `DENY`, `REQUIRE_APPROVAL`, or
`ALLOW`. 12. If allowed, persist execution intent before adapter call.
13. Execute with timeout and adapter-side idempotency. 14. Persist
result/evidence and audit event.

OPA should return structured decision metadata, but the gateway is
responsible for enforcement, persistence and execution.

## 6. Risk tiers

  -----------------------------------------------------------------------
  Tier                    Example                 Default treatment
  ----------------------- ----------------------- -----------------------
  R0 --- Read-only        Query logs, retrieve    Allow only if scoped
                          endpoint metadata       

  R1 --- Low-impact       Add an investigation    Allow if scoped
  simulation              note in mock            
                          environment             

  R2 --- Containment      Simulated endpoint      Human approval required
  simulation              isolation               

  R3 --- Destructive/high Disable protection,     Deny in MVP
  impact                  delete logs, wipe       
                          device                  
  -----------------------------------------------------------------------

Risk tiers are configured per action, not inferred solely from an
agent's natural-language reason.

## 7. Network and credential topology

-   Browser talks to API.
-   Agent runtime talks to API/gateway only.
-   Gateway talks to OPA, database, audit and tool runtime.
-   Tool runtime talks only to local fixtures in MVP.
-   No agent-to-tool route.
-   No integration secrets in agent environment variables.
-   Adapter credentials (when integrations exist) are held by the tool
    runtime or a dedicated secret broker, never returned to agents.
-   Restrict outbound network by default.

A network diagram in Docker Compose should make the allowed paths
explicit. Tests should attempt direct connection from agent container to
adapter and verify failure.

## 8. Audit architecture

Each event includes: - event ID, sequence number, timestamp UTC -
investigation/work-order/request IDs - event type and actor - canonical
payload digest - policy version and decision/reason - previous event
hash and current event hash - optional signature/checkpoint metadata

Hash formula for a canonical JSON event:
`event_hash = SHA256(previous_hash || canonical_json(event_without_hash_fields))`

Use a stable canonical JSON serialization (sorted keys, UTF-8, fixed
number/string representation). Store events in append-only application
semantics. Periodically sign the latest chain hash with a key
unavailable to the database writer. For a student MVP, implement hash
chaining and a verifier; describe signed checkpoints as a hardening step
if key management is not completed.

Hash chaining detects alteration when a trusted checkpoint/head is
retained. It does not by itself prevent a privileged attacker from
rewriting the whole chain.

## 9. Evidence architecture

Evidence is immutable after ingestion: - Metadata in PostgreSQL. - Raw
object in local/MinIO storage. - SHA-256 digest. - Source and collection
timestamp. - Parser/schema version. - Sensitivity label. - Investigation
linkage.

Findings refer to evidence IDs, not copied excerpts alone. Store
redacted excerpts for UI and preserve original fixture objects
separately.

## 10. Retrieval / RAG

RAG is optional and secondary to the governance contribution. Start with
deterministic IOC fixture lookup. Add Qdrant only after the end-to-end
gateway works: - Curate source documents and record
source/license/date. - Chunk and embed documents. - Retrieve top-k
passages with source IDs. - Treat retrieved text as untrusted data. - Do
not let retrieved content alter policy or grant permissions. - Record
retrieval query, document IDs and content hashes in evidence.

## 11. Suggested deployment profiles

### Local development

Docker Compose, mock tools, seeded data, local LLM or hosted model
configured via backend only.

### Demonstration

Same services with fixed scenario seed, deterministic mock adapter
outputs and a reproducible run script.

### Future pilot

Separate gateway and tool runtime, managed database, secrets manager,
signed audit checkpoints, authenticated analyst identity, read-only
vendor integrations and formal security review.
