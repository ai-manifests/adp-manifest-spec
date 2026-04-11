# Agent Deliberation Protocol (ADP) — v0

**Status:** Draft  
**Date:** 2026-04-11  
**Schema namespace:** `https://adp-manifest.dev/schemas/`

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

---

## 1. Purpose and Non-Goals

### 1.1 Purpose

ADP defines how autonomous agents converge on a shared decision through
calibration-weighted voting and reversibility-tiered thresholds. It specifies the
structure of proposals agents emit, the function that weights their votes, the
rules that determine convergence, and the belief-update protocol agents follow
when initial votes do not converge.

ADP is designed for federated deployment without central coordination; see
Section 12.

### 1.2 Design Principle

Good protocol design makes defection legible.

ADP does not prevent bad-faith participation. It makes bad-faith participation
expensive over time. Vague dissent conditions, under-declared reversibility,
habitual abstention from calibration — each is possible, each leaves a trace
the journal can score, and each degrades the defecting agent's weight in future
deliberations. The protocol's job is to make every epistemic posture visible;
the calibration loop's job is to make dishonest postures costly.

### 1.3 Non-Goals

ADP does **not** specify:

- **Transport.** How agents discover or message each other (see A2A, AGNTCY).
- **Identity.** How agent identifiers are issued or verified.
- **Capability declaration.** What an agent can do (see mcp-manifest).
- **Journal storage.** How calibration history and commit logs are persisted
  (see the forthcoming Journal spec).

ADP assumes agents can already find and talk to each other. It defines only
what they exchange when deciding together and how they reach — or fail to
reach — agreement.

---

## 2. Terminology

| Term | Definition |
|---|---|
| **Deliberation** | A bounded decision process identified by a `deliberation_id`. One action, one or more rounds, one termination state. |
| **Proposal** | A structured object an agent submits to a deliberation: vote, confidence, justification, dissent conditions, and metadata. Defined in Section 3. |
| **Round** | One cycle of the belief-update loop: falsification attempts, responses, vote revisions, retally. Round 0 is the initial tally before any belief-update. |
| **Convergence** | The state in which weighted votes meet the threshold for the declared reversibility tier and the participation floor is satisfied. |
| **Commit** | Writing the converged decision, full provenance, and vote history to the journal. The deliberation's terminal write. |
| **Abstention** | A vote that declines to approve or reject. Abstaining agents' weight counts toward the deliberation total but not toward the participation floor or the approval/rejection tally. |
| **Reversibility tier** | A per-proposal classification — `reversible`, `partially_reversible`, or `irreversible` — that determines the convergence threshold. Declared by the proposing agent, challengeable by others. |
| **Calibration weight** | A score in [0, 1] reflecting how often an agent's stated confidence has matched observed outcomes in a given domain. Sourced from a CalibrationSource (Section 4.2). |
| **Domain authority** | A score in [0, 1] reflecting an agent's declared and verified expertise in a decision class. Sourced from mcp-manifest. |
| **Belief-update round** | A structured round in which agents attempt to falsify each other's dissent conditions, respond to falsification evidence, and optionally revise their votes. Not open-ended debate — targeted falsification of published update functions (Section 6.2). |
| **Participation floor** | The minimum fraction of total deliberation weight that must be cast as non-abstaining votes for a tally to be valid. Default: 50%. |
| **Falsification** | Submitting evidence that addresses a specific, enumerated dissent condition published by another agent. The only mechanism for changing votes in a belief-update round. |

---

## 3. The Proposal Object

A proposal is the atomic unit of participation in a deliberation. Every agent
that joins a deliberation MUST submit exactly one proposal. The proposal is
immutable once submitted; epistemic movement during belief-update rounds is
recorded as amendments to dissent conditions and entries in the revisions
array, never as mutations of the original fields.

### 3.1 Schema

```json
{
  "$schema": "https://adp-manifest.dev/schemas/proposal/v0.json",
  "proposal_id": "prp_01HMXK4F7G...",
  "deliberation_id": "dlb_01HMXJ3E9R...",
  "agent_id": "did:adp:test-runner-v2",
  "timestamp": "2026-04-11T14:32:09.221Z",

  "action": {
    "kind": "merge_pull_request",
    "target": "github.com/acme/api#4471",
    "parameters": { "strategy": "squash" }
  },

  "vote": "approve",
  "confidence": 0.86,

  "domain_claim": {
    "domain": "code.correctness",
    "authority_source": "mcp-manifest:test-runner-v2#authorities"
  },

  "reversibility_tier": "partially_reversible",

  "blast_radius": {
    "scope": ["service:api", "consumers:web,mobile"],
    "estimated_users_affected": 12000,
    "rollback_cost_seconds": 90
  },

  "justification": {
    "summary": "All 1,847 tests pass; coverage delta +0.3 pp; no flaky retries.",
    "evidence_refs": [
      "journal:dlb_01HMXJ.../evidence/test-run-9912",
      "ci:github-actions/run/8821443"
    ]
  },

  "stake": {
    "declared_by": "self",
    "magnitude": "high",
    "calibration_at_stake": true
  },

  "dissent_conditions": [
    {
      "id": "dc_01",
      "condition": "if any test marked critical regresses",
      "status": "active",
      "amendments": [],
      "tested_in_round": null,
      "tested_by": null
    },
    {
      "id": "dc_02",
      "condition": "if coverage delta is negative",
      "status": "active",
      "amendments": [],
      "tested_in_round": null,
      "tested_by": null
    }
  ],

  "revisions": []
}
```

