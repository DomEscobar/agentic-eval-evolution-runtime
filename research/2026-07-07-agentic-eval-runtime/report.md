# DeepResearch Report: Agentic Eval Evolution Runtime

## Executive summary

Dom's repo idea sits in a real and active gap: production tools cover eval execution and metrics well, while research systems cover prompt/config optimization loops well, but few systems combine both into a generic, adapter-driven runtime that can evaluate, mutate, archive, and converge across RAG, classifiers, coding agents, and chat/agent systems.

The closest matches are:

- **Execution/eval substrate:** Inspect AI, promptfoo, DeepEval.
- **Self-improving modular pipelines:** DSPy.
- **Evolutionary/textual optimization loop:** GEPA, OPRO, TextGrad, AutoPDL, Artemis-style agent optimization.
- **Direct but immature comparable repos:** harness-evals and opendatahub-io/agent-eval-harness.

My strong take: do not build a generic eval framework from scratch in isolation. Build a thin, opinionated runtime around the local `TaskAdapter` idea, reuse existing concepts from Inspect/DeepEval/promptfoo for execution and metrics, and borrow GEPA/TextGrad-style diagnostic feedback for the mutator.

## Evidence-backed findings

### 1. The local architecture is directionally right

The local doc separates one app-specific interface from reusable parts: golden dataset schema, metric registry, eval harness, mutator, archive, convergence logic, and guardrails (`file:///root/agentic-eval-evolution-runtime/Generisches_Eval_Harness_Framework.md`). This matches the strongest arXiv framing: a unified agent-evaluation framework should standardize the execution/evaluation layer around agents while letting benchmark content and goals vary: https://arxiv.org/html/2602.03238v2

That means the main design object should be the **runtime contract**, not a benchmark. The four-method `TaskAdapter` is a good start, but I would extend it before coding v1.

Suggested adapter contract:

```python
class TaskAdapter(Protocol):
    def run(self, case: EvalCase, config: dict) -> RunResult: ...
    def get_config_schema(self) -> dict: ...
    def get_layer_boundaries(self) -> list[str]: ...
    def get_coupling_constraints(self) -> list[dict]: ...
    def get_trace(self, run_result: RunResult) -> dict: ...
    def validate_config(self, config: dict) -> list[ConstraintViolation]: ...
```

The extra `trace`/`validate_config` pieces are not nice-to-have. GEPA, TextGrad, and Artemis-style systems all point toward using rich feedback, not only scores.

### 2. Existing eval frameworks solve pieces, not the whole loop

**Promptfoo** is mature and production-oriented: it tests prompts/models, agents and RAGs, supports red-teaming, provider comparison, local evals, and CI/CD: https://github.com/promptfoo/promptfoo

**DeepEval** is the clearest reference for metric ergonomics. It is Pytest-like for LLM apps and includes metrics for agents, RAG, chatbots, hallucination, task completion, tool correctness, and LLM-as-judge patterns: https://github.com/confident-ai/deepeval

**Inspect AI** is the strongest execution-substrate reference. It covers prompt engineering, tool usage, multi-turn dialogue, model-graded evals, extensions, and a collection of prebuilt evals: https://github.com/UKGovernmentBEIS/inspect_ai

None of these is exactly the local idea. They mainly run/evaluate. The local proposal adds an optimizer loop: mutate config -> run harness -> score -> archive -> stop by convergence.

### 3. Optimization research strongly supports the mutator/archive design

**OPRO** establishes the basic LLM-as-optimizer loop: feed prior solutions and values into an LLM, generate candidates, evaluate, and repeat: https://arxiv.org/abs/2309.03409

**DSPy** abstracts LM pipelines as modular programs and compiles them to maximize a metric, which is close to the "generic harness + app adapter" philosophy when the app can be expressed as a DSPy program: https://arxiv.org/abs/2310.03714 and https://github.com/stanfordnlp/dspy

**TextGrad** argues for textual feedback as the optimization signal for compound AI systems, analogous to gradients: https://arxiv.org/abs/2406.07496

**GEPA** is the closest match to the envisioned mutator/archive runtime. It samples trajectories, reflects on failures, proposes prompt updates, and keeps a Pareto frontier: https://arxiv.org/abs/2507.19457. Its GitHub implementation already exposes `optimize_anything` for text artifacts including code, agent architectures, and configurations: https://github.com/gepa-ai/gepa

So the mutator should not receive only aggregate metric means. It should receive:

- aggregate scores
- per-case failures
- traces/tool logs
- layer attribution
- constraint violations
- prior candidate lineage
- holdout result deltas

### 4. Guardrails are central, not secondary

The Agentic Benchmark Checklist paper reports that bad benchmark setup or reward design can over/underestimate results by up to 100% relative terms, and proposes checks around task validity, outcome validity, and reporting: https://arxiv.org/html/2507.02825v2

For this runtime, guardrails should be v1:

