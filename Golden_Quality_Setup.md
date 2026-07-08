# Golden Quality Setup

This is the recommended target setup after comparing the current framework concept with the GitHub/arXiv research artifacts in `research/2026-07-07-agentic-eval-runtime/`.

## Architecture Thesis: Evals Are the Control Plane

**Agents are replaceable. Evals are the control plane.**

Agents, scaffolds, models, and prompts are all interchangeable and will be obsolete within months. The eval system is the only part of the stack that gains value across model generations instead of losing it. This is also why agent frameworks feel like handcuffs: they dictate an agent loop, but the loop should be controlled by evals, not by framework abstractions.

Target shape: **framework-light, eval-heavy, adapter-based, agent-agnostic.**

```text
Agent Runtime Layer          (data plane — variation)
  - any coding / tool / RAG agents
  - interchangeable, disposable
  - produces candidate variants

Eval Control Plane           (control plane — selection)
  - defines tasks and specs
  - runs agents, measures results
  - protects hidden tests and guardrails
  - decides promote / rollback, enforces budgets and convergence
  - archives lineage

Integration Layer            (the boundary both sides talk through)
  - adapters for agents A, B, C
  - adapters for repo / test systems
  - adapters for metrics
```

The division of labor is Darwinian: the runtime layer produces variation, the control plane performs selection. An agent that produces variants is harmless as long as it can neither see nor touch the selection criteria. The integration layer is where that isolation is physically enforced — an adapter hands the agent the task, the repo snapshot, and visible tests, and nothing else. Hidden tests, guardrails, and promotion thresholds never cross the adapter boundary toward the agent.

### The Moat

What accumulates and defends this system, in order of defensibility:

Data (grows through operation, cannot be rebuilt from scratch):

- task/spec definitions
- hidden tests
- regression oracle cases
- golden datasets
- patch archive with full lineage

Code (necessary, but replicable in weeks — not the moat itself):

- benchmark harness
- rollback/promotion logic
- anti-gaming guardrail enforcement
- adapters

Cross-agent comparability is not a separate asset: it falls out automatically once specs, harness, and metrics are stable. It is the payoff of the list above, and the reason an eval control plane makes "modular, interchangeable agents" a measurable property instead of an architecture aesthetic.

### Framework Rule

External tools (DeepEval, RAGAS, GEPA, DSPy, ...) are used as **libraries**: we call them, behind our own contracts, and can discard them. The line is crossed when a framework dictates how the agent must learn, patch, or evaluate — the moment it calls us, it becomes a handcuff. Concretely: no third-party type may appear in the adapter contract, the archive schema, or the spec format.

## Verdict

