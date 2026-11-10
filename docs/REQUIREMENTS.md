# Requirements --- TRAIL-SOC

> **Status:** Proposed MVP requirements. These requirements describe
> the system to build; they are not evidence that an implementation
> exists. The five-slide source presentation is
> [Cyberforge.pdf](Cyberforge.pdf).

## How to use this document

This is the normative product baseline. “Shall” statements are required
for MVP acceptance, while “should” statements are recommended
implementation guidance. A requirement is complete only when it has an
implementation reference, a test, and an observable result in the final
demo. The acceptance criteria in section 7 are the minimum release gate.

## 1. Purpose

TRAIL-SOC addresses the risks introduced when AI agents investigate
security incidents and interact with security tools. Agents may consume
attacker-controlled logs, reports and tool output; they may be
manipulated by prompt injection, request actions outside their intended
scope, retry operations, or produce conclusions that are difficult to
audit.

The system provides bounded agent work orders, deterministic runtime
authorization, isolated tool execution, evidence-linked findings and an
auditable investigation ledger.

## 2. Goals

-   Enable multiple specialised agents to investigate a security alert.
-   Ensure every tool request passes through one mandatory policy
    gateway.
-   Keep authorization deterministic, testable and independent of LLM
    output.
-   Enforce agent identity, work-order scope, allowed tools/actions,
    target scope, budgets, timeouts and stopping conditions.
-   Require human approval for configured high-impact actions.
-   Prevent duplicate side effects through idempotency controls.
-   Preserve provenance from input evidence to final report.
-   Provide an analyst-facing interface for investigation status,
    evidence, decisions and approvals.
-   Evaluate safety and usefulness using reproducible benign and
    adversarial scenarios.

## 3. Non-goals for the MVP

-   Production SOC replacement or autonomous incident response.
-   Direct execution against customer endpoints or production cloud
    accounts.
-   Full SIEM/EDR vendor integration.
-   Autonomous deletion, disabling of security controls, credential
    changes or destructive remediation.
-   General-purpose agent marketplace.
-   Multi-tenant SaaS hardening.
-   Claiming that LLM reasoning is reliable authorization.
-   Guaranteed prevention of every prompt injection. The design reduces
    impact through capability restriction and enforcement.

## 4. Actors

  -----------------------------------------------------------------------
  Actor                               Responsibilities
  ----------------------------------- -----------------------------------
  SOC analyst                         Starts/reviews investigations,
                                      reviews evidence, approves or
                                      rejects gated actions

  Administrator                       Configures policies, tool adapters,
                                      users and system limits

  Orchestrator                        Creates work orders, schedules
                                      agents and tracks workflow

  Investigation agent                 Analyses assigned evidence and
                                      submits structured requests

  Policy gateway                      Authenticates requests and
                                      makes/enforces authorization
                                      decisions

  Tool adapter                        Executes a narrow, typed operation
                                      against a simulated or approved
                                      tool

  Auditor                             Verifies ledger integrity and
                                      reviews investigation provenance
  -----------------------------------------------------------------------

## 5. Functional requirements

### FR-01 --- Alert intake

The system shall accept a synthetic security alert containing an ID,
source, timestamp, severity, summary and evidence references. It shall
validate schema and size limits.

### FR-02 --- Investigation creation

An analyst shall be able to create an investigation from an alert. The
system shall assign a unique investigation ID and initial status.

### FR-03 --- Bounded work orders

The orchestrator shall create a work order for each agent with
objective, permitted tools/actions, resource budget, deadline, allowed
target scope, maximum steps and stop conditions. Work orders shall be
persisted and immutable after activation; amendments create a new
version.

### FR-04 --- Specialist agents

The MVP shall provide three logical roles: - Log Analyst: query and
analyse SIEM/security logs. - Threat Intel Agent: look up and enrich
indicators. - Endpoint Analyst: retrieve endpoint/process details and
propose response actions.

