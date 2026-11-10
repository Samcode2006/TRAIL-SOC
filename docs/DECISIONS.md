# Architecture Decision Records --- TRAIL-SOC

> **Status:** Accepted design decisions for the proposed MVP. A
> decision may be superseded, but the replacement must record its
> security and operational consequences.

## Decision review rule

When implementation pressure conflicts with a decision, update the ADR
before changing the design. In particular, do not silently replace
deterministic gateway enforcement with an agent prompt, UI check or
in-memory shortcut.

## ADR-001: Deterministic policy is the final authority

**Status:** Accepted

**Decision:** Use deterministic policy evaluation (OPA/Rego) as the
final allow/deny/approval authority. LLMs may explain context or
recommend actions but cannot override policy.

**Reason:** Authorization must be reproducible, testable and independent
of prompt manipulation.

**Consequence:** Policy rules and tool risk metadata require engineering
ownership and tests.

## ADR-002: Agents never hold tool credentials

**Status:** Accepted

**Decision:** Agents submit structured requests to the gateway. Only the
gateway/tool runtime can access adapters.

**Reason:** A policy gateway that can be bypassed is not an enforcement
boundary.

**Consequence:** Network segmentation and credential isolation are
mandatory, not optional hardening.

## ADR-003: Simulated tools for MVP

**Status:** Accepted

**Decision:** Use deterministic mock SIEM, threat-intelligence and
endpoint adapters. Endpoint isolation is simulated.

**Reason:** The project can demonstrate governance safely without
production access or destructive side effects.

**Consequence:** The report must distinguish simulation results from
real-world integration claims.

## ADR-004: Modular monolith before microservices

**Status:** Accepted

**Decision:** Start with modular FastAPI application and separately
isolated OPA/tool-runtime containers.

**Reason:** Reduces deployment and integration overhead while preserving
security boundaries.

**Consequence:** Module contracts must remain explicit to allow future
extraction.

## ADR-005: PostgreSQL as source of truth

**Status:** Accepted

**Decision:** Use PostgreSQL for work orders, requests, approvals,
policy decisions, evidence metadata and audit events.

**Reason:** Transactions and uniqueness constraints support correct
idempotency and budget enforcement.

**Consequence:** Redis is optional, not a source of truth.

## ADR-006: RAG is a supporting capability

**Status:** Accepted

**Decision:** Begin with deterministic threat-intelligence fixtures. Add
Qdrant only after the governance path works.

**Reason:** The central contribution is controlled tool authority, not
vector retrieval.

**Consequence:** Do not make RAG a blocker for the MVP.

## ADR-007: Hash-chained audit with explicit limitations

**Status:** Accepted

**Decision:** Implement canonical hash-linked events and a verifier; add
signed checkpoints if time permits.

**Reason:** Provides tamper-evidence and reproducible integrity checks.

**Consequence:** Documentation must not call the database ledger
immutable or tamper-proof. A privileged attacker may rewrite the entire
chain unless a trusted checkpoint exists.