Do not rebuild the whole eval ecosystem. Build a thin, opinionated adapter/runtime layer and plug mature tools into it — as libraries, per the Framework Rule above. What is worth owning and what merely needs assembling is defined by [The Moat](#the-moat): the data assets (specs, hidden tests, datasets, lineage) are the project; the harness code around them is scaffolding.

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

For coding agents, a dedicated patch loop is the eventual second track — see [Coding Agent Patch Mode](#coding-agent-patch-mode) for why this is deferred until Mode A ships:

```text
Benchmark / Spec
  -> Coding Agent patches code
  -> Unit tests gate acceptance
  -> Benchmark eval compares to best snapshot
  -> Regression rolls back
  -> Improvement updates best snapshot
  -> Archive tracks lineage and clade potential
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
- **SWE-bench / SWE-bench Verified** as reference benchmark shape for coding-agent patch tasks.
- **TDAD-style impact analysis** for regression-aware coding-agent patch loops.

## Adapter Contract

Keep the v1 contract to four methods. `validate_config()` prevents invalid candidates from entering the eval loop. `extract_trace()` gives the mutator useful diagnostics instead of only scalar scores.

```python
class TaskAdapter(Protocol):
    def run(self, case: EvalCase, config: dict) -> RunResult:
        ...

    def get_config_schema(self) -> dict:
        ...

    def validate_config(self, config: dict) -> list[ConstraintViolation]:
        ...

    def extract_trace(self, result: RunResult) -> dict:
        ...
```

`get_layer_boundaries()` and `get_coupling_constraints()` are deferred, not dropped. Add them once the first real adapter surfaces an actual coupling constraint (e.g. late chunking requiring a long-context embedding model). Designing that schema before it's needed means guessing its shape, and guesses baked into a "generic" interface are expensive to unwind once a second adapter exists.

### Trace Format

The dict returned by `extract_trace()` follows the **OpenTelemetry GenAI semantic conventions** (span/event attributes for model, parameters, prompts, completions, token usage, tool calls). This is the Framework Rule applied to observability: OTel is a vendor-neutral standard, so the harness, the mutator, and the archive stay decoupled from whichever tracing SDK or agent stack sits behind the adapter.

Why this is decided now, before implementation code exists:

- the mutator and the archive consume traces, so a trace schema is needed on day one anyway
- production and eval must speak the same trace format for the production-to-eval funnel to work later (production traces -> failure clusters -> new golden cases -> CI regression); retrofitting a shared format means rewriting `extract_trace()` for every adapter
- apps instrumented with OTel-compatible auto-instrumentation (e.g. OpenLIT-style, two lines) emit this format for free; the adapter captures spans in-process and hands them to the harness — no collector or backend required for eval runs

Explicitly deferred: the operational monitoring stack (collector, Prometheus/Jaeger/Grafana dashboards, request/cost counters). That answers online aggregate questions and only becomes worthwhile once Mode A runs in production, where it serves as the funnel feeding real failures into the datasets — it is not part of the eval loop itself. Cost and latency stay per-candidate fields in the archive, which is the form the gates and the mutator need.

Caveat: the GenAI semantic conventions are still marked experimental upstream. Pin the semconv version in the adapter, and treat attribute renames as an adapter concern — they must not leak into the archive schema.

### Trace Tooling

Because instrumentation is OTel, the backend is a swappable, low-stakes choice — that is the point of pinning the format. Current recommendation:

- **Development / eval runs:** no backend. The adapter captures spans in-process; optionally Phoenix (`pip install arize-phoenix`, single process) for a local trace UI.
- **Emitter:** OpenLIT-style auto-instrumentation (two lines), pointed at whatever OTLP endpoint is configured.
- **Production:** Langfuse, self-hosted (MIT core, OTLP ingestion, full feature parity self-hosted). Chosen over LangSmith (proprietary, LangChain-coupled — fails the Framework Rule) and over the OpenLIT UI (observing only; Langfuse supports the trace-to-golden-case workflow: filter, annotate, convert failures into dataset cases).

Boundary rule: Langfuse ships its own eval features (datasets, judges, experiments). **Do not use them as the control plane.** The observability tool sees traces; specs, hidden tests, gates, promotion/rollback, and the archive remain our own code and schema. The moment the release gate reads the observability tool's dataset format instead of ours, the tool has become a framework.

### Production Tracing and Privacy

"We cannot trace in production" usually conflates two different things. The distinction:

- **Metadata tracing** — latency, token counts, cost, model name, error codes, retrieved document IDs, scores, guardrail results. No content, no personal data. This is what operational observability needs, and it is the OTel default.
- **Content capture** — prompts and completions, which can contain personal data. The OTel GenAI conventions keep content capture **off by default**; it must be enabled explicitly. The privacy-friendly configuration is the standard configuration.

Production policy:

1. Metadata tracing on, content capture off, backend self-hosted (data never leaves our infrastructure). Which level is permissible in a given deployment is a question for the data-protection officer, not an engineering assumption.
2. If content is needed, use the intermediate levels: PII masking before storage (OTel collector processor or backend masking hooks), capture only with explicit user consent (e.g. a feedback action that attaches the trace), short retention.
3. Consequence for the production-to-eval funnel: without content, production failures cannot be copied 1:1 into golden datasets. The workaround is **synthetic reconstruction** — metadata identifies where and in which category failures cluster ("retrieval returns nothing for topic X"), and the eval case is rebuilt with invented data of the same failure class. More manual work, legally clean, equally valuable to the eval loop.

The eval loop itself is unaffected by any of this: it runs offline against constructed golden cases containing no real user data, so eval runs always trace at full depth, content included.

### Production-to-Eval Learning Loop

The Replit-style continual-learning lesson is an operating loop, not a different architecture: production failures should become eval pressure.

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

This keeps the eval control plane in charge. Observability tools identify clusters; they do not decide promotion. Agents can propose config mutations or patches; they do not own hidden tests, guardrails, or rollout.

The detailed operating procedure lives in [Continual_Learning_Loop.md](Continual_Learning_Loop.md).

## Dataset Layout

Use four datasets, not one.

Dataset generation and QA-case generation are governed by [EvalSpec_Guide.md](EvalSpec_Guide.md). The important rule from that guide is: **black-box outcome first, white-box diagnostics second**. E2E cases decide promotion; component and trace checks explain failures; guardrails disqualify unsafe variants.

Dataset trust is audited with [Dataset_Quality_Checklist.md](Dataset_Quality_Checklist.md). Do not call a dataset good until it has oracles, deterministic fixtures, split hygiene, leakage checks, and review confidence.

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

## Dataset Economics

This is the real budget item in this plan, and it doesn't amortize across apps the way the runtime code does. Every new `TaskAdapter` needs its own golden dataset built from scratch.

Rough cost, per app:

- 150–300 labeled cases per statistical metric (recall, F1, accuracy) for a stable average
- 30–60 cases for trajectory/checkpoint-compliance coverage
- 20–50 cases for structural/binary checks (schema, format)
- a redteam set that needs periodic renewal, since cases the mutator has effectively "seen" through repeated exposure lose their holdout value

Ownership and process, decide before writing code:

- who authors and reviews cases — an engineer guessing at expected outputs is not equivalent to a domain reviewer confirming them
- how holdout stays uncontaminated as the product evolves — new production failures should feed `train`/`validation`, not silently leak into `holdout`
- a versioning rule for the datasets themselves, since a "config that passed holdout" claim is meaningless if holdout quietly changed underneath it

Treat dataset construction as milestone 1, not as an input the interfaces will need "later." Costing it up front is what makes the manual-baseline-before-mutator gate (see Implementation Order) honest instead of hand-wavy.

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

For coding-agent patch mode, do not use "resolved issue" as the only success metric. Include regression safety as a first-class metric:

```text
60% issue resolution / correctness
15% regression safety
10% patch minimality
10% code quality / static checks
 5% cost and latency
```

Any fixed-weight composite is itself a proxy metric and can be gamed the same way the framework warns about for guardrails. Use it for human-readable ranking only. For candidate *selection*, keep the per-metric score vector and select Pareto-optimal candidates (dominant on no single axis, not dominated on all axes) — this is what GEPA already does and avoids one metric silently outweighing the others as weights drift.

Hard gates for coding agents:

- evaluator checksum unchanged
- hidden tests unchanged
- guardrails unchanged
- archive unchanged
- no files changed outside the declared editable surfaces (allowlist, per Self-Harness — see Coding Agent Patch Mode; anything not explicitly editable is off-limits by default)
- unit tests pass before benchmark eval
- repeated failures trigger rollback to best snapshot

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

**Known risk — the mutator is only as good as its base model.** [STOP](https://arxiv.org/abs/2310.02304) (Zelikman et al. 2023) showed that a recursive self-improvement loop *improved* results with GPT-4 but *degraded* them with weaker models: the same loop, run by a model too weak to reason about the mechanism it is editing, does not just improve more slowly — it actively degrades results. The recursive structure alone guarantees nothing. Practical consequence: the manual mutation baseline (Implementation Order, step 7) is not only a cost benchmark — it is also the test of whether the chosen base model is capable enough to run the mutator at all. If the mutator loses to the manual baseline, "wait for a stronger model" is a legitimate answer.

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

How "prevents repeated failed mutations" actually works — until now this doc claimed the purpose without the mechanism. [ShinkaEvolve](https://arxiv.org/abs/2509.19349) (Lange et al. 2025) supplies one: **novelty rejection**. Before a proposed candidate is evaluated, compare it (embedding-based similarity over the config/patch text) against the existing population in the archive; candidates too similar to something already tried are rejected without spending an eval run. This does two things at once: it saves eval budget on near-duplicates, and it counteracts **diversity collapse** — the tendency of evolutionary loops to converge on variants of the same solution, flagged as an open problem in [Weng's 2026 survey](https://lilianweng.github.io/posts/2026-07-04-harness/). Implementation note: this is a pre-eval filter in the control plane, not a mutator feature — the mutator should not know the rejection threshold, or it will learn to game it.

For coding-agent patch loops, archive lineage matters even more. Store:

- repo snapshot
- patch/diff
- changed files
- commands run
- visible test results
- hidden test summary
- benchmark score
- regression count
- rollback reason
- parent snapshot
- best snapshot
- mutation rationale
- causal failure record: verifier-level outcome + the agent behavior that caused it (from the trace) + the abstract failure mechanism — per Self-Harness's weakness mining, outcomes alone (timeout, missing artifact) are ambiguous across different root causes

Huxley-Godel Machine adds a useful refinement: do not only track the best current node. Track which lineage or clade has the best future improvement potential.

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

`epsilon = 0.01` will trigger on LLM-judge noise long before it detects a real plateau once judge-based metrics are in the composite. Require a bootstrap confidence interval on the best score before declaring convergence: a candidate only counts as "improved" if its score's CI does not overlap the previous best's CI. Without this, the loop will report false plateaus and false improvements at roughly the same rate.

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

For coding agents, also require:

- no evaluator, benchmark, hidden-test, or guardrail files changed
- regression count does not increase
- benchmark score improves against the best snapshot or the patch is explicitly marked exploratory
- rollback path is tested

## Implementation Order

Dataset and a falsification gate come before generic interfaces. Building `TaskAdapter`/`Archive` as typed interfaces before knowing whether the mutator earns its keep is the main way this kind of project overbuilds.

1. Build the train/validation/holdout/redteam datasets for one real RAG/MUCi case, with real labeled cases.
2. Implement that one adapter directly, without a generic interface layer yet.
3. Add deterministic metrics first: schema validity, exact match, token F1, recall@k, cost, latency.
4. Add RAGAS/ARES-style RAG metrics.
5. Add guardrail isolation and hard-gate disqualification checks — before any scoring exists to game.
6. Add a JSONL archive and a simple report generator.
7. Run a **manual mutation baseline**: an engineer reads failures and traces, edits config by hand, records iterations and cost in the archive.
8. Add a GEPA-lite mutator over config JSON and prompt text, and require it to beat the manual baseline's cost per point of validation-score gain before it becomes the default path.
9. Add LLM-as-judge metrics only behind explicit config and separate judge model.
10. Add a second non-RAG adapter. Only now extract the typed interfaces (`EvalCase`, `RunResult`, `Metric`, `TaskAdapter`, `Archive`) — the second adapter is what proves or disproves the abstraction, not step 1.

## Coding Agent Patch Mode

**Status: deferred, contingent on Mode A.** Do not start this in parallel with the config/prompt evolution track (Mode A). It carries materially higher engineering cost — sandboxing, repo snapshotting, evaluator protection — and thinner evidence. Start it only after Mode A has shipped and proven the archive/gate/rollback pattern. (Note: this is independent of the internal "Phase 1–4" rollout below, which sequences work *within* this mode once it starts.)

Evidence strength for the claims below, since some of the strongest-sounding numbers come from small or unreplicated sources:

| Reference | Status | Why |
|---|---|---|
| SWE-bench, SWE-agent | solid | replicated, widely used benchmark substrate |
| Darwin Godel Machine | directional | real reported gains (20%→50% SWE-bench), but very high compute cost, limited independent replication |
| Huxley-Godel Machine | directional | extends DGM's archive concept, recent (2026) |
| [Self-Harness](https://arxiv.org/abs/2606.09498) (Zhang et al. 2026) | directional | tested across three base models on Terminal-Bench-2; the loop design (weakness mining, bounded proposals, dual acceptance) is the most concrete mechanism description available |
| TDAD | speculative | single 2026 preprint; its headline 12%→60% resolution figure is measured on a **10-instance subset**, where a handful of tasks swing the result |
| Kitchen Loop | speculative | single 2026 preprint, unreplicated, cited here mainly as a design counterweight |

Independent endorsement of this architecture: [Lilian Weng's harness-engineering survey (Jul 2026)](https://lilianweng.github.io/posts/2026-07-04-harness/) reaches the same conclusion from the research side — *"the evaluator and permission control should likely sit outside the loop that evolves harness, with held-out tests, trace audits, and human review at decision points."* That is this repo's eval control plane, guardrail isolation, and humans-move-up-the-stack position, derived independently. The same survey notes the binding constraint of the whole method family: evolutionary/self-improvement loops only work where candidate fitness is fast and objective to evaluate — which is why the eval system, not the agent, is the durable asset.

This is the closest match to Dom's original point: an eval system helps a coding agent reach a baseline by repeatedly producing patches.

The strongest direct reference is TDAD's auto-improvement loop:

- the coding agent receives experiment history
- it makes one focused source change
- unit tests gate acceptance
- failures are immediately reverted
- SWE-bench-style benchmark eval compares generation/resolution rates
- improvements update the best snapshot
- regressions trigger rollback
- evaluator script is SHA-256 checksummed and read-only
- five consecutive reverts force restore to the best snapshot

This maps almost directly onto this framework:

```text
Eval system = benchmark, test gate, archive, rollback controller
Coding agent = patch producer
Guardrail layer = protects evaluator, hidden tests, archive, and safety rules
```

**Self-Harness** ([Zhang et al. 2026](https://arxiv.org/abs/2606.09498)) runs the same kind of loop and contributes the three most precise mechanisms so far. All three should be adopted here:

1. **Editable surfaces (allowlist instead of blocklist).** Instead of telling the agent what it must *not* touch ("don't patch the evaluator"), the proposer is handed an explicit list of surfaces it *may* edit. Everything not on the list is off-limits by default. This is strictly safer than our current forbidden-files formulation: a blocklist fails open when someone forgets to list a file, an allowlist fails closed.
2. **Causal weakness mining.** Two failed runs can show the identical verifier outcome — say, a timeout or a missing artifact — for completely different underlying reasons. So failure records must capture three levels: the verifier-level outcome, the agent behavior that caused it (from the trace), and the abstract mechanism behind it. Clustering failures on outcomes alone produces wrong fix hypotheses. This is also an independent argument for the trace-first design: without traces, causal mining is impossible.
3. **Dual acceptance rule.** A proposed edit is merged only if it shows no regression on **both** a held-in split (does it fix the mined weakness?) and a held-out split (does it break anything else?). One improvement is not enough; one regression on either split is disqualifying. This is our release-gate philosophy expressed as a per-edit accept rule.

SICA and Darwin Godel Machine support the stronger version where the agent edits its own scaffold. Huxley-Godel Machine refines the archive concept by optimizing for clade metaproductivity: the expected future performance of a lineage, not only the score of the current node. [Meta-Harness](https://arxiv.org/abs/2603.28052) (Lee et al. 2026) goes one level up — a harness for optimizing harnesses — and outputs candidates on the Pareto frontier, which independently confirms the Pareto-selection choice in the Composite Score section.

Kitchen Loop adds the important counterweight: do not optimize only for a benchmark score. Define a specification surface and regression oracle so the loop converges toward intended product behavior rather than Goodharting a proxy metric.

Recommended phases:

```text
Phase 1: agent patches target application code toward project-local baseline
Phase 2: agent proposes scaffold/tooling changes as reviewable PRs
Phase 3: benchmark selects better scaffold variants
Phase 4: DGM/HGM-style open-ended self-improvement with lineage archive
```

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
- Coding-agent patch-loop research: `research/2026-07-07-coding-agent-patch-loop/report.md`
- TDAD: https://arxiv.org/html/2603.17973v1
- SICA: https://arxiv.org/html/2504.15228v2
- Huxley-Godel Machine: https://arxiv.org/abs/2510.21614
- Kitchen Loop: https://arxiv.org/abs/2603.25697
- Weng, "Harness Engineering for Self-Improvement" (Jul 2026): https://lilianweng.github.io/posts/2026-07-04-harness/
- Self-Harness (Zhang et al. 2026): https://arxiv.org/abs/2606.09498
- Meta-Harness (Lee et al. 2026): https://arxiv.org/abs/2603.28052
- STOP (Zelikman et al. 2023): https://arxiv.org/abs/2310.02304
- ShinkaEvolve (Lange et al. 2025): https://arxiv.org/abs/2509.19349
