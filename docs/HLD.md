# High-Level Design (HLD) --- TRAIL-SOC

> **Status:** Proposed MVP design. This document explains what the
> system must do and where trust boundaries exist; it is not a deployed
> architecture diagram.

## Design objective

The design turns the presentation's “let AI investigate, but never let
it self-authorize” principle into an enforceable runtime sequence. The
gateway is the security boundary, not the prompt, agent framework or
UI. Every later design choice should preserve that invariant.

## 1. System overview

TRAIL-SOC is a governed multi-agent SOC investigation framework. It
separates **analysis** from **authority**:

-   LLM agents interpret evidence and propose next steps.
-   A deterministic gateway decides whether a requested tool operation
    is permitted.
-   Narrow adapters execute only gateway-approved operations.
-   An audit service records the complete lifecycle.
-   Evidence and reports preserve provenance.

The central invariant is:

> No agent can directly invoke a security tool. Every operation must be
> authenticated, scoped, validated, authorized and audited by the
> TRAIL-SOC gateway.

## 2. Context and boundaries

### In scope

-   Synthetic alert ingestion.
-   Work-order generation and agent orchestration.
-   Log, threat-intelligence and endpoint investigation.
-   Policy evaluation and enforcement.
-   Human approval for high-impact simulated actions.
-   Simulated tool execution.
-   Audit ledger and evidence/report generation.
-   Analyst console.

### Out of scope for MVP

-   Production endpoint containment.
-   Production SIEM write operations.
-   Autonomous destructive remediation.
-   Full enterprise IAM / multi-tenant controls.
-   Vendor-specific integration certification.

## 3. Logical components

  -----------------------------------------------------------------------
  Component                           Responsibility
  ----------------------------------- -----------------------------------
  Web console                         Analyst workflow, approvals,
                                      evidence and reports

  API service                         Authentication, request validation,
                                      resource APIs

  Alert/investigation service         Alert intake and investigation
                                      lifecycle

  Orchestrator                        Work orders, agent scheduling and
                                      state transitions

  Agent runtime                       Specialist reasoning and structured
                                      findings

  Policy gateway                      Sole tool-call ingress; identity,
                                      scope, policy, budget, idempotency

  OPA policy service                  Deterministic policy evaluation

  Tool runtime                        Narrow simulated adapters

  Audit service                       Append-only events, hash chain,
                                      verification

  Evidence service                    Evidence metadata and object
                                      storage

  Report service                      Consolidated report generation

  PostgreSQL                          Transactional state, work orders,
                                      decisions and ledger

  Object storage                      Raw evidence/report artifacts

  Qdrant (later)                      Optional semantic retrieval over
                                      curated threat-intelligence
                                      documents
  -----------------------------------------------------------------------

## 4. Deployment view

MVP deployment uses Docker Compose: - `web` - `api` (can host
orchestrator and domain modules initially) - `opa` - `postgres` -
`tool-runtime` - optional `redis` - optional `minio`

Keep the agent runtime and tool runtime on separate Docker networks. The
agent network can reach only the API/gateway endpoint. The tool network
is reachable only by the gateway/runtime. Do not expose adapter ports to
the agent network or host unless needed for development debugging.

A modular monolith is acceptable: the API can initially contain
investigation, approval, evidence and reporting modules. The policy
gateway must still be a clearly isolated module with a separate
interface and no bypass path.

## 5. Primary data flow

1.  Analyst submits an alert.
2.  API validates and persists it.
3.  Orchestrator creates an investigation and bounded work orders.
4.  Agents receive only their work-order context and assigned evidence.
5.  Agent submits a structured `ToolRequest` to the gateway.
6.  Gateway authenticates service identity and loads the active work
    order.
7.  Gateway validates tool/action/target/arguments and computes a
    canonical request hash.
8.  Gateway checks idempotency, budget and rate limits.
9.  Gateway sends policy input to OPA.
10. OPA returns a decision and reason codes. Gateway enforces the
    decision.
11. If denied: persist decision and return; no adapter call.
12. If approval required: persist pending approval and return; no
    adapter call.
13. If allowed: gateway records intent, invokes the narrow adapter,
    records result and evidence.
14. Agents use returned evidence to form structured findings.
15. Report service compiles claims and provenance.

### 5.1 Request outcome state machine

```text
RECEIVED
   |-- validation/scope/policy failure --> DENIED
   |-- approval required ---------------> PENDING_APPROVAL
   |                                      |-- reject/expire --> DENIED
   |                                      '-- approve -------> APPROVED
   '-- allow ----------------------------> EXECUTING
                                              |-- success --> SUCCEEDED
                                              |-- failure --> FAILED
                                              '-- uncertain -> TIMED_OUT/UNKNOWN
```

Every transition emits an audit event. `APPROVED` is not an execution
result: the gateway must re-check the request hash, work-order status,
expiry, budget and current policy before entering `EXECUTING`.

## 6. Trust boundaries

-   **External/untrusted input:** alerts, log messages, threat reports,
    endpoint strings, model output.
-   **Agent boundary:** model context and agent process are treated as
    untrusted; prompt injection is expected.
-   **Gateway boundary:** trusted enforcement component; validates all
    claims rather than trusting agent assertions.
-   **Policy boundary:** policy bundle is versioned and reviewed; OPA
    availability failure means deny.
-   **Tool boundary:** adapters own credentials and expose narrow typed
    operations only.
-   **Audit boundary:** ledger writer is restricted; verification
    detects tampering but hash chaining alone is not a complete WORM
    guarantee.
-   **Human boundary:** approval identity and exact request hash must be
    authenticated and recorded.

## 7. Failure behaviour

  -----------------------------------------------------------------------
  Failure                             Required behaviour
  ----------------------------------- -----------------------------------
  OPA unavailable                     Fail closed; no tool execution

  PostgreSQL unavailable              No execution; cannot guarantee
                                      audit

  Audit append fails                  Do not execute action

  Adapter timeout                     Record timeout/unknown outcome; do
                                      not blindly retry non-idempotent
                                      action

  LLM unavailable                     Mark agent step failed or retry
                                      boundedly; no policy bypass

  Approval expired                    Reject and require a new
                                      request/approval

  Budget exhausted                    Stop work order and record reason

  Hash verification failure           Flag ledger compromised; preserve
                                      data and halt trusted reporting
  -----------------------------------------------------------------------

## 8. Scalability path

Scale only after MVP correctness: 1. Move long-running agent workflows
to a worker process. 2. Add Redis for transient coordination/rate
counters if needed. 3. Add durable queue or workflow engine if restart
recovery requires it. 4. Separate gateway and audit services. 5. Add
tenant isolation and external IAM. 6. Add real read-only SIEM/TI
integrations behind adapters. 7. Consider Kubernetes only when
operational requirements justify it.

## 9. Key design principles

-   Least privilege and capability-based work orders.
-   Deterministic authorization.
-   Fail-closed execution.
-   No credentials in prompts or agent environment.
-   Structured schemas instead of arbitrary command execution.
-   Evidence provenance and explicit uncertainty.
-   Human approval for high-impact actions.
-   Reproducible tests and policy versioning.