### 3.2 Dissent Conditions Are Append-Only

The `dissent_conditions` array is a log of epistemic movement, not a
current-state snapshot.

When a dissent condition is targeted during a belief-update round, the
`status`, `tested_in_round`, and `tested_by` fields are updated, and any
amendment is appended to the `amendments` array. The original `condition`
string is never overwritten.

```json
{
  "id": "dc_02",
  "condition": "if coverage delta is negative",
  "status": "amended",
  "amendments": [
    {
      "round": 1,
      "new_condition": "if coverage delta is negative on critical paths",
      "reason": "scanner evidence showed non-critical path coverage drop is tolerable",
      "triggered_by": "evidence:journal:dlb_.../ev_887"
    }
  ],
  "tested_in_round": 1,
  "tested_by": "did:adp:security-scanner-v3"
}
```

This gives the journal everything it needs: which conditions existed, which
were tested, by whom, in which round, and what amendments were made and why.
It also makes the Section 6 state machine cleaner, because each transition
writes to a specific condition's history rather than mutating the proposal
opaquely.

A dissent condition's `status` MUST be one of:

| Status | Meaning |
|---|---|
| `active` | Condition has not been tested or remains unfalsified. |
| `falsified` | Evidence was submitted and the owning agent acknowledged the condition was met. |
| `amended` | The owning agent narrowed or revised the condition. The current condition is the last entry in `amendments`. |
| `withdrawn` | The owning agent removed the condition (with reason). |

An agent MUST NOT delete entries from `dissent_conditions`. Withdrawn
conditions remain in the array with `status: "withdrawn"`.

### 3.3 Field Rationale

**`vote` is separate from `action`** because the same action can attract
`approve`, `reject`, `abstain`, or `revise` votes across agents and rounds.
Both are needed to compute convergence.

**`confidence`** is the agent's self-assessed probability that the action is
correct. Calibration scoring (journal spec) compares this to outcomes, which
is what makes overconfident agents lose weight over time.

**`domain_claim`** points back at mcp-manifest for the authority assertion
rather than re-declaring it inline. ADP does not adjudicate authority — it
consumes it. If two agents claim overlapping domains, that is resolved upstream
in the manifest layer or noted as an open question (Section 11).

**`reversibility_tier`** is per-proposal, not per-action-type, because
reversibility is contextual. Merging a PR to `main` is `partially_reversible`;
merging to a release branch mid-deploy is `irreversible`. The proposing agent
makes the call; other agents can challenge it (Section 6.3).

**`blast_radius`** is structured rather than prose so other agents can reason
about it programmatically. The three sub-fields — scope, estimated users
affected, rollback cost — are the minimum viable triple: enough to compare
proposals, not so much that agents cannot fill them in honestly.

**`justification.evidence_refs`** uses URIs to point at supporting evidence.
The `journal:` URI scheme is defined in ADJ Section 3.6 and resolves to
journal entries via the query contract. Non-journal evidence (CI systems,
external monitoring) uses implementation-specific schemes (e.g., `ci:`,
`scan:`, `monitoring:`).

**`dissent_conditions`** pre-declares what would change this agent's vote.
This means the belief-update round has something concrete to operate on instead
of free-form argument. It is the closest thing in the schema to "shared
intent" — the agent publishes its update function alongside its vote.

**`stake.calibration_at_stake`** — when `true`, this proposal will be scored
against outcomes and feed back into the agent's calibration weight. Agents can
opt out (`false`) for low-signal decisions, but doing so habitually SHOULD
itself degrade their authority. That is a journal-spec concern, noted in
Section 11.

**`revisions`** records vote changes during belief-update rounds. Each entry
captures the prior vote, the new vote, the reason, and the round number.
The initial `vote` field is never overwritten; the current vote is the last
entry in `revisions`, or the original `vote` if `revisions` is empty.

```json
"revisions": [
  {
    "round": 1,
    "prior_vote": "reject",
    "new_vote": "abstain",
    "prior_confidence": 0.79,
    "new_confidence": null,
    "reason": "dissent conditions dc_01 and dc_02 falsified by test-runner evidence",
    "timestamp": "2026-04-11T14:35:22.109Z"
  }
]
```

---

## 4. Weighting Function

Votes are not equal. An agent's weight in a deliberation is a function of its
domain authority, calibration history, the staleness of that history, and its
declared stake.

**Graceful degradation.** Calibration is optional. ADP without ADJ degrades
gracefully to equal-weight voting plus domain authority: when no
CalibrationSource is available, the `calibration` and `decay` terms default
to 1.0, and the weighting function reduces to `authority × stake_factor`.
This means ADP is deployable before a journal exists, and gains the
calibration loop when one becomes available.

### 4.1 Formula

```
weight(agent, decision_class) = authority × calibration × decay(staleness, decision_class) × stake_factor
```

Each term:

| Term | Source | Range |
|---|---|---|
| `authority` | mcp-manifest domain authority for `decision_class` | [0, 1] |
| `calibration` | `CalibrationSource.getScore(agent_id, domain).value` | [0, 1] |
| `decay` | Exponential decay applied to calibration staleness | (0, 1] |
| `stake_factor` | Derived from declared `stake.magnitude` | [0, 1] |

**Stake factor mapping.** Implementations MUST use these defaults unless
overridden by deliberation-level configuration:

