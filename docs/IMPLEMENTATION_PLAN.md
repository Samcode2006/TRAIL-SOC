# Implementation Plan --- TRAIL-SOC

> **Status:** Proposed delivery plan. Dates and week counts are
> estimates; the dependency order is intentional and should not be
> shortened by adding live agents before the gateway is proven.

## Delivery principle

Build the smallest vertical security slice first: one typed mock tool,
one gateway decision, one audit event and one negative test. Expand to
three agents only after the no-bypass path, approval binding,
idempotency and fail-closed behaviour are observable.

## 1. Delivery strategy

Build a vertical slice first, then add agents and sophistication. The
highest-risk engineering work is not the LLM workflow; it is enforcing a
non-bypassable tool boundary, handling approvals/idempotency correctly,
and preserving audit provenance.

Recommended team size: 3--4 developers. For a smaller team, keep the
same sequence and reduce parallel work. Estimated duration: **8 weeks**
for a robust academic MVP, assuming consistent part-time/full-time
project effort. Adjust after a one-week technical spike.

## 2. Milestones

  ---------------------------------------------------------------------------
  Week                    Milestone               Exit condition
  ----------------------- ----------------------- ---------------------------
  0                       Scope and design freeze Scenario, roles, risk
                                                  tiers, contracts and threat
                                                  model agreed

  1                       Repository and platform UI/API/DB/OPA/Compose boot;
                          skeleton                CI and migrations work

  2                       Work orders and gateway Agent identity, work-order
                          skeleton                lookup, typed tool request,
                                                  no direct tool path

  3                       Deterministic policy    Allow/deny/approval
                          and mock tools          decisions; mock
                                                  SIEM/TI/endpoint adapters

  4                       Idempotency, budgets    Duplicate/race/budget tests
                          and audit               pass; hash-chain verifier
                                                  works

  5                       Agent orchestration     Three specialist agents run
                                                  bounded workflow and
                                                  produce structured findings

  6                       Approval and analyst    Analyst reviews request,
                          console                 approves/rejects, sees
                                                  evidence and timeline

  7                       Reporting and           Full incident report;
                          adversarial evaluation  prompt
                                                  injection/bypass/approval
                                                  tests

  8                       Stabilisation and       Reproducible demo,
                          presentation            benchmark results,
                                                  documentation and final
                                                  review
  ---------------------------------------------------------------------------

## 3. Work breakdown

### Phase 0 --- Scope and contracts (2--3 days)

**Tasks** - Agree one incident scenario: suspicious login followed by
suspicious process. - Define three agent responsibilities and
tool/action registry. - Define R0/R1/R2/R3 risk tiers. - Decide
simulated actions only. - Freeze initial schemas and API contract. -
Create threat model and security invariants.

**Deliverables** - Scenario specification. - Work-order schema. - Tool
registry table. - Initial Rego policy tests. - Architecture decision
records.

**Do not proceed** until team agrees that agents have no direct tool
credentials or network path.

### Phase 1 --- Platform skeleton (Week 1)

**Tasks** - Create monorepo and module boundaries. - Set up FastAPI,
React/Vite and PostgreSQL. - Add Alembic migrations and Pydantic
settings. - Add Docker Compose networks: `frontend`, `agent`, `gateway`,
`tools`, `data`. - Add OPA container and health checks. - Add GitHub
Actions: lint, type-check, unit tests, migration check. - Implement auth
stubs for local development; keep human and workload principals
distinct.

**Deliverables** - `docker compose up` starts core stack. - `/healthz`
and `/readyz`. - CI green on main. - Empty UI shell and API docs.

### Phase 2 --- Work orders and gateway skeleton (Week 2)

**Tasks** - Implement alert and investigation models. - Implement
work-order creation and persistence. - Bind work order to agent
principal. - Implement tool registry and typed request models. -
Implement gateway request lifecycle with deny-by-default behaviour. -
Ensure agents cannot access tool-runtime network. - Add request
correlation IDs.

**Deliverables** - Create alert → investigation → work orders. - Submit
one mock tool request through gateway. - Direct agent-to-tool connection
test fails.

### Phase 3 --- Policy engine and mock adapters (Week 3)

**Tasks** - Implement Rego policies for action allow-list, target scope,
status/expiry and risk tier. - Add R0 read-only SIEM/TI/endpoint
actions. - Add R2 `simulate_isolate` requiring approval. - Add R3
destructive actions denied. - Implement mock adapters with deterministic
fixture data. - Persist policy decision and reason codes. - Fail closed
when OPA unavailable.

**Deliverables** - Policy unit tests. - Allowed request executes once. -
Denied request has no adapter call. - Approval-required request remains
pending and does not execute.

### Phase 4 --- Idempotency, budgets and ledger (Week 4)

**Tasks** - Implement canonical JSON and request digest. - Add DB unique
constraint for idempotency. - Add atomic budget reservation and
request-attempt limits. - Add per-action rate limits. - Add append-only
audit event writer and hash-chain verifier. - Ensure audit write happens
before execution intent; do not execute if audit persistence fails. -
Add tests for parallel requests and adapter timeout.

