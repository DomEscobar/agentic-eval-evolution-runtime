# DeepResearch Report: Eval-Guided Patch Loops for Coding Agents

## Executive summary

Yes: the concept Dom described exists, is research-backed, and is probably the sharpest framing for the coding-agent branch of this repo.

The clean formulation is:

```text
An eval system acts as the benchmark/fitness layer for a coding agent.
The coding agent produces code patches.
The eval system runs tests, hidden checks, static checks, and quality gates.
The loop repeats until a baseline benchmark or quality gate is reached.
```

This is different from prompt/config evolution. For coding agents, the mutation target can be real code: application code, agent scaffold code, tool-use logic, context management, review logic, or test-generation logic.

The strongest evidence:

- **SWE-bench** defines the canonical real-world GitHub issue -> patch benchmark: https://arxiv.org/abs/2310.06770
- **SWE-agent** shows the standard agent shape: repository navigation, editing, test execution, and patch submission: https://github.com/swe-agent/swe-agent
- **AutoCodeRover** shows autonomous program improvement via code search/debugging/patch generation: https://arxiv.org/abs/2404.05427
- **TDAD** is the closest direct reference for the exact loop: agent modifies tool/source files, unit tests gate acceptance, SWE-bench subset measures improvement, regressions roll back, and evaluator integrity is protected by checksum/read-only controls: https://arxiv.org/html/2603.17973v1
- **SICA** is the formal term for self-improving coding agents that edit their own codebases/scaffolds to improve cost, speed, and benchmark performance: https://arxiv.org/html/2504.15228v2
- **Darwin Godel Machine (DGM)** directly validates the self-improving coding-agent idea: it modifies its own code, archives variants, and uses coding benchmarks as empirical fitness: https://arxiv.org/abs/2505.22954
- **Huxley-Godel Machine (HGM)** refines DGM by selecting lineages/clades with better future improvement potential, not only higher current benchmark score: https://arxiv.org/abs/2510.21614
- **AlphaEvolve** validates evaluator-guided evolutionary code improvement in algorithm/scientific-code settings: https://arxiv.org/abs/2506.13131
- **Kitchen Loop** is the strongest counterpoint to pure benchmark chasing: it anchors autonomous evolution to a specification surface and regression oracle, not just a metric: https://arxiv.org/abs/2603.25697
- **Live-SWE-agent** pushes the idea toward runtime self-evolution of the agent scaffold: https://arxiv.org/abs/2511.13646

My verdict: the repo should explicitly document two modes:

1. **Config/prompt evolution** for RAG/chat/classifier systems.
2. **Code patch evolution** for coding agents, where the agent produces patches until an eval-defined baseline is reached.

## Evidence-backed findings

### 1. SWE-bench gives the benchmark substrate

SWE-bench is the right mental model for coding-agent evaluation. It uses real GitHub issues and asks the model/agent to edit the repository to resolve the issue: https://arxiv.org/abs/2310.06770

SWE-bench Verified matters because it narrows this to 500 human-filtered tasks reviewed for clarity, test correctness, and solvability: https://www.swebench.com/verified.html

For this repo, the project-local equivalent should be:

```text
issue/task description
+ repository snapshot
+ allowed tools
+ visible tests
+ hidden tests
+ patch output
+ scoring result
```

### 2. Patch-producing coding agents already exist

SWE-agent is a direct reference implementation class: it takes a GitHub issue and uses an LM with tools to fix real repositories: https://github.com/swe-agent/swe-agent

Its paper argues that the agent-computer interface is a performance lever because it affects repository navigation, file editing, and test execution: https://arxiv.org/abs/2405.15793

AutoCodeRover is another direct reference, with a more software-engineering-flavored approach: code search, debugging, fault localization, and patch generation for GitHub issues: https://github.com/AutoCodeRoverSG/auto-code-rover and https://arxiv.org/abs/2404.05427

Repeton is useful because it names the patch-loop pattern clearly: diagnose -> propose code changes -> validate by tests -> repeat: https://arxiv.org/html/2506.08173v1

### 3. TDAD is the closest direct reference

TDAD's Section 5.5 describes almost the exact mechanism Dom is pointing at:

```text
full experiment history
-> coding agent makes one focused source change
-> unit tests gate acceptance
-> failed tests trigger immediate revert
-> benchmark eval on 5-25 SWE-bench instances
-> improvement updates best snapshot
-> regression triggers rollback
-> lateral move may be kept for exploration
```

The anti-gaming safeguards are directly reusable:

- evaluation script is SHA-256 checksummed
- evaluator is set read-only
- unit-test failure reverts immediately
- five consecutive reverts force restore to the best snapshot
- regression rate is measured separately from resolution

TDAD reports that this auto-improvement loop raised resolution from 12% to 60% on a 10-instance subset with 0% regression: https://arxiv.org/html/2603.17973v1

This should be copied into the framework as the first concrete implementation pattern.

### 4. The self-improving coding-agent version is real, not speculative

