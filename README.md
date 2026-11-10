# TRAIL-SOC

## Tool Runtime Authorization & Investigation Ledger for SOC Agents

TRAIL-SOC (the **Cyberforge** project) is a security research and
engineering project for building safer AI-assisted SOC investigations.
It allows specialist agents to investigate a suspicious alert, but does
not allow an agent to authorise its own tool calls.

The central rule is:

> **Agents may reason about security data and propose actions. Only the
> deterministic policy gateway may authorise an action.**

The repository currently contains the project specification and design
baseline. It is **not an implementation** and does not claim that the
described services, tests, or security controls are already running.

## Why this project exists

AI agents are useful for correlating logs, enriching indicators and
summarising incidents. They also process attacker-controlled material:
logs, threat reports, endpoint data and tool output can contain prompt
injection. An unconstrained agent may request tools outside its role,
repeat a side effect, or produce a conclusion that cannot be traced back
to evidence.

TRAIL-SOC addresses those risks with:

- bounded work orders that define objective, scope, tools, targets,
  budgets and expiry;
- a mandatory gateway between agents and every security tool;
- typed, allow-listed tool actions rather than shell or arbitrary URL
  execution;
- deterministic policy evaluation independent of LLM output;
- human approval for high-impact simulated actions;
- idempotency, rate and budget controls;
- isolated mock tool adapters with no credentials in agent context; and
- an append-only, hash-linked investigation ledger plus evidence-linked
  findings.

## Demonstration scenario

The MVP uses one reproducible synthetic incident:

1. A SIEM alert reports repeated failed logins followed by a successful
   login from an unfamiliar address.
2. The log analyst correlates authentication events.
3. The threat-intelligence agent looks up the indicator and adds context.
4. The endpoint analyst reviews the device and process tree.
5. The agents return claims, evidence for and against, assumptions,
   confidence and limitations.
6. The endpoint analyst proposes `endpoint.simulate_isolate`.
7. The gateway returns `REQUIRE_APPROVAL`; it does not execute the
   adapter.
8. An authenticated analyst approves the exact request, which is
   revalidated and then executed by the simulated endpoint adapter.
9. The final report includes the complete decision, approval, execution
   and evidence history.

The scenario also includes negative tests: prompt injection in a log,
out-of-scope targets, an unapproved action, duplicate requests,
concurrent budget consumption, expired approval and an unavailable
policy service.

## Architecture at a glance

```text
Synthetic alert / analyst
            |
            v
      FastAPI API
            |
            v
       Orchestrator
       /    |     \
   Log    Threat   Endpoint       (untrusted agents)
  agent    intel     agent
       \    |     /
        typed ToolRequest
               |
               v
       TRAIL-SOC gateway
   auth -> scope -> schema -> idempotency
          -> budget -> OPA/Rego -> audit
               |
         approved requests only
               v
       simulated tool adapters
      SIEM | Threat Intel | Endpoint
               |
               v
       evidence + final report
```

Agents have no adapter credentials and no direct network route to the
tool runtime. A denied or approval-pending request never reaches an
adapter. OPA, database, identity or audit failures fail closed.

## Selected MVP stack

| Area | MVP choice | Why |
|---|---|---|
| Analyst UI | React, TypeScript, Vite, Tailwind, shadcn/ui, Recharts | Clear investigation and approval workflow |
| API | Python, FastAPI, Pydantic | Typed contracts and fast local development |
| Orchestration | LangGraph, with deterministic stubs for tests | Bounded specialist workflow |
| Policy | OPA/Rego, enforced by the gateway | Reproducible authorization |
| State | PostgreSQL | Transactions, constraints and audit metadata |
| Object evidence | Local filesystem first; MinIO-compatible storage later | Simple reproducible development |
| Tools | Deterministic simulated SIEM, TI and endpoint adapters | No production side effects |
| Runtime | Docker Compose | Repeatable local deployment |
| Tests | Pytest, HTTPX, policy and adversarial tests | Security boundary and workflow coverage |
| CI | GitHub Actions | Repeatable checks |

Redis, Celery, Qdrant, Kubernetes, vendor integrations and real
credentials are **future options**, not MVP requirements. The slide deck
lists alternatives; this repository selects the smallest safe baseline.

## Repository status and layout

This repository is currently a documentation-first specification. The
layout below is the intended implementation layout:

```text
apps/
  api/                 FastAPI API, domain modules and gateway
  web/                 React analyst console
services/
  orchestrator/        LangGraph workflow and agent boundaries
  tool-runtime/        Simulated adapters
  audit/               Ledger writer and verifier
policies/              Versioned Rego policies and tests
contracts/             JSON schemas and OpenAPI snapshots
scenarios/             Seed data and reproducible incident fixtures
tests/
  unit/ integration/ adversarial/ evaluation/
infra/                 Compose, migrations and CI support
docs/                  Requirements and design package
```

## Documentation map

Read these documents in order:

1. [REQUIREMENTS.md](docs/REQUIREMENTS.md) - goals, actors, functional
   requirements and acceptance criteria.
2. [HLD.md](docs/HLD.md) - boundaries, components, flows and failure
   behaviour.
3. [ARCHITECTURE.md](docs/ARCHITECTURE.md) - deployment topology and
   allowed dependencies.
4. [LLD.md](docs/LLD.md) - schemas, gateway algorithm, adapters and
   implementation rules.
5. [DATA_MODEL.md](docs/DATA_MODEL.md) - entities, constraints,
   retention and provenance.
6. [API.md](docs/API.md) - versioned endpoint contract and examples.
7. [SECURITY_THREAT_MODEL.md](docs/SECURITY_THREAT_MODEL.md) - threats,
   invariants and abuse cases.
8. [TEST_PLAN.md](docs/TEST_PLAN.md) - tests, metrics, targets and exit
   criteria.
9. [IMPLEMENTATION_PLAN.md](docs/IMPLEMENTATION_PLAN.md) - phased build
   plan and definition of done.
10. [DECISIONS.md](docs/DECISIONS.md) - accepted architecture decisions.
11. [GLOSSARY.md](docs/GLOSSARY.md) - shared vocabulary.

The source presentation is [Cyberforge.pdf](docs/Cyberforge.pdf). It
communicates the project vision; the Markdown documents add the
engineering detail needed to implement and evaluate it.

## Safety and scope

All default data and tools are synthetic. Endpoint isolation is a
simulation. The MVP must not connect to production endpoints, execute
arbitrary commands, modify real SIEM/EDR state, delete evidence or
disable security controls.

If real integrations are added later, they must be read-only by
default, least-privileged, explicitly allow-listed, separately reviewed
and isolated behind the same gateway. No agent may receive raw
integration credentials.

## Contribution rule

Any new tool action must have a typed schema, trusted risk tier,
work-order scope rule, policy tests, adapter validation, timeout,
idempotency behaviour, audit events and positive/negative tests before
it is considered complete. See the [implementation definition of done](docs/IMPLEMENTATION_PLAN.md#5-definition-of-done-for-every-tool-action).