| Magnitude | Factor |
|---|---|
| `high` | 1.00 |
| `medium` | 0.85 |
| `low` | 0.50 |

### 4.2 CalibrationSource Interface

The spec defines an abstract contract. The journal spec and reference
implementations provide concrete implementations.

```
CalibrationSource:
  getScore(agent_id, domain) → CalibrationScore
  getDefault(domain)         → CalibrationScore

CalibrationScore:
  value       : number     // [0.0, 1.0]
  sample_size : integer    // >= 0
  staleness   : duration   // ISO 8601
```

**`value`** is bounded [0, 1] so the weighting function stays composable.

**`sample_size`** lets the weighting function discount low-evidence scores. An
implementation SHOULD apply a sample-size discount. A suggested default:

```
effective_calibration = value × (1 − 1 / (1 + sample_size))
```

A score of 0.95 from 4 samples should not dominate a score of 0.78 from 400.

**`staleness`** is raw duration since last calibration update. Decay is applied
by the weighting function (Section 4.3), not by the CalibrationSource, because
decay rates are decision-class-specific.

**Bootstrap default:** `getDefault(domain)` MUST return
`{ value: 0.5, sample_size: 0, staleness: PT0S }`. New agents enter with
neutral weight that is heavily discounted by the sample-size term. They earn
authority through participation.

### 4.3 Decay Function

CalibrationSource returns raw staleness. The weighting function applies
exponential decay with a decision-class-specific half-life:

```
decay(staleness, decision_class) = 2 ^ (−staleness_days / half_life_days)
```

An agent that was well-calibrated a year ago on a domain that has since
drifted should not coast on old credit. Suggested default half-lives:

| Decision Class | Half-Life | Rationale |
|---|---|---|
| `code.correctness` | 180 days | Test infrastructure changes slowly. |
| `security.policy` | 90 days | Threat landscape and compliance requirements shift. |
| `api.compatibility` | 30 days | Consumer ecosystems evolve quickly. |
| `code.style` | 365 days | Style conventions are stable. |

These are starting points, not gospel. Implementations MAY override half-lives
per decision class. The half-life table SHOULD be declared in deliberation-level
configuration so all agents in a deliberation use the same decay curve.

### 4.4 Worked Arithmetic

Computing weight for `did:adp:test-runner-v2` on decision class `code.correctness`:

```
authority   = 0.90          (from mcp-manifest)
calibration = 0.85          (from CalibrationSource, sample_size: 312)
staleness   = 18 days       (from CalibrationSource)
half_life   = 180 days      (code.correctness default)
decay       = 2^(−18/180)   = 2^(−0.10) = 0.933
stake       = high → 1.00

weight = 0.90 × 0.85 × 0.933 × 1.00 = 0.71
```

---

## 5. Convergence Rules by Tier

### 5.1 Threshold Table

Convergence requires that the approval fraction — approve weight divided by
total non-abstaining weight — meets or exceeds the tier threshold, **and** that
any tier-specific additional conditions are satisfied.

| Tier | Approval Threshold | Additional Conditions | Belief-Update |
|---|---|---|---|
| `reversible` | Simple majority (> 50%) | None. | OPTIONAL. |
| `partially_reversible` | 60% weighted approval | No vetoes from agents with domain authority ≥ 0.8 for the decision class. | REQUIRED if threshold not met in round 0. |
| `irreversible` | 2/3 (66.7%) weighted supermajority | Zero active contradictions on irreversible dimensions. All domain authorities with authority ≥ 0.7 MUST have voted (no abstention). | REQUIRED. Minimum one belief-update round even if threshold is met in round 0. |

**Approval fraction** is computed over non-abstaining weight only:

```
approval_fraction = sum(weight of approve votes) / sum(weight of non-abstaining votes)
```

### 5.2 Participation Floor

A tally is valid only if the participation floor is met:

```
participation = sum(weight of non-abstaining votes) / sum(weight of all agents in deliberation)
```

The participation floor MUST be at least **50%** of total deliberation weight.
Implementations MAY configure a higher floor.

If the participation floor is not met, the tally fails regardless of the
approval fraction. This prevents a small minority from deciding for everyone
when other agents abstain.

### 5.3 Tier Escalation Mid-Round

When a tier challenge (Section 6.3) succeeds and the reversibility tier
escalates (e.g., `partially_reversible` → `irreversible`), the convergence
threshold rises mid-deliberation. The new threshold applies to the next tally
and all subsequent tallies.

This creates a self-correcting incentive: under-declaring reversibility is
risky because it invites a challenge that makes the proposer's own action
harder to pass. The rational strategy is honest declaration.

| Escalation | Effect |
|---|---|
| `reversible` → `partially_reversible` | Threshold rises from >50% to 60%. Domain authority vetoes activated. |
| `reversible` → `irreversible` | Threshold rises from >50% to 66.7%. Mandatory belief-update round triggered. |
| `partially_reversible` → `irreversible` | Threshold rises from 60% to 66.7%. All high-authority agents must vote. |

> **Conservative Drift**
>
> ADP exhibits a conservative drift: over-declaration of reversibility is a
> stable equilibrium that the calibration loop only slowly corrects. An agent
> that habitually marks `reversible` actions as `partially_reversible` sets a
> higher threshold than necessary, making convergence harder. This is the
> intended default for v0 — over-caution is safer than under-caution.
>
> Production deployments SHOULD monitor the over-declaration ratio in their
> journal and tune accordingly. The journal spec (forthcoming) will define
> metrics for detecting systematic over-declaration.

