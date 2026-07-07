# DeepResearch Report: Eval Dataset Quality

## Executive summary

Short verdict: **the design is good, but the dataset is not "really good" yet because there is no real dataset yet.**

What the repo currently has is a strong **dataset architecture**:

- EvalSpec separates E2E outcomes, component diagnostics, trace checks, and guardrails.
- It requires oracles, invariants, fixtures, and split policy instead of relying on endpoint lists.
- It uses train/validation/holdout/redteam splits.
- It treats guardrails as disqualifiers, not weighted metrics.
- It warns against over-constraining internal implementation details.

That is the right shape. But actual dataset quality depends on the concrete cases, labels, reviewers, hidden tests, production representativeness, and contamination resistance. Those do not exist yet in the repo.

My rating:

```text
Dataset architecture: 8.5/10
Actual dataset quality today: unknown / not yet measurable
Risk if generated naively from endpoints: high
Risk if generated from EvalSpec + human/oracle review: medium-low
```

The biggest correction: do not call the dataset "good" until it passes a dataset audit. Call the current artifact a **good EvalSpec design for building a high-quality dataset**.

## Evidence-backed findings

### 1. The repo's EvalSpec design matches current best practice

The local [EvalSpec guide](file:///root/agentic-eval-evolution-runtime/EvalSpec_Guide.md) says:

```text
Black-box outcome first, white-box diagnostics second.
E2E decides. Components explain. Traces verify critical steps. Guardrails disqualify.
```

That is aligned with the Agentic Benchmark Checklist, which emphasizes task validity, outcome validity, and rigorous reporting. ABC warns that benchmark flaws can under- or overestimate agent performance by up to 100% relative terms: https://arxiv.org/abs/2507.02825

This is a strong point in our design.

### 2. The split policy is right, but must be enforced mechanically

Our design uses:

- train: visible to optimizer
- validation: candidate selection
- holdout: final gate only
- redteam: adversarial and regression gate

This is directionally correct. But it only works if holdout/redteam are mechanically hidden from:

- mutator prompts
- visible traces
- detailed failure explanations
- generated reports seen by the optimizer
- coding-agent filesystem access

If the optimizer repeatedly sees detailed holdout failures, the holdout becomes validation data.

### 3. Endpoint-generated QA is dangerous without oracles

The repo is right to say endpoint lists are not enough.

An OpenAPI file can generate coverage like:

```text
call POST /orders
assert status 200 or 201
```

That is weak. Good generated cases need:

- fixtures
- expected state transitions
- invariants
- forbidden behavior
- permission rules
- side-effect rules
- domain-specific oracles

RAGAS makes a similar point for RAG: an ideal eval dataset should include different production question types and difficulty levels because LLM generation tends to follow common paths by default: https://docs.ragas.io/en/v0.1.21/concepts/testset_generation.html

### 4. Human verification is a quality multiplier

SWE-bench Verified is relevant because it is a human-filtered subset reviewed for:

- clear problem descriptions
- correct tests
- solvability given available information

Source: https://www.swebench.com/verified.html

SWE-bench Pro similarly emphasizes human verification and sufficient context for harder tasks: https://arxiv.org/html/2509.16941v1

Conclusion: generated datasets should not go straight into holdout. They need review, or at least a verifier-agent plus spot human review.

### 5. Contamination and memorization are serious risks

The SWE-bench Illusion paper is directly relevant. It argues some SWE-bench Verified performance may come from memorization or dataset artifacts; models can identify buggy file paths from issue descriptions alone: https://arxiv.org/html/2506.12286v4

Data contamination surveys and retro-holdout work show why public or repeatedly exposed eval cases become unreliable over time:

- https://arxiv.org/html/2406.04244v1
- https://arxiv.org/html/2410.09247v1
- https://arxiv.org/html/2605.19999v1

For our system, this means:

- public examples are fine for smoke tests
- private project-local holdout is required for real claims
- holdout must rotate or be replenished
- redteam cases must be renewed after repeated exposure
- production-derived failures should usually feed train/validation, not holdout

### 6. Our layered eval design is better than a single composite score

The repo says hard gates first, then score vectors/Pareto tradeoffs. That is better than one composite score.

Why:

- a composite score can hide catastrophic failures
- a weighted metric can be gamed
- guardrail violations should disqualify, not merely penalize
- internal component metrics can be useful but should not override E2E outcomes

This aligns with the benchmark-quality literature and the Kitchen Loop argument already captured elsewhere in the repo: optimize toward a specification/regression oracle, not only a proxy score.

### 7. What is missing before we can call it "good"

The current repo still lacks:

- actual cases
- labeled expected outputs
- fixture reset/seed scripts
- review workflow
- case ownership
- dataset versioning policy
- leakage policy
- minimum acceptance thresholds
- judge calibration plan
- inter-rater agreement plan for human labels
- production-distribution sampling policy
- failed-case promotion flow from production into datasets

These are not optional. They are what turn a good schema into a good dataset.

## Recommendation

### Keep the EvalSpec architecture

The design is strong enough to keep. Do not simplify it to endpoint-based QA.

### Add a dataset quality gate

Before any dataset is trusted, run a dataset audit.

Recommended checklist:

```text
Task validity
  - each case maps to a real user/task need
  - task is solvable with available context
  - task is not ambiguous unless ambiguity is explicitly tested

Outcome validity
  - expected output or expected state is clear
  - oracle is deterministic or judge rubric is calibrated
  - tests reject bad solutions and accept correct alternatives

Coverage
  - happy path
  - edge cases
  - negative cases
  - permission/security cases
  - regression cases
  - cross-step workflows

Split hygiene
  - train/validation/holdout/redteam assigned
  - holdout hidden from optimizer
  - redteam hidden or restricted
  - no duplicate or near-duplicate leakage across splits

Contamination resistance
  - public benchmark cases not used for final claims
  - project-local holdout private
  - production failures feed train/validation first
  - holdout refreshed over time

Operational readiness
  - reset/seed fixtures work
  - no production secrets
  - no live external dependencies unless explicitly marked
  - dataset versioned
  - report includes dataset version and split hashes
```

### Add a Dataset Quality Score

Do not use it as the final product metric, but use it to audit the dataset itself.

Suggested score:

```text
25% oracle clarity
20% production representativeness
15% split hygiene
15% edge/negative/security coverage
10% fixture determinism
10% reviewer confidence
 5% contamination resistance
```

Any case with no oracle should be rejected, not scored.

### Recommended next doc/code addition

Add `Dataset_Quality_Checklist.md` or a `dataset_audit` section to `EvalSpec_Guide.md`.

The most useful practical output would be:

```text
aer dataset-audit evalspec.yaml datasets/
```

It should produce:

```text
dataset_quality_report.md
split_leakage_report.json
coverage_report.json
oracle_gap_report.json
```

## Caveats / blockers

- This research audits the design, not a real dataset, because the repo has no concrete dataset cases yet.
- Two fetched sources failed through the artifact fetcher: OpenAI SWE-bench Verified page returned HTTP 403 and LangSmith docs timed out. Similar content was available through other fetched sources and search snippets, but this is still a fetch limitation.
- Vendor docs such as DeepEval/RAGAS/LangSmith are useful implementation references but not neutral benchmark-quality research.
- Some 2026 contamination-resistant benchmark proposals are new and directional; use them as warning signs, not fixed requirements.

## Sources

- Local EvalSpec guide: `file:///root/agentic-eval-evolution-runtime/EvalSpec_Guide.md`
- Agentic Benchmark Checklist: https://arxiv.org/abs/2507.02825
- ABC project page: https://uiuc-kang-lab.github.io/agentic-benchmarks/
- SWE-bench Verified: https://www.swebench.com/verified.html
- OpenAI SWE-bench Verified page: https://openai.com/index/introducing-swe-bench-verified/
- SWE-bench Illusion: https://arxiv.org/html/2506.12286v4
- Data contamination survey: https://arxiv.org/html/2406.04244v1
- Contamination-resistant benchmarks: https://arxiv.org/html/2605.19999v1
- Retro-holdouts: https://arxiv.org/html/2410.09247v1
- OpenAI evaluation best practices: https://developers.openai.com/api/docs/guides/evaluation-best-practices
- OpenAI Evals repo: https://github.com/openai/evals
- DeepEval datasets: https://deepeval.com/docs/evaluation-datasets
- DeepEval introduction: https://deepeval.com/docs/evaluation-introduction
- DeepEval metrics: https://deepeval.com/docs/metrics-introduction
- RAGAS docs: https://docs.ragas.io/en/stable/
- RAGAS metrics: https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/
- RAGAS testset generation: https://docs.ragas.io/en/v0.1.21/concepts/testset_generation.html
- LangSmith evaluation docs: https://docs.langchain.com/langsmith/evaluation
- Agent evaluation readiness checklist: https://www.langchain.com/blog/agent-evaluation-readiness-checklist
- SWE-bench GitHub: https://github.com/swe-bench/SWE-bench
- SWE-bench Pro: https://arxiv.org/html/2509.16941v1
- SWE-EVO: https://arxiv.org/html/2512.18470v5
