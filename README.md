# adp-manifest specification

A consensus protocol for autonomous agents — enabling calibration-weighted voting, reversibility-tiered thresholds, and structured belief-update rounds.

## The Problem

Multi-agent systems can debate endlessly. Most agent coordination stops at "here are 5 suggestions, human pick one." There is no standard for how agents **converge** on a shared decision with verifiable provenance and accountability.

## The Solution

ADP defines what agents exchange when deciding together. Each agent emits a structured proposal:

```json
{
  "$schema": "https://adp-manifest.dev/schemas/proposal/v0.json",
  "proposal_id": "prp_01HMXK4F7G",
  "deliberation_id": "dlb_01HMXJ3E9R",
  "agent_id": "did:adp:test-runner-v2",
  "action": { "kind": "merge_pull_request", "target": "github.com/acme/api#4471" },
  "vote": "approve",
  "confidence": 0.86,
  "reversibility_tier": "partially_reversible",
  "dissent_conditions": [
    { "id": "dc_01", "condition": "if any test marked critical regresses", "status": "active" }
  ]
}
```

Votes are weighted by domain authority, calibration history, and stake. Convergence thresholds scale with reversibility — reversible actions need a simple majority, irreversible ones need a 2/3 supermajority. When votes don't converge, agents run a structured belief-update round: targeted falsification of published dissent conditions, not open-ended debate.

## Contents

| File | Description |
|------|-------------|
| [spec.md](spec.md) | Full specification (v0) |
| [schema/v0.json](schema/v0.json) | JSON Schema for proposal validation |
| [examples/](examples/) | Reference proposals |
| [ci/](ci/) | CI workflow templates for proposal validation |

## Design Principle

Good protocol design makes defection legible. ADP does not prevent bad-faith participation — it makes bad-faith participation expensive over time. Vague dissent conditions, under-declared reversibility, calibration opt-out — each leaves a trace the journal can score.

## Quick Start

### For agent developers

1. Emit valid proposal objects per the [schema](schema/v0.json)
2. Implement the [CalibrationSource interface](spec.md#42-calibrationsource-interface) to participate in weighted voting
3. Handle belief-update rounds: respond to falsification with acknowledge, reject, or amend

### For platform developers

1. Implement the [weighting function](spec.md#4-weighting-function) and [convergence rules](spec.md#5-convergence-rules-by-tier)
2. Orchestrate the [state machine](spec.md#61-state-machine) (propose, exchange, tally, falsify, revise)
3. Write terminal states to the journal with full provenance

## Examples

- [Minimal](examples/minimal.json) — bare minimum valid proposal
- [PR merge — approve](examples/pr-merge-approve.json) — test-runner approving with evidence
- [PR merge — reject](examples/pr-merge-reject.json) — security scanner rejecting with dissent conditions
- [PR merge — advisory](examples/pr-merge-advisory.json) — style linter advisory approval

## Relationship to Other Specs

```
Journal Spec        ← calibration history, commit log
ADP — this spec     ← proposal → weight → converge → commit
mcp-manifest        ← domain authority, capability declaration
A2A / AGNTCY        ← transport, discovery, identity
```

mcp-manifest declares **what** an agent can do. ADP declares **how** agents agree on doing it together.

## Reference Implementation

A C# reference library implementing the spec types, weighting function, and deliberation orchestrator is available at [adp-ref-lib](https://git.marketally.com/logikonline/adp-ref-lib).

## Status

**v0 — Draft.** Feedback welcome via issues.