The Darwin Godel Machine is the key paper. It iteratively modifies its own agent code, archives generated agents, and validates changes on coding benchmarks. It reports improvement from 20.0% to 50.0% on SWE-bench and 14.2% to 30.7% on Polyglot, with sandboxing and human oversight: https://arxiv.org/abs/2505.22954

That is almost exactly Dom's point:

```text
baseline score
-> agent patches itself / its scaffold
-> benchmark eval
-> archive variant
-> repeat until better variant reaches target
```

AlphaEvolve supports the broader evaluator-guided evolutionary coding pattern, but in algorithm/scientific discovery rather than GitHub issue repair: https://arxiv.org/abs/2506.13131

Live-SWE-agent is the more aggressive runtime version: the agent evolves its own scaffold on the fly while solving issues. Treat it as inspiration, not v1 scope: https://arxiv.org/abs/2511.13646

SICA gives the formal label for this class: self-improving coding agent. Its key distinction is that the mutable codebase includes the agent scaffold itself, not only the target application: https://arxiv.org/html/2504.15228v2

Huxley-Godel Machine refines the archive concept: do not only store the highest current score. Estimate which lineage or clade is most likely to produce stronger descendants. That maps well to this repo's Archive concept because it turns the archive into a search tree rather than a leaderboard: https://arxiv.org/abs/2510.21614

### 5. The eval system should be the judge, not the patcher

The clean separation is:

```text
Eval System
  - owns benchmark
  - runs tests/checks
  - computes score
  - enforces hidden gates
  - archives variants
  - decides baseline reached or not

Coding Agent
  - inspects repo
  - edits code
  - runs allowed local commands
  - submits patch
  - reads visible feedback
  - tries again

Sandbox / Guardrails
  - restrict file scope
  - restrict commands
  - cap diff size
  - block secrets/network/destructive ops
  - prevent benchmark leakage
```

If the eval system also patches, the roles blur. That makes it harder to audit why a variant won and easier for the optimizer to tamper with its own judge.

### 6. Baseline-reaching loops need hidden gates

The obvious failure mode: the coding agent learns to pass visible tests without solving the real task.

Required gates:

- visible tests for feedback
- hidden tests for validation
- static checks
- style/type/lint checks
- diff-size limits
- forbidden-file checks
- no changes to benchmark/eval harness
- no changes to guardrails
- no future-repo-state leakage
- holdout tasks not visible to the agent loop

The SWE-bench ecosystem has known leakage/loophole concerns, including future repository state exposure: https://github.com/SWE-bench/SWE-bench/issues/465

The Agentic Benchmark Checklist is relevant here: benchmark setup and reward design can strongly misestimate agent capability if validity checks are weak: https://arxiv.org/html/2507.02825v2

### 7. Kitchen Loop: specification beats proxy optimization

The Kitchen Loop is the most important corrective to a naive "benchmark reached = done" framing.

It argues that autonomous software evolution should be anchored to:

- a specification surface: what the product claims to support
- "As a User x1000": repeated synthetic usage against that surface
- unbeatable tests: verification the code author cannot fake
- drift control: pause gates when quality degrades
- regression oracle: bounded answer to "is the system at least as good as before?"

The key lesson: optimize toward a specification, not only a score. Otherwise the coding agent can Goodhart the benchmark while product behavior degrades outside measured dimensions.

Source: https://arxiv.org/abs/2603.25697

### 8. Archive design is not optional

For self-improving coding agents, the archive should store:

- agent version
- target repo snapshot
- task id
- full patch/diff
- changed files
- commands run
- stdout/stderr logs
- visible test results
- hidden test result summary
- static check results
- score
- failure reason
- parent agent/version
- mutation rationale
- cost/latency
- model versions
- sandbox policy

DGM's archive/tree of generated agents is the strongest reason to make archive a first-class component rather than a log folder.

### 9. Repo maturity snapshot

GitHub API spot-check during this run:

| Repo | Role | Stars | Fit |
|---|---:|---:|---|
| https://github.com/SWE-agent/SWE-agent | coding agent for GitHub issues | ~19.7k | very high |
| https://github.com/swe-agent/mini-swe-agent | minimal strong coding agent | ~5.6k | high |
| https://github.com/SWE-bench/SWE-bench | benchmark/eval substrate | ~5.4k | very high |
| https://github.com/AutoCodeRoverSG/auto-code-rover | autonomous program improvement | ~3.1k | high |
| https://github.com/OpenHands/benchmarks | benchmark infrastructure | ~97 | medium |
| https://github.com/OpenAutoCoder/live-swe-agent | runtime self-evolving SWE agent | ~409 | research inspiration |

## Recommendation

Add a dedicated **Coding Agent Patch Mode** to the architecture.

### Name

Good names:

- **Eval-Guided Patch Loop**
- **Benchmark-Driven Self-Improvement**
- **Coding Agent Baseline Climber**
- **Patch-to-Baseline Runtime**

Best precise phrase:

```text
Eval-guided autonomous patch loop for coding agents
```