- immutable golden and holdout datasets
- separate train/dev/holdout splits
- no self-judging with the same model instance
- cost and latency as first-class metrics
- per-layer metric caps so one metric cannot dominate the composite
- archive all configs, scores, traces, and mutation rationale
- report confidence intervals or bootstrap variance once datasets are large enough

### 5. Repo landscape and maturity snapshot

GitHub API spot-check on 2026-07-07:

| Repo | Role | Stars | License | Fit |
|---|---:|---:|---|---|
| https://github.com/stanfordnlp/dspy | modular LM programs + optimizers | ~35.9k | MIT | high for optimizer inspiration |
| https://github.com/promptfoo/promptfoo | evals, red-team, CI | ~23.0k | MIT | high for production eval UX |
| https://github.com/confident-ai/deepeval | Pytest-like LLM metrics | ~16.7k | Apache-2.0 | high for metric registry |
| https://github.com/gepa-ai/gepa | reflective prompt/text evolution | ~5.5k | MIT | very high for mutator/archive |
| https://github.com/zou-group/textgrad | textual feedback optimization | ~3.6k | MIT | high for feedback model |
| https://github.com/UKGovernmentBEIS/inspect_ai | serious eval substrate | ~2.3k | MIT | high for runner model |
| https://github.com/google-deepmind/opro | canonical LLM optimizer code | ~762 | Apache-2.0 | conceptual reference |
| https://github.com/opendatahub-io/agent-eval-harness | skill/agent eval harness | ~32 | Apache-2.0 | niche but relevant |
| https://github.com/harness/harness-evals | normalized score + prompt optimizer | ~8 | Apache-2.0 | direct but early |

## Recommendation

Build v1 as an **adapter-first Python library plus CLI**, not a web app and not a giant platform.

Minimum useful v1:

1. `EvalCase`, `RunResult`, `Metric`, `TaskAdapter`, `Archive` typed interfaces.
2. JSON/YAML dataset format with train/dev/holdout split support.
3. Metric registry with deterministic metrics first: exact match, token F1, recall@k, cost, latency, schema validity.
4. LLM-judge metrics behind explicit config, with separate judge model enforcement.
5. Config schema and coupling-constraint validator before every run.
6. Archive stored as SQLite or JSONL: config, scores, traces, mutation rationale, parent config, timestamp, cost.
7. Mutator v1: GEPA-lite loop over config JSON and prompt text using failure cases and traces.
8. Convergence: patience/epsilon plus budget ceilings.
9. Report generator: top configs, per-category regressions, holdout delta, cost/latency tradeoff.

Avoid in v1:

- full AutoML complexity
- code mutation
- multi-agent orchestration
- dashboards
- universal benchmark claims

Best implementation path:

1. Implement the generic core locally.
2. Build one serious RAG/MUCi adapter.
3. Add one non-RAG adapter quickly after, ideally classifier or chatbot, to force generality.
4. Compare against promptfoo/DeepEval/Inspect only where useful; do not chase feature parity.
5. Consider integrating GEPA later rather than reinventing a sophisticated optimizer.

## Caveats / blockers

- Search was focused on GitHub and arXiv, as requested. I did not do a full package-registry, vendor-doc, or blog sweep.
- Some 2026 arXiv items are very recent; treat them as directional until they have broader replication.
- Stars are a rough maturity signal only; API numbers were checked during this run on 2026-07-07.
- The local repo currently contains a concept doc and an architecture image, not implementation code, so recommendations are architectural rather than code-review findings.

## Sources

- Local framework doc: `file:///root/agentic-eval-evolution-runtime/Generisches_Eval_Harness_Framework.md`
- Promptfoo: https://github.com/promptfoo/promptfoo
- DeepEval: https://github.com/confident-ai/deepeval
- Inspect AI: https://github.com/UKGovernmentBEIS/inspect_ai
- DSPy repo: https://github.com/stanfordnlp/dspy
- DSPy paper: https://arxiv.org/abs/2310.03714
- Harness Evals: https://github.com/harness/harness-evals
- Opendatahub agent-eval-harness: https://github.com/opendatahub-io/agent-eval-harness
- OPRO paper: https://arxiv.org/abs/2309.03409
- OPRO code: https://github.com/google-deepmind/opro
- TextGrad paper: https://arxiv.org/abs/2406.07496
- TextGrad repo: https://github.com/zou-group/textgrad
- GEPA paper: https://arxiv.org/abs/2507.19457
- GEPA repo: https://github.com/gepa-ai/gepa
- Unified framework for LLM agentic capabilities evaluation: https://arxiv.org/html/2602.03238v2
- VeRO: https://arxiv.org/html/2602.22480v1
- AutoPDL: https://arxiv.org/html/2504.04365v1
- Agentic Benchmark Checklist: https://arxiv.org/html/2507.02825v2
- Environment-grounded automated prompt optimization: https://arxiv.org/html/2606.17838v1
- Evolving Excellence / Artemis: https://arxiv.org/html/2512.09108v1
- Evaluation and Benchmarking of LLM Agents survey: https://arxiv.org/html/2507.21504v1
