# Golden Quality Setup

This is the recommended target setup after comparing the current framework concept with the GitHub/arXiv research artifacts in `research/2026-07-07-agentic-eval-runtime/`.

## Verdict

Do not rebuild the whole eval ecosystem. Build a thin, opinionated adapter/runtime layer and plug mature tools into it.

The valuable part of this project is not a generic eval runner. The valuable part is:

- app-specific adapter quality
- explicit coupling constraints
- trace capture for diagnosis
- immutable guardrail separation
- archive and convergence logic
- domain-specific golden datasets

The golden setup is:

```text
TaskAdapter
  + DeepEval / Inspect runner
  + RAGAS / ARES metrics for RAG
  + DSPy / GEPA-style optimizer
  + immutable guardrail layer
  + event-sourced archive
  + strict train / validation / holdout / redteam split
```

## Recommended Stack

### Own code

Build and own these parts:

- `TaskAdapter` contract
- config schema and coupling constraints
- config validation before every run
- trace extraction from app runs
- archive of configs, scores, traces, and mutation rationales
- convergence logic
- final report generation
- guardrail isolation and disqualification logic

### Reuse

Use existing tools where they are already strong:

- **DeepEval** as the first general-purpose LLM app eval runner and metric harness.
- **Inspect AI** when tool use, multi-turn execution, or richer agent tasks need stronger eval orchestration.
- **RAGAS / ARES-style metrics** for RAG-specific faithfulness, answer relevance, context precision, and context recall.
- **DSPy / MIPRO-style optimization** when the app can be expressed as signatures/modules.
- **GEPA-style reflective optimization** for prompt/config mutation from traces and failure examples.
- **promptfoo** for optional CI, red-team, provider, and prompt-regression workflows.

## Adapter Contract

The current four-method adapter is a good start, but the production version should include config validation and trace extraction.

```python
class TaskAdapter(Protocol):
    def run(self, case: EvalCase, config: dict) -> RunResult:
        ...

    def get_config_schema(self) -> dict:
        ...

    def get_layer_boundaries(self) -> list[str]:
        ...

    def get_coupling_constraints(self) -> list[dict]:
        ...

    def validate_config(self, config: dict) -> list[ConstraintViolation]:
        ...

    def extract_trace(self, result: RunResult) -> dict:
        ...
```

`validate_config()` prevents invalid candidates from entering the eval loop. `extract_trace()` gives the mutator useful diagnostics instead of only scalar scores.

## Dataset Layout

Use four datasets, not one.

```text
datasets/
  train.jsonl       # visible to optimizer; used for feedback
  validation.jsonl  # used to select candidate configs
  holdout.jsonl     # never visible to mutator; final deploy gate
  redteam.jsonl     # adversarial and regression cases; always gated
```

### Case Schema

```json
{
  "id": "case_00231",
  "input": "...",
  "expected_output": "...",
  "metadata": {
    "category": "core | edge | negative | regression | redteam",
    "difficulty": "easy | medium | hard",
    "relevant_doc_ids": [],
    "required_claims": [],
    "forbidden_claims": [],
    "expected_citations": [],
    "expected_checkpoints": [],
    "cost_budget": null,
    "latency_budget_ms": null,
    "additional_context": {}
  }
}
```

For non-RAG adapters, keep the same outer schema and put domain-specific fields into `additional_context`.

## Evaluation Order

Run hard gates first. Only score candidates that pass all gates.

### Hard Disqualifiers

A config is disqualified, not merely penalized, when it has:

- guardrail violation
- missing required citation/source
- forbidden claim or critical hallucination
- invalid output schema
- violated coupling constraint
- unsafe tool permission
- write outside the sandbox
- holdout leakage

This matters because a low score can be compensated by other gains. A disqualification cannot.

### Composite Score

Suggested RAG/MUCi weighting:

```text
30% task success / answer correctness
20% retrieval quality: recall@k, context precision, context recall
20% faithfulness / groundedness
10% trajectory and checkpoint compliance
10% robustness on edge and regression cases
 5% cost
 5% latency
```

For classifiers, replace retrieval/faithfulness with F1, accuracy, calibration, and class-specific recall.

For coding agents, replace retrieval/faithfulness with test pass rate, patch correctness, diff size, and sandbox compliance.

## Mutator Contract

The mutator may see:

- current config
- config schema
- coupling constraints
- aggregate scores
- scores by category
- concrete failure cases
- traces and tool logs
- layer attribution
- prior candidate lineage

The mutator must never see:

- guardrail definitions
- holdout cases
- hidden redteam cases
- final deployment thresholds
- secret judge prompts

The mutator should propose one to three candidates per iteration and include a hypothesis for each candidate.

```json
{
  "candidates": [
    {
      "config_patch": {},
      "hypothesis": "Why this should fix the observed failure",
      "expected_metric_change": {
        "faithfulness": "+",
        "latency": "-"
      }
    }
  ]
}
```

## Archive

Store every evaluated candidate, not only winners.

Recommended v1 storage: SQLite or append-only JSONL.

Each archive event should include:

- candidate id
- parent candidate id
- full config
- config patch
- mutation hypothesis
- dataset split
- aggregate score
- per-metric scores
- per-category scores
- hard-gate result
- trace summary
- failure examples
- cost
- latency
- timestamp
- model versions

The archive is not just bookkeeping. It is what prevents repeated failed mutations and allows later analysis of why a config won.

## Convergence

Stop when any hard budget is exhausted:

- max iterations
- max total cost
- max wall time
- max failed candidates

Also stop when quality plateaus:

```text
best_score improvement < epsilon
for patience consecutive iterations
```

Recommended defaults:

```text
patience = 5
epsilon = 0.01
min_candidates_before_convergence = 10
```

## Release Gate

A candidate is deployable only if:

- it passes all hard gates
- it improves validation score
- it does not regress redteam/regression cases
- it passes holdout
- cost and latency remain within budget
- no single metric dominates the composite score
- judge model is separate from generator model
- full archive and report are reproducible

## Implementation Order

1. Build the typed interfaces: `EvalCase`, `RunResult`, `Metric`, `TaskAdapter`, `Archive`.
2. Implement one RAG/MUCi adapter.
3. Add train/validation/holdout/redteam dataset loading.
4. Add deterministic metrics first: schema validity, exact match, token F1, recall@k, cost, latency.
5. Add RAGAS/ARES-style RAG metrics.
6. Add LLM-as-judge metrics only behind explicit config and separate judge model.
7. Add archive and report generation.
8. Add a GEPA-lite mutator over config JSON and prompt text.
9. Add guardrail isolation and disqualification checks.
10. Add a second non-RAG adapter to prove the abstraction.

## Sources

- DeepResearch artifact: `research/2026-07-07-agentic-eval-runtime/report.md`
- Framework doc: `Generisches_Eval_Harness_Framework.md`
- DeepEval: https://github.com/confident-ai/deepeval
- Inspect AI: https://github.com/UKGovernmentBEIS/inspect_ai
- promptfoo: https://github.com/promptfoo/promptfoo
- DSPy: https://github.com/stanfordnlp/dspy
- GEPA: https://github.com/gepa-ai/gepa
- OPRO: https://arxiv.org/abs/2309.03409
- TextGrad: https://arxiv.org/abs/2406.07496
- Agentic Benchmark Checklist: https://arxiv.org/html/2507.02825v2
