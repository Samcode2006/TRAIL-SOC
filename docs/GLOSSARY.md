# Glossary --- TRAIL-SOC

> **Status:** Shared vocabulary for the proposal and implementation.
> Use these terms consistently in API fields, UI labels, tests and
> reports so “proposed”, “approved” and “executed” are never conflated.

  -----------------------------------------------------------------------
  Term                                Meaning
  ----------------------------------- -----------------------------------
  Agent                               LLM-driven or deterministic
                                      specialist that analyses assigned
                                      evidence and proposes structured
                                      requests

  Work order                          Bounded capability envelope
                                      defining agent objective, permitted
                                      actions, targets, budgets and stop
                                      conditions

  Tool request                        Structured request from an agent to
                                      invoke a registered tool action

  Gateway                             Mandatory enforcement point that
                                      authenticates, validates,
                                      authorizes, deduplicates and audits
                                      tool requests

  OPA                                 Open Policy Agent, a policy engine
                                      commonly used with Rego policy
                                      language

  Rego                                Declarative policy language used by
                                      OPA

  Adapter                             Narrow implementation of a
                                      registered tool action

  Evidence bundle                     Structured claims, evidence
                                      references, assumptions, confidence
                                      and limitations

  Audit event                         Append-only record of a
                                      security-relevant system event

  Hash chain                          Sequence where each event hash
                                      incorporates the previous event
                                      hash

  Idempotency                         Repeating the same logical request
                                      does not cause duplicate side
                                      effects

  Semantic duplicate                  Different request IDs that
                                      represent effectively the same
                                      operation

  R0--R3                              Project-defined action risk tiers
                                      from read-only to destructive

  Prompt injection                    Malicious instructions embedded in
                                      data consumed by an AI model

  Fail closed                         On uncertainty or dependency
                                      failure, deny tool execution rather
                                      than allow

  Provenance                          Traceable origin and handling
                                      history of evidence or a finding

  Human-in-the-loop                   A human approval step required
                                      before configured actions execute
  -----------------------------------------------------------------------

## Terms used in the presentation

**Controlled autonomy** means the agents can perform useful analysis
within a bounded capability envelope, while authority remains with
deterministic controls and authenticated humans where required.

**Tool runtime authorization** is the decision and enforcement path
between a tool request and an adapter invocation. It includes identity,
scope, schema, budget, policy and idempotency checks.

**Evidence-linked report** is a report whose claims reference immutable
evidence records and distinguish support, contradiction, assumptions and
limitations. A model-generated explanation without resolvable evidence
is not sufficient.

**Tamper-evident ledger** means hash-linked events allow modification or
missing-link detection when compared with a trusted chain head. It does
not mean that a database administrator cannot rewrite the entire chain.