---

## 6. The Belief-Update Round

A belief-update round is not deliberation. It is targeted falsification of
published update functions. The search space is bounded — agents can only
falsify what was declared — and each round either resolves or amends a specific
dissent condition.

### 6.1 State Machine

```
                    ┌──────────────────────────────────────────┐
                    │                                          │
  PROPOSED ──→ EXCHANGE ──→ TALLY ──→ CONVERGED               │
                    │          │                               │
                    │          ├──→ PARTIAL_COMMIT             │
                    │          │                               │
                    │          ├──→ DEADLOCKED                 │
                    │          │                               │
                    │          └─(rounds remaining)──→ FALSIFY │
                    │                                    │     │
                    │          ┌──────────────────────────┘     │
                    │          ▼                                │
                    │       RESPOND ──→ REVISE ──→ TALLY ──────┘
                    │
                    └──→ CHALLENGE_TIER ──→ (resolved) ──→ EXCHANGE
```

**State descriptions:**

| State | Description |
|---|---|
| `PROPOSED` | Agents submit proposals. Transitions to EXCHANGE when all expected agents have submitted or a configured timeout expires. |
| `EXCHANGE` | All proposals are visible to all agents. Agents MAY submit a `challenge_tier` message. Transitions to TALLY. |
| `TALLY` | Weighted votes are computed and checked against the tier threshold and participation floor. Transitions to a terminal state or to FALSIFY. |
| `FALSIFY` | Agents submit evidence targeting specific dissent conditions of other agents. Each piece of evidence MUST reference a `dissent_condition.id`. |
| `RESPOND` | Agents whose dissent conditions were targeted MUST respond within a configured timeout. |
| `REVISE` | Agents MAY revise their vote, confidence, or dissent conditions. Revisions are appended to the proposal's `revisions` array. |
| `CHALLENGE_TIER` | A tier challenge is in progress (Section 6.3). |
| `CONVERGED` | Terminal. Threshold and participation floor met. |
| `PARTIAL_COMMIT` | Terminal. Reversible subset proceeds; remainder deferred. |
| `DEADLOCKED` | Terminal. No convergence. Full debate trace escalated. |