### v1 architecture

```text
Task Spec
  -> Repo Snapshot
  -> Coding Agent
  -> Patch
  -> Eval Harness
       - unit tests
       - hidden tests
       - lint/type/static checks
       - forbidden-file checks
       - diff-size checks
       - cost/latency
  -> Score + Trace
  -> Archive
  -> Agent gets visible feedback
  -> repeat until baseline reached or budget exhausted
```

### TDAD-style v1 loop

The most directly buildable loop is:

```text
snapshot current repo
checksum evaluator
set evaluator/hidden tests read-only
invoke coding agent: "make one focused improvement"
if no files changed: continue
run unit tests
if unit tests fail: restore snapshot and continue
run benchmark subset
compare against best snapshot
if improved: save as best
if regressed: restore best snapshot
if five consecutive reverts: mandatory restore to best
archive patch, logs, score, and rationale
```

This is the concrete version of "the eval system helps a coding agent reach a baseline by patching."

### v1 mutation target

Start with patches to the target app/repo, not self-modifying the agent scaffold.

Then add scaffold improvement later:

```text
Phase 1: agent patches target repo to pass project benchmark
Phase 2: agent proposes changes to its own scaffold as PRs
Phase 3: archive/benchmark selects better scaffold variants
Phase 4: DGM-like open-ended agent evolution
```

### Baseline gate

Baseline should not be a single score. Use a bundle:

```text
required:
  visible_tests: pass
  hidden_tests: pass
  forbidden_files: unchanged
  guardrails: pass
  diff_size: under limit
  no_eval_harness_changes: true
  cost: under budget

score:
  correctness: 60%
  regression_safety: 15%
  patch_minimality: 10%
  code_quality_static_checks: 10%
  cost_latency: 5%
```

### Critical rule

The coding agent may patch application code. It may not patch the evaluator, hidden tests, guardrails, or archive.

Self-patching the agent scaffold should be treated as a separate advanced mode with stronger gates and human review.

## Caveats / blockers

- The research strongly supports the concept, but v1 should avoid unrestricted self-modifying agent code.
- SWE-bench-style public benchmarks are useful for comparison, but project-local hidden tests are necessary to avoid overfitting and leakage.
- Some newer 2025/2026 papers and repos, especially Live-SWE-agent and long-horizon benchmarks, should be treated as directional until reproduced broadly.
- Coding-agent patch loops require real sandboxing, not just prompt instructions.
- A baseline score is only meaningful if the benchmark is clean, hidden, reproducible, and not editable by the agent.
- TDAD's auto-improvement loop used a fixed 10-instance evaluation set, so overfitting cannot be ruled out; project-local loops need rotating validation/holdout tasks.
- Kitchen Loop's specification-surface approach is only as good as the enumerated product claims and regression oracle.

## Sources

- SWE-bench paper: https://arxiv.org/abs/2310.06770
- SWE-bench site: https://www.swebench.com/
- SWE-bench Verified: https://www.swebench.com/verified.html
- SWE-bench GitHub: https://github.com/swe-bench/SWE-bench
- SWE-bench experiments: https://github.com/swe-bench/experiments
- SWE-agent repo: https://github.com/swe-agent/swe-agent
- mini-SWE-agent repo: https://github.com/swe-agent/mini-swe-agent
- SWE-agent paper: https://arxiv.org/abs/2405.15793
- OpenHands Benchmarks: https://github.com/OpenHands/benchmarks
- OpenHands: https://github.com/All-Hands-AI/OpenHands
- AutoCodeRover repo: https://github.com/AutoCodeRoverSG/auto-code-rover
- AutoCodeRover paper: https://arxiv.org/abs/2404.05427
- TDAD: https://arxiv.org/html/2603.17973v1
- SICA: https://arxiv.org/html/2504.15228v2
- Repeton patch-and-test repair: https://arxiv.org/html/2506.08173v1
- APR-agent pipeline analysis: https://arxiv.org/html/2506.08311v2
- Darwin Godel Machine: https://arxiv.org/abs/2505.22954
- Huxley-Godel Machine: https://arxiv.org/abs/2510.21614
- HGM repo: https://github.com/metauto-ai/HGM
- DGM HTML: https://arxiv.org/html/2505.22954v3
- Live-SWE-agent: https://arxiv.org/abs/2511.13646
- Live-SWE-agent repo: https://github.com/OpenAutoCoder/live-swe-agent
- AlphaEvolve: https://arxiv.org/abs/2506.13131
- AlphaEvolve blog: https://deepmind.google/blog/alphaevolve-a-gemini-powered-coding-agent-for-designing-advanced-algorithms/
- Kitchen Loop: https://arxiv.org/abs/2603.25697
- Kitchen Loop repo: https://github.com/0xagentkitchen/kitchenloop
- SWE-EVO: https://arxiv.org/html/2512.18470v5
- SWE-bench Pro: https://arxiv.org/html/2509.16941v1
- Agentic Benchmark Checklist: https://arxiv.org/html/2507.02825v2
