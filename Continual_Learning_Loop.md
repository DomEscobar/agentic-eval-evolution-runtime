# Continual Learning Loop

This note captures the missing operational loop from the Replit-style "continual learning for agents" framing.

The important idea is not that the model updates its weights. The important idea is that the **system learns from production trajectories** by turning recurring failures into evaluated fixes.

```text
Production traces / logs
  -> failure clusters
  -> human-approved hypothesis
  -> new regression case or EvalSpec update
  -> candidate patch or config mutation
  -> validation + regression + holdout gates
  -> human review
  -> limited launch
  -> archive lineage
```

This is the practical bridge between observability and self-improvement. Tracing alone is passive. Evals alone are static. The learning loop is the procedure that converts observed failures into durable selection pressure.

## What We Already Covered

The existing architecture already includes most of the necessary pieces:

- OTel-compatible trace extraction
- production privacy policy
- eval archive and lineage
- generated QA / golden datasets
- split hygiene with train, validation, holdout, and redteam sets
- guardrail isolation
- coding-agent patch mode
- promotion and rollback gates

So the repo does not need a different architecture. It needs an explicit operating loop that says how production evidence becomes new eval pressure.

## What This Adds

### 1. Failure Clustering

Production telemetry should be grouped by failure class, not handled as isolated anecdotes.

Useful clusters:

- repeated tool-call failures
- environment setup failures
- retrieval misses for the same topic
- planner loops
- invalid final-answer formats
- high-latency paths
- guardrail near-misses
- user corrections for the same behavior

Cluster records should preserve privacy. Prefer metadata and synthetic reconstruction over copying real user content into datasets.

```yaml
cluster:
  id: cold_start_env_setup_failure
  source: production_traces
  first_seen: 2026-07-08
  frequency_7d: 31
  severity: medium
  affected_tasks:
    - coding_agent_bootstrap
  evidence:
    trace_ids:
      - trc_...
    content_policy: metadata_only
  suspected_failure_mode: agent assumes dependency exists before checking runtime
```

### 2. Hypothesis Selection

Not every cluster deserves optimization budget. A human or reviewer agent should choose which clusters are worth converting into eval work.

Selection criteria:

- frequency
- severity
- user impact
- regression risk
- estimated fix cost
- whether the behavior is covered by existing evals
- whether the oracle can be made deterministic

This is where human taste still matters. The loop can surface candidates, but humans decide which failures are worth teaching the system.

### 3. Eval Case Creation

Every accepted cluster should produce one of:

- a new train case if the optimizer should learn from it
- a validation case if it should guide candidate selection
- a holdout case if it represents a release gate
- a redteam case if it is adversarial or safety-sensitive
- an EvalSpec update if the oracle or invariant was missing

Default rule:

```text
production cluster -> train/validation first
confirmed recurring regression -> holdout/redteam only after review
```

Do not put raw production content directly into holdout. Reconstruct a synthetic case with the same failure mode, deterministic fixtures, and a clear oracle.

### 4. Patch or Mutation

The system can respond in two ways:

- Mode A: mutate prompt, config, routing, retrieval settings, or tool policy
- Mode B: ask a coding agent to produce a code patch

Both must pass through the same gates. The source of the candidate does not matter. The eval control plane remains the judge.

```text
cluster hypothesis
  -> candidate change
  -> visible regression tests
  -> validation score
  -> holdout gate
  -> redteam gate
  -> archive event
```

### 5. Evidence Bundle for Review

Before launch, the loop should produce a compact evidence bundle:

- cluster summary
- selected hypothesis
- changed files or config diff
- new or updated eval cases
- before/after scores
- regression results
- guardrail results
- known remaining risk
- rollout recommendation

Humans should review evidence, not raw agent enthusiasm.

### 6. Launch and Archive

A promoted change should be launched with blast-radius control:

- canary first
- rollback pointer recorded
- post-launch trace watch
- archive event links production cluster to patch/config lineage

The archive should answer:

- which production failure triggered this change
- which hypothesis was tested
- which eval cases were added
- which candidate won
- why it was safe to ship
- whether the cluster disappeared after launch

## Required Archive Event

Add this event type to the future archive schema:

```json
{
  "event_type": "production_cluster_remediation",
  "cluster_id": "cold_start_env_setup_failure",
  "hypothesis_id": "hyp_2026_07_08_001",
  "candidate_id": "cand_042",
  "eval_case_ids_added": ["case_reg_184", "case_val_092"],
  "evidence_bundle_path": "reports/remediations/cold_start_env_setup_failure.md",
  "review_status": "approved",
  "rollout": {
    "strategy": "canary",
    "rollback_candidate_id": "cand_031"
  }
}
```

## Agent Workflow

When an agent is asked to improve the system from production evidence:

1. Read the relevant traces, logs, support notes, or failure reports.
2. Cluster failures by root-cause hypothesis, not by superficial error string only.
3. Rank clusters by severity, frequency, fixability, and evalability.
4. Choose a small number of clusters for remediation.
5. Create or update EvalSpec cases with deterministic oracles.
6. Apply a candidate config mutation or code patch.
7. Run visible tests, validation, holdout, redteam, and guardrail gates as appropriate.
8. Produce an evidence bundle.
9. Require human review for launch.
10. Archive the cluster, hypothesis, candidate, eval cases, result, and rollout.

## Non-Goals

This loop is not:

- online weight training
- automatic production launch without review
- feeding raw private production content into benchmarks
- optimizing directly against observability dashboards
- letting the coding agent modify hidden tests or release gates

## Bottom Line

The Replit-style lesson is mostly an operating discipline:

```text
Observe production -> compress failures -> make them evaluable -> test fixes -> launch carefully -> remember lineage
```

That strengthens the existing eval-control-plane thesis. It does not replace it.