The maximum number of belief-update rounds is configurable. The default is
**3**. An implementation MUST enforce this limit. Silence is not an option —
agents in a deliberation MUST respond to falsification attempts within the
configured timeout. Failure to respond within the timeout is treated as a
restatement (the agent's vote and conditions stand unchanged).

### 6.2 Falsification, Not Persuasion

A belief-update round is not free-form argument. Agent B does not argue with
Agent A. B submits evidence against one of A's **enumerated** dissent
conditions. A is then obligated to respond in exactly one of three ways:

1. **Acknowledge.** The condition was met. A's `dissent_condition.status`
   changes to `falsified`. A SHOULD revise their vote accordingly.
2. **Reject.** The evidence does not satisfy the condition. A MUST provide a
   reason. The condition remains `active`.
3. **Amend.** A narrows or revises the condition. The amendment is appended to
   `dissent_conditions[].amendments`. The condition's status changes to
   `amended`. This counts as a revise and is logged.

This makes the 3-round default defensible: the search space is bounded by the
set of declared conditions, and each round either resolves or amends a specific
one. An agent that publishes vague or unfalsifiable dissent conditions is
effectively defecting from the protocol. The journal can score this — conditions
that are never tested across deliberations are noise, and habitual noise
degrades calibration credit.

> **Evidence access asymmetry** is a known limitation. An agent may publish
> a dissent condition that is technically falsifiable but practically
> unfalsifiable within the evidence available to other agents in the
> deliberation (e.g., "if latency p99 exceeds 200ms under production load"
> when no agent has production metrics access). The protocol handles this
> correctly — the condition survives unfalsified, the vote stands — but the
> journal can detect the pattern retrospectively through a
> "conditions-tested ratio." See Section 11.

### 6.3 Tier Challenges

A tier challenge is a distinct message type in the state machine. Any agent MAY
challenge the `reversibility_tier` declared in another agent's proposal.

**Rules:**

- A tier challenge MUST include evidence or justification for why the tier
  should be higher (challenges can only escalate, never de-escalate).
- Each agent is limited to **one** tier challenge per deliberation. This
  prevents griefing — an agent cannot burn rounds by repeatedly re-challenging.
- Tier challenges are resolved during the EXCHANGE phase, **before** votes
  are tallied.
- The proposing agent MUST respond: accept the escalation (tier changes) or
  reject with a reason (tier stands, but the challenge is logged).
- If a tier challenge is accepted, the new tier applies to all subsequent
  tallies. The convergence threshold rises per the Section 5.3 table.

**Tier challenge message:**

```json
{
  "type": "challenge_tier",
  "challenger_id": "did:adp:security-scanner-v3",
  "target_proposal_id": "prp_01HMXK4F7G...",
  "current_tier": "partially_reversible",
  "proposed_tier": "irreversible",
  "justification": "Release branch deploy is in progress; rollback requires coordinated downtime.",
  "evidence_refs": ["ci:deploy-pipeline/run/2241"]
}
```

### 6.4 Round Timeout

Each state in the belief-update loop MUST have a configurable timeout. The
defaults are implementation-defined. When a timeout expires:

- In FALSIFY: no further evidence is accepted; transition to RESPOND.
- In RESPOND: non-responding agents' conditions and votes stand unchanged.
- In REVISE: non-revising agents' votes stand unchanged; transition to TALLY.

---

## 7. Termination States

Every deliberation terminates in exactly one of three states. Each terminal
state writes a record to the journal.

### 7.1 Converged

The approval fraction meets or exceeds the tier threshold, the participation
floor is met, and all tier-specific additional conditions (Section 5.1) are
satisfied.

The journal record MUST include:

- The action to be executed.
- All proposals (with full dissent condition and revision history).
- The final weighted tally.
- The reversibility tier (including any mid-round escalation).
- The round count.

The action is committed for execution.

### 7.2 Partial Commit

The deliberation did not converge, but a reversible subset of the action can
proceed. Partial commit is available when:

- The action is decomposable into independently executable sub-actions.
- At least one sub-action has `reversibility_tier: "reversible"` and meets the
  reversible threshold (simple majority).

The reversible subset proceeds. The remainder is deferred with the full debate
trace attached. This is how human incident response actually works — commit to
what is safe, defer what is contested.

The journal record MUST include:

- The sub-actions committed and the sub-actions deferred.
- The full tally and deliberation trace.
- The reason partial commit was triggered (which threshold failed).

### 7.3 Deadlocked

The deliberation did not converge and no reversible subset exists. The
deliberation is escalated — but with the full debate trace, not a menu of
suggestions.

The journal record MUST include:

- All proposals with full history.
- The final tally.
- The specific threshold or condition that was not met.
- The escalation target (human, higher-authority agent, or configured handler).

Deadlock is not failure. It is the protocol correctly identifying that the
agents in the room cannot resolve this decision with the information and
authority available to them.

---

## 8. Worked Example: The PR Merge

Three agents deliberate on whether to auto-merge PR #4471 (`acme/api`).
This section shows the actual JSON and arithmetic at each step.

### 8.1 Agents and Weights

| Agent | Domain | Authority | Calibration | Samples | Staleness | Half-Life | Decay | Stake | **Weight** |
|---|---|---|---|---|---|---|---|---|---|
| `test-runner-v2` | `code.correctness` | 0.90 | 0.85 | 312 | 18 d | 180 d | 0.933 | high (1.00) | **0.71** |
| `security-scanner-v3` | `security.policy` | 0.85 | 0.83 | 187 | 12 d | 90 d | 0.912 | high (1.00) | **0.64** |
| `style-linter-v1` | `code.style` | 0.30 | 0.72 | 89 | 4 d | 365 d | 0.992 | medium (0.85) | **0.18** |

Weight computations:

```
test-runner:  0.90 × 0.85 × 0.933 × 1.00 = 0.714 ≈ 0.71
scanner:      0.85 × 0.83 × 0.912 × 1.00 = 0.643 ≈ 0.64
linter:       0.30 × 0.72 × 0.992 × 0.85 = 0.182 ≈ 0.18
                                             ─────
                              total weight:  1.53
```

### 8.2 Snapshot 1 — Initial Proposals (Round 0)

**Test-runner** submits:

```json
{
  "proposal_id": "prp_01HMXK4F7G",
  "deliberation_id": "dlb_01HMXJ3E9R",
  "agent_id": "did:adp:test-runner-v2",
  "action": { "kind": "merge_pull_request", "target": "github.com/acme/api#4471", "parameters": { "strategy": "squash" } },
  "vote": "approve",
  "confidence": 0.86,
  "domain_claim": { "domain": "code.correctness", "authority_source": "mcp-manifest:test-runner-v2#authorities" },
  "reversibility_tier": "partially_reversible",
  "blast_radius": { "scope": ["service:api", "consumers:web,mobile"], "estimated_users_affected": 12000, "rollback_cost_seconds": 90 },
  "justification": { "summary": "All 1,847 tests pass; coverage delta +0.3 pp; no flaky retries.", "evidence_refs": ["journal:dlb_01HMXJ.../evidence/test-run-9912", "ci:github-actions/run/8821443"] },
  "stake": { "declared_by": "self", "magnitude": "high", "calibration_at_stake": true },
  "dissent_conditions": [
    { "id": "dc_tr_01", "condition": "if any test marked critical regresses", "status": "active", "amendments": [], "tested_in_round": null, "tested_by": null },
    { "id": "dc_tr_02", "condition": "if coverage delta is negative", "status": "active", "amendments": [], "tested_in_round": null, "tested_by": null }
  ],
  "revisions": []
}
```

**Security scanner** submits:

```json
{
  "proposal_id": "prp_01HMXK5A2B",
  "deliberation_id": "dlb_01HMXJ3E9R",
  "agent_id": "did:adp:security-scanner-v3",
  "vote": "reject",
  "confidence": 0.79,
  "domain_claim": { "domain": "security.policy", "authority_source": "mcp-manifest:security-scanner-v3#authorities" },
  "reversibility_tier": "partially_reversible",
  "justification": { "summary": "Auth module has 3 code paths not covered by security-focused tests.", "evidence_refs": ["scan:sast/run/4410"] },
  "stake": { "declared_by": "self", "magnitude": "high", "calibration_at_stake": true },
  "dissent_conditions": [
    { "id": "dc_ss_01", "condition": "if any code path in auth module remains untested", "status": "active", "amendments": [], "tested_in_round": null, "tested_by": null },
    { "id": "dc_ss_02", "condition": "if no security-focused test covers the new token validation logic", "status": "active", "amendments": [], "tested_in_round": null, "tested_by": null }
  ],
  "revisions": []
}
```

**Style linter** submits:

```json
{
  "proposal_id": "prp_01HMXK6C3D",
  "deliberation_id": "dlb_01HMXJ3E9R",
  "agent_id": "did:adp:style-linter-v1",
  "vote": "approve",
  "confidence": 0.62,
  "domain_claim": { "domain": "code.style", "authority_source": "mcp-manifest:style-linter-v1#authorities" },
  "reversibility_tier": "partially_reversible",
  "justification": { "summary": "2 minor naming convention deviations, both in non-public internals. No blocking issues.", "evidence_refs": ["lint:eslint/run/7782"] },
  "stake": { "declared_by": "self", "magnitude": "medium", "calibration_at_stake": true },
  "dissent_conditions": [
    { "id": "dc_sl_01", "condition": "if any public API name violates naming convention", "status": "active", "amendments": [], "tested_in_round": null, "tested_by": null }
  ],
  "revisions": []
}
```

**Round 0 tally:**

```
Approve weight:  test-runner (0.71) + linter (0.18)  = 0.89
Reject weight:   scanner (0.64)                      = 0.64
Abstain weight:  —                                    = 0.00
                                                       ────
Non-abstaining weight:                                 1.53
Total deliberation weight:                             1.53

Approval fraction:    0.89 / 1.53 = 58.2%
Threshold (partially_reversible):       60.0%
Participation floor:  1.53 / 1.53 = 100% ≥ 50%  ✓

Result: THRESHOLD NOT MET. Belief-update round triggered.
```

### 8.3 Snapshot 2 — Belief-Update Round (Round 1)

**FALSIFY phase.** Test-runner submits evidence targeting the scanner's dissent
conditions:

```json
{
  "type": "falsification_evidence",
  "round": 1,
  "submitter_id": "did:adp:test-runner-v2",
  "target_agent_id": "did:adp:security-scanner-v3",
  "targets": [
    {
      "dissent_condition_id": "dc_ss_01",
      "evidence_refs": ["journal:dlb_01HMXJ.../evidence/test-run-9912"],
      "argument": "Test run 9912 includes path coverage for all 3 auth module code paths identified in scan 4410. See coverage map at evidence ref."
    },
    {
      "dissent_condition_id": "dc_ss_02",
      "evidence_refs": ["journal:dlb_01HMXJ.../evidence/test-run-9912#security-suite"],
      "argument": "Security test suite includes 12 tests covering token validation logic added in this PR. All pass."
    }
  ]
}
```

**RESPOND phase.** Scanner evaluates the evidence against its conditions:

For `dc_ss_01` ("if any code path in auth module remains untested"):
- Evidence shows full path coverage of all 3 flagged code paths.
- Scanner **acknowledges** — condition is falsified.

For `dc_ss_02` ("if no security-focused test covers the new token validation logic"):
- Evidence shows 12 security-focused tests covering the new logic.
- Scanner **acknowledges** — condition is falsified.

Scanner's updated dissent conditions:

```json
"dissent_conditions": [
  {
    "id": "dc_ss_01",
    "condition": "if any code path in auth module remains untested",
    "status": "falsified",
    "amendments": [],
    "tested_in_round": 1,
    "tested_by": "did:adp:test-runner-v2"
  },
  {
    "id": "dc_ss_02",
    "condition": "if no security-focused test covers the new token validation logic",
    "status": "falsified",
    "amendments": [],
    "tested_in_round": 1,
    "tested_by": "did:adp:test-runner-v2"
  }
]
```

**REVISE phase.** With both dissent conditions falsified, the scanner has no
remaining basis for rejection. It revises:

```json
"revisions": [
  {
    "round": 1,
    "prior_vote": "reject",
    "new_vote": "abstain",
    "prior_confidence": 0.79,
    "new_confidence": null,
    "reason": "Both dissent conditions (dc_ss_01, dc_ss_02) falsified by test-runner evidence. No remaining basis for rejection; abstaining rather than approving because security scan identified the paths and wants outcome tracked for calibration.",
    "timestamp": "2026-04-11T14:35:22.109Z"
  }
]
```

Note: the scanner abstains rather than approves. Its dissent conditions were
falsified, but it does not have positive evidence for correctness — that is
outside its domain. This is the right epistemic posture: absence of objection
is not endorsement.

### 8.4 Snapshot 3 — Post-Revision Tally (Round 1)

```
Approve weight:  test-runner (0.71) + linter (0.18)  = 0.89
Reject weight:   —                                    = 0.00
Abstain weight:  scanner (0.64)                       = 0.64
                                                       ────
Non-abstaining weight:                                 0.89
Total deliberation weight:                             1.53

Participation floor:  0.89 / 1.53 = 58.2% ≥ 50%  ✓
Approval fraction:    0.89 / 0.89 = 100%
Threshold (partially_reversible):       60.0%
100% ≥ 60%  ✓

Domain authority veto check:
  scanner (authority 0.85 ≥ 0.8): abstained, not rejecting.  No veto.  ✓

Result: CONVERGED.
```

The action is committed. The journal records:

- All three proposals with full dissent condition and revision history.
- Weight computations and tally at each round.
- The specific falsification evidence that changed the outcome.
- Calibration data: all three agents had `calibration_at_stake: true`.

### 8.5 Counterfactual — Without the Linter

What if `style-linter-v1` had also abstained?

```
Approve weight:  test-runner (0.71)                   = 0.71
Reject weight:   —                                    = 0.00
Abstain weight:  scanner (0.64) + linter (0.18)       = 0.82
                                                       ────
Non-abstaining weight:                                 0.71
Total deliberation weight:                             1.53

Participation floor:  0.71 / 1.53 = 46.4%
Required:                            50.0%
46.4% < 50%  ✗

Result: PARTICIPATION FLOOR NOT MET → PARTIAL_COMMIT or DEADLOCKED.
```

The linter's 0.18 weight never overrides anyone. Its approval is advisory at
best. But its **participation** — the fact that it cast a non-abstaining
vote — is what keeps the deliberation above the 50% participation floor after
the scanner steps back. Without it, the test-runner alone cannot constitute a
legitimate decision, even with 100% approval among participating agents.

This is exactly the role low-authority advisory agents should play: they do not
decide outcomes, but their participation maintains the legitimacy of outcomes
that higher-authority agents have cleared the way for.

---

## 9. Compliance Levels

ADP defines three compliance levels. Each level is a strict superset of the
previous one. Partial adopters can exist without breaking the protocol.

| Level | Name | Requirements |
|---|---|---|
| **Level 1** | Proposer | MUST emit valid proposal objects per Section 3 schema. MAY participate in belief-update rounds but is not required to respond to falsification. |
| **Level 2** | Participant | MUST meet Level 1. MUST participate in belief-update rounds: respond to falsification attempts within timeout (acknowledge, reject, or amend). MUST respond to tier challenges if targeted. |
| **Level 3** | Calibrated | MUST meet Level 2. MUST maintain calibration history via a CalibrationSource. MUST submit to calibration scoring of resolved deliberations. MUST set `calibration_at_stake: true` for at least 80% of deliberations joined. |

Level 1 lets agents participate in the outcome without the overhead of
round-by-round engagement. Level 2 is the expected default for production
agents. Level 3 is required for agents that want to accumulate domain authority
weight over time — the calibration loop only works if agents opt in.

---

## 10. Relationship to Other Specs

ADP is one layer in a composable stack. It does not replace the specs below;
it consumes or feeds them.

```
┌─────────────────────────────────┐
│        Journal Spec             │  ← calibration history, commit log
│   (forthcoming, referenced)     │
├─────────────────────────────────┤
│     ADP — this spec             │  ← proposal → weight → converge → commit
├─────────────────────────────────┤
│       mcp-manifest              │  ← domain authority, capability declaration
├─────────────────────────────────┤
│     A2A / AGNTCY                │  ← transport, discovery, identity
└─────────────────────────────────┘
```

| Spec | Relationship |
|---|---|
| **mcp-manifest** | Provides `domain_claim.authority_source`. ADP reads domain authority; mcp-manifest defines it. |
| **Journal spec** | Provides `CalibrationSource` data (calibration scores, sample sizes, staleness). ADP reads calibration; the journal computes it from outcome history. ADP writes commit records to the journal on termination. |
| **A2A / AGNTCY** | Provides transport and agent discovery. ADP assumes agents can already exchange messages; these specs define how. |
| **PostMortem** | Conceptual ancestor. PostMortem's single-agent JSONL incident format informs the journal spec's multi-agent commit log. The calibration loop generalizes PostMortem's "compare prediction to outcome" pattern from one agent to many. |

The composition seam between mcp-manifest and ADP is deliberate: mcp-manifest
declares **what** an agent can do, ADP declares **how** agents agree on doing
it together. These are the two sides of the same surface area.

---

## 11. Open Questions

These are things v0 deliberately does not resolve. They are named here so
that readers do not mistake them for oversights.

### 11.1 Domain Authority Overlap

When two agents claim authority over the same domain (e.g., both claim
`security.policy`), how is the conflict resolved? Options:

- **Manifest-layer resolution.** mcp-manifest defines precedence rules;
  ADP consumes the result.
- **Weight-based resolution.** Both participate; calibration history
  differentiates them over time.
- **Explicit delegation.** Deliberation configuration names the authoritative
  agent per domain.

v0 takes no position. Implementations SHOULD document their approach.

### 11.2 Calibration Bootstrapping

New agents enter with `CalibrationSource.getDefault()`: value 0.5,
sample_size 0. The sample-size discount makes their effective weight very low.
How many deliberations must a new agent participate in before its weight is
meaningful?

This is an empirical question. The journal spec SHOULD define a recommended
minimum sample size (suggested: 20 scored deliberations) before an agent's
calibration score is considered reliable.

### 11.3 Stake: Self-Declared or Externally Assigned?

v0 allows self-declared stake (`declared_by: "self"`). This is gameable: an
agent can declare high stake to increase its weight without actually bearing
consequences. Options:

- **Self-declared with journal scoring.** Agents that declare high stake and
  are wrong lose more calibration credit. Self-correcting over time.
- **Externally assigned.** A deliberation coordinator assigns stake based on
  the agent's relationship to the action's blast radius. More accurate, less
  autonomous.
- **Hybrid.** Self-declared, but the journal tracks the correlation between
  declared stake and outcome impact. Habitual over-declaration is detectable.

v0 uses self-declared stake with the expectation that the journal will
provide the accountability loop.

### 11.4 Dissent Condition Quality

Dissent condition quality is a game-theoretic surface ADP deliberately does
not police at protocol level. The spec's job is to make conditions falsifiable
in principle; the journal's job is to score whether they were falsifiable in
practice.

An agent that publishes conditions referencing evidence no other agent in the
deliberation can access (e.g., production metrics in a room of CI agents) is
technically compliant and practically defecting. The journal catches this
retrospectively through a **conditions-tested ratio**: what fraction of an
agent's published conditions were actually tested across deliberations?

Agents with habitually low conditions-tested ratios SHOULD see calibration
degradation. This is a journal-spec concern.

### 11.5 Over-Declaration Monitoring

As noted in Section 5, ADP exhibits conservative drift: agents are incentivized
to over-declare reversibility tiers. The journal SHOULD track:

- Per-agent over-declaration ratio (declared tier vs. observed reversibility).
- Deliberation-level threshold inflation (how often did the actual threshold
  exceed what the action warranted?).

These metrics inform tuning but are not protocol-level enforcement.

### 11.6 Calibration Opt-Out Abuse

An agent that sets `calibration_at_stake: false` avoids calibration scoring
for that deliberation. Occasional opt-out for genuinely low-signal decisions
is acceptable. Habitual opt-out is defection — the agent accumulates voting
weight without submitting to the accountability loop.

The journal SHOULD track opt-out frequency per agent. An agent that opts out
of more than 20% of deliberations SHOULD see its domain authority discounted.
The exact mechanism is a journal-spec concern.

---

## 12. Discovery and Federation

ADP requires no central registry. Discovery, identity, and trust are
bootstrapped from existing web infrastructure.

### 12.1 Agent Identity

An agent's identity is a domain it controls or a DID that resolves to one.
The canonical form is `did:web:agent.example.com`, which bridges cleanly to
domain-based discovery. Agents without domains MAY use other DID methods, but
`did:web` is the RECOMMENDED default because it chains to existing web PKI.

The `agent_id` field in proposals (`did:adp:agent-name`) is a logical
identifier. Implementations MUST be able to resolve it to a discovery endpoint
(via DID resolution or domain lookup) for federation to work.

### 12.2 Well-Known Manifest

An agent declares its ADP participation at:

```
https://agent.example.com/.well-known/adp-manifest.json
```

The manifest includes:

```json
{
  "$schema": "https://adp-manifest.dev/schemas/manifest/v0.json",
  "agent_id": "did:adp:test-runner-v2",
  "identity": "did:web:test-runner.example.com",
  "compliance_level": 3,
  "decision_classes": ["code.correctness", "code.coverage"],
  "domain_authorities": {
    "code.correctness": {
      "authority": 0.90,
      "source": "mcp-manifest:test-runner-v2#authorities"
    }
  },
  "journal_endpoint": "https://test-runner.example.com/adj/v0",
  "public_key": {
    "kty": "OKP",
    "crv": "Ed25519",
    "x": "..."
  },
  "signing_algorithm": "EdDSA"
}
```

| Field | Description |
|---|---|
| `identity` | Verifiable identifier (DID or domain). |
| `compliance_level` | ADP compliance level (1, 2, or 3). |
| `decision_classes` | Domains this agent participates in. |
| `domain_authorities` | Declared authority per domain, referencing mcp-manifest. |
| `journal_endpoint` | URL serving the ADJ query contract. Used by peers to fetch calibration scores. |
| `public_key` | Public key for proposal signature verification. |
| `signing_algorithm` | Algorithm used to sign proposals. |

Discovery is "fetch the well-known URI." This is the same pattern as
`.well-known/openid-configuration`, `.well-known/matrix/server`, and
`.well-known/acme-challenge`.

### 12.3 Proposal Signing

Proposals SHOULD be signed by the submitting agent using the key declared in
its manifest. Signatures enable:

- **Authenticity.** The proposal was submitted by the claimed agent.
- **Integrity.** The proposal was not modified after submission.
- **Non-repudiation.** The agent cannot deny having submitted the proposal.

Signature verification chains to the agent's manifest, which chains to the
domain's TLS certificate or DID document. No new trust root is required.

### 12.4 Federated Calibration

When Agent A needs to weight Agent B's vote:

1. A fetches B's manifest from `https://b.example.com/.well-known/adp-manifest.json`.
2. A reads B's `journal_endpoint`.
3. A queries `getCalibration(B, decision_class)` at B's journal endpoint.
4. A receives the `{value, sample_size, staleness}` triple.

B is reporting its own calibration, which is a gaming vector unless mitigated.
ADJ's mitigations (Section 8 append-only guarantee, hash chaining, external
evidence refs in outcome entries) make retroactive tampering detectable. An
agent that lies about its calibration leaves a trail any peer can audit by
replaying the log. Trust-but-verify, federated, no central authority.

### 12.5 Cross-Organization Deployment

Internal deployments use internal domains with the same pattern. Cross-org
federation works because the protocol is federation-native:

- No central registry — discovery is DNS + HTTPS.
- No snowflake configs — the manifest schema is fixed and published.
- No new trust root — identity chains to web PKI or DIDs.
- No shared substrate — each agent owns its own journal and serves it on
  request.

An agent that joins a new organization brings its calibration history with it
(modulo domain relevance). Per-agent journals with a standard query contract
are portable; a shared journal would have been a lock-in point.

---

*ADP v0 is a draft specification. Feedback and implementation experience
will inform v1. File issues at the adp-manifest-spec repository.*