**Deliverables** - Replay returns prior result. - Changed payload with
same key returns conflict. - Concurrent requests cannot overspend
budget. - Tamper test is detected.

### Phase 5 --- Agent workflow (Week 5)

**Tasks** - Implement LangGraph workflow/state model. - Build Log
Analyst, Threat Intel and Endpoint Analyst nodes. - Give each agent only
the gateway client and its assigned evidence. - Add prompt templates
that explicitly treat logs/reports as untrusted data. - Require
structured outputs with
claims/evidence/assumptions/confidence/limitations. - Add max-step and
time limits. - Add orchestration stop and failure handling.

**Deliverables** - End-to-end synthetic investigation. - No agent
adapter credentials. - All tool requests visible in gateway/audit. -
Findings linked to evidence.

### Phase 6 --- Human approval and UI (Week 6)

**Tasks** - Implement approval queue and request detail view. - Bind
approval to request hash, target, action and expiry. - Revalidate on
approval and before execution. - Add analyst UI pages: investigation
list/detail, agent status, evidence, decisions, approval queue, audit
timeline. - Add role-based access checks. - Add safe stop control.

**Deliverables** - Analyst can approve/reject simulated isolation. -
Altered request cannot reuse approval. - UI clearly distinguishes
proposed, denied, pending, approved and executed actions.

### Phase 7 --- Reporting and evaluation (Week 7)

**Tasks** - Generate Markdown/JSON report. - Include supporting and
contradictory evidence, assumptions and confidence. - Add evidence
provenance and report limitations. - Implement prompt-injection and
abuse fixtures. - Run comparative baseline tests. - Capture latency,
prevention, false denial, duplicate and integrity metrics.

**Deliverables** - Reproducible full scenario. - Evaluation results and
plots/tables. - Adversarial regression suite.

### Phase 8 --- Stabilisation and presentation (Week 8)

**Tasks** - Review code and threat model. - Run dependency/container
scans. - Verify clean installation from README. - Record demo script and
expected outputs. - Fix UI/API edge cases. - Prepare architecture,
novelty, limitations and results presentation. - Tag release and archive
experiment artifacts.

**Deliverables** - Demo-ready MVP. - Complete docs. - Test report. -
Known limitations and future work.

## 4. Parallel work allocation

For a 4-person team:

  -----------------------------------------------------------------------
  Workstream              Owner                   Primary deliverables
  ----------------------- ----------------------- -----------------------
  Gateway & policy        Developer A             Tool registry, OPA,
                                                  validation,
                                                  idempotency, budgets

  Agents & orchestration  Developer B             LangGraph, prompts,
                                                  structured findings,
                                                  workflow

  Backend & audit         Developer C             API, DB, approvals,
                                                  evidence, ledger,
                                                  report

  Frontend & integration  Developer D             Analyst console,
                                                  scenario UX,
                                                  integration tests
  -----------------------------------------------------------------------

All team members participate in threat modelling and adversarial tests.
One person should own the end-to-end demo so integration is not left
until the final week.

For a 3-person team, combine backend/audit with gateway/policy, and
reduce frontend scope to a clean investigation dashboard.

## 5. Definition of done for every tool action

A tool action is not complete until it has: - Typed input schema with
strict unknown-field rejection. - Trusted risk classification in
registry. - Work-order scope rule. - OPA policy and unit tests. -
Adapter-side validation and timeout. - Idempotency behaviour. - Audit
event on allow, deny and failure. - At least one positive and two
negative tests. - Documentation and sample request/response.

## 6. Recommended development order

1.  Mock tool API with no agent.
2.  Gateway with deterministic policy.
3.  Audit and idempotency.
4.  Human approval.
5.  Agent calling gateway.
6.  UI and report.
7.  Optional RAG and additional integrations.

This order deliberately proves the security boundary before adding
autonomous agents.

## 7. Risks and contingency

  -----------------------------------------------------------------------
  Risk                                Contingency
  ----------------------------------- -----------------------------------
  LLM/provider instability            Use deterministic agent stubs for
                                      integration tests and demo fallback

  OPA/Rego learning curve             Start with 3--5 simple policies and
                                      expand

  Agent orchestration complexity      Implement agents as sequential
                                      graph nodes before parallel
                                      execution

  Scope creep                         Freeze one scenario and three
                                      agents for MVP

  Audit complexity                    Implement append-only DB + hash
                                      chain first; signed checkpoints
                                      later

  UI takes too long                   Use a minimal table/detail UI;
                                      prioritize workflow correctness

  Real integration unavailable        Keep all tools simulated; this is
                                      sufficient to prove gateway
                                      semantics

  Team integration delays             Agree OpenAPI/Pydantic contracts in
                                      Week 0 and use mock clients
  -----------------------------------------------------------------------

## 8. Stretch goals (only after MVP)

-   Qdrant retrieval over curated threat intelligence.
-   Real read-only SIEM integration.
-   Signed audit checkpoints and external timestamping.
-   Multi-tenant policy isolation.
-   Durable workflow engine.
-   Kubernetes deployment.
-   Policy simulation / what-if interface.
-   Agent quality evaluation and confidence calibration.