Agents may be implemented as LangGraph nodes/subgraphs, not necessarily
separate deployed processes.

### FR-05 --- Structured tool requests

An agent shall submit a typed request containing work-order ID, agent
identity, tool, action, target, arguments, reason, evidence references
and idempotency key. Free-form model text shall never be executed as a
tool command.

### FR-06 --- Mandatory gateway

Every tool request shall pass through the gateway. Agents shall not
possess adapter credentials or network access to tool services. The
gateway shall deny requests if identity or work-order context cannot be
verified.

### FR-07 --- Deterministic policy decision

The gateway shall evaluate identity, work-order scope, tool/action
allow-list, target scope, argument schema, risk tier, approval state,
time/resource budget and rate/idempotency limits. OPA/Rego may evaluate
policy; the gateway remains the enforcement point.

### FR-08 --- Argument validation

The gateway shall validate arguments against per-action Pydantic/JSON
schemas, reject unknown fields, enforce length/range limits and
canonicalise targets. Shell strings and arbitrary URLs shall not be
accepted as generic execution arguments.

### FR-09 --- Human approval

Actions classified as approval-required shall enter `PENDING_APPROVAL`.
The system shall bind approval to the exact request hash, target,
action, work order and expiry. Any mutation invalidates the approval.

### FR-10 --- Execution isolation

Only approved requests shall be forwarded to a narrow tool adapter. MVP
adapters shall simulate SIEM queries, threat-intelligence lookups and
endpoint read/simulated-control actions. Adapter processes shall run
with least privilege and bounded network access.

### FR-11 --- Budgets and stopping

The system shall enforce per-work-order maximum tool calls, elapsed
time, token/compute budget where measurable, and action-specific rate
limits. Exhaustion shall stop further tool calls and record the reason.

### FR-12 --- Idempotency and duplicate prevention

The gateway shall enforce a unique idempotency key per work order and
operation. A repeated key with the same canonical request shall return
the prior outcome; a repeated key with different content shall be
rejected as a conflict. High-risk operations shall also use a semantic
duplicate window.

### FR-13 --- Audit ledger

The system shall record request receipt, authentication result, policy
input/version, decision, approval, execution start/result, budget
change, evidence references and workflow stop reason. Denied requests
must be logged. Events shall be append-only at the application layer and
hash-linked.

### FR-14 --- Evidence bundle

Agents shall return structured claims, evidence references, assumptions,
confidence and limitations. Evidence objects shall include source,
collection time, content hash and provenance metadata.

### FR-15 --- Investigation report

The system shall generate a report containing incident summary,
timeline, findings, supporting/contradictory evidence, confidence,
actions proposed/executed/denied, approvals, limitations and recommended
next steps.

### FR-16 --- Analyst UI

The UI shall display investigations, agent status, work orders,
evidence, tool requests, policy decisions, pending approvals, audit
timeline and final reports.

### FR-17 --- Policy administration

An administrator shall be able to view active policy version and inspect
policy decisions. Editing policies in the MVP may be file/configuration
based; a self-service policy editor is not required.

### FR-18 --- Ledger verification

The system shall provide a verifier that recomputes the hash chain and
reports the first invalid/missing link. It shall not silently repair
evidence.

## 6. Non-functional requirements

  -----------------------------------------------------------------------
  ID                                  Requirement
  ----------------------------------- -----------------------------------
  NFR-01                              Authorization decisions must be
                                      deterministic for identical policy
                                      inputs and policy version.

  NFR-02                              A denied request must never reach
                                      the tool adapter.

  NFR-03                              Agent processes must not have
                                      direct network paths or credentials
                                      to tool adapters.

  NFR-04                              All timestamps are stored in UTC;
                                      UI may render local time.

  NFR-05                              APIs use versioned JSON contracts
                                      and structured error responses.

  NFR-06                              Secrets are provided through
                                      environment/secret manager, never
                                      committed.

  NFR-07                              Logs must redact credentials,
                                      tokens and sensitive payload
                                      fields.

  NFR-08                              Core workflow should run locally
                                      through Docker Compose.

  NFR-09                              Tests must cover allow, deny,
                                      approval, duplicate, budget and
                                      tamper cases.

  NFR-10                              The system should expose
                                      health/readiness endpoints and
                                      structured operational logs.

  NFR-11                              The MVP should be reproducible from
                                      documented seed data and one
                                      scenario command.

  NFR-12                              Failure of OPA, audit persistence
                                      or identity verification shall fail
                                      closed for tool execution.
  -----------------------------------------------------------------------

## 6.1 Scenario fixture contract

The seeded demonstration must be deterministic and self-contained. It
shall include:

| Fixture | Required content | Used to prove |
|---|---|---|
| `alert-001` | High-severity suspicious-login alert, timestamps, account, asset and source address | Alert intake and investigation creation |
| `auth-events-001` | Failed-login burst followed by a success, including one injected instruction in a log message | Correlation and prompt-injection resistance |
| `indicator-001` | Synthetic address/domain reputation and provenance | Threat-intelligence enrichment |
| `device-001` | Device metadata and process tree containing a suspicious process | Endpoint correlation |
| `isolation-request-001` | Request for `endpoint.simulate_isolate` on `device-001` | Approval binding and controlled execution |

The fixture must define expected evidence IDs and expected policy
outcomes. A test must fail if a denied or pending request increments the
adapter side-effect counter.

## 7. MVP acceptance criteria

1.  A seeded suspicious-login scenario creates an investigation and
    three bounded work orders.
2.  Agents retrieve evidence using simulated adapters and generate
    structured findings.
3.  A prompt-injection string embedded in a log is treated as data and
    cannot cause an unauthorized tool invocation.
4.  A read-only SIEM request within scope is allowed.
5.  An out-of-scope target/action is denied and recorded.
6.  Endpoint isolation is not executed until a valid human approval is
    attached to the exact request.
7.  A duplicate request does not execute twice.
8.  Budget exhaustion prevents subsequent requests.
9.  Audit verification passes on an untampered ledger and fails after a
    test mutation.
10. A final report links each finding to evidence and lists
    contradictory evidence/limitations.
11. A reviewer can inspect the complete action timeline in the UI.

## 7.1 Requirement traceability

| Concern | Primary specification | Verification |
|---|---|---|
| No direct agent-to-tool path | [HLD](HLD.md), [ARCHITECTURE](ARCHITECTURE.md) | Network integration test |
| Deterministic allow/deny/approval | [LLD](LLD.md), [DECISIONS](DECISIONS.md) | Rego unit tests and gateway integration tests |
| Duplicate and budget safety | [DATA_MODEL](DATA_MODEL.md), [LLD](LLD.md) | Conflict and concurrent-request tests |
| Human approval integrity | [API](API.md), [SECURITY_THREAT_MODEL](SECURITY_THREAT_MODEL.md) | Altered, expired and replayed approval tests |
| Evidence and report provenance | [DATA_MODEL](DATA_MODEL.md), [TEST_PLAN](TEST_PLAN.md) | Evidence-link and report traceability tests |
| Tamper-evident audit | [ARCHITECTURE](ARCHITECTURE.md), [LLD](LLD.md) | Ledger mutation and verifier tests |

## 8. Assumptions and open questions

-   Initial deployment is a single-team academic prototype.
-   Initial security tools are simulated; no production credentials are
    required.
-   The first scenario is suspicious authentication followed by
    suspicious process activity.
-   The team must decide final LLM provider/model and whether inference
    is local or hosted.
-   The team must choose whether OPA runs as a sidecar/container or is
    embedded in a test-only evaluator; recommended MVP is a separate OPA
    container.
-   Data retention, multi-user identity provider and deployment
    environment need confirmation before any production pilot.
