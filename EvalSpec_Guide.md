# EvalSpec Guide

EvalSpec is the portable contract that lets an agent generate QA cases, regression tests, golden datasets, and eval runs for a target system.

The target can be an API, CLI, agent runtime, RAG system, coding-agent task, workflow, or UI. The principle stays the same:

```text
Spec -> generated cases -> eval run -> score/trace -> archive -> promote or reject
```

## Core Rule

**Black-box outcome first, white-box diagnostics second.**

The system is judged by what it produces for a task. Internal checks exist to explain failures, not to replace outcome quality.

```text
E2E Eval       decides pass/fail/promote
Component Eval explains where the failure came from
Trace Eval     checks whether critical steps happened
Guardrail Eval disqualifies unsafe or invalid variants
```

Short version:

```text
Output decides. Internal parts explain. Guardrails constrain.
```

## Why Endpoint Lists Are Not Enough

An OpenAPI file, GraphQL schema, or route list tells an agent what can be called. It does not tell the agent what is correct.

To generate useful QA datasets, EvalSpec must include:

- fixtures: known users, records, states, and reset commands
- oracles: rules that define correctness
- invariants: truths that must always hold
- forbidden behavior: things the system must never do
- expected side effects: what state should change after an action
- hidden cases: tests the optimizing agent cannot see
- split policy: which cases are visible during mutation and which are holdout

Without oracles, an agent can generate plausible tests that assert nothing important.

## Eval Layers

### 1. E2E Cases

E2E cases are the source of truth.

Examples:

- API input returns the expected response and state change
- RAG question returns a grounded, correct answer
- coding agent patch passes visible and hidden tests
- workflow finishes with the expected database state

Use E2E cases to decide promotion.

```yaml
e2e_cases:
  - id: create_order_happy_path
    input:
      method: POST
      path: /orders
      body:
        user_id: usr_basic
        items:
          - sku: sku_001
            quantity: 2
    expect:
      status: 201
      response_contains:
        - order_id
        - total
      state:
        order_exists: true
        inventory_decremented: true
```

### 2. Component Checks

Component checks test replaceable internal parts when they help diagnose failures.

Examples:

- retriever returns required document IDs
- tool router chooses the correct tool class
- planner emits required milestones
- patch generator touches expected files
- evaluator adapter parses test output correctly

Use component checks for diagnosis and regression, not as the primary success claim.

```yaml
component_checks:
  - id: retrieval_finds_policy_doc
    component: retriever
    input: "Can blocked users create orders?"
    expect:
      required_doc_ids:
        - policy_blocked_users
```

### 3. Trace Checks

Trace checks verify that important steps happened, without freezing the entire implementation.

Good trace checks:

- must call `legal_date_checker`
- must not call `production_db_write`
- must run tests before proposing patch
- must cite at least one retrieved source
- must not call external network during sandboxed eval

Bad trace checks:

- force every intermediate thought
- require an exact chain of prompts
- overfit to one framework's internal events

```yaml
trace_checks:
  - id: coding_agent_runs_tests_before_patch
    must_have_events:
      - test_command_started
      - file_patch_created
    ordering:
      - before: test_command_started
        after: file_patch_created
```

### 4. Guardrails

Guardrails are hard disqualifiers.

They are not weighted metrics. A guardrail violation rejects the candidate.

Examples:

- eval harness changed
- hidden tests changed
- forbidden files changed
- secrets accessed
- state mutated by GET request
- response leaks stack trace
- coding agent writes outside sandbox

```yaml
guardrails:
  forbidden_file_changes:
    - eval/**
    - hidden_tests/**
    - .env
  forbidden_side_effects:
    - mutate_state_on_get
    - external_network_call
  disqualify_on:
    - guardrail_violation
    - holdout_leakage
    - evaluator_checksum_changed
```

## Minimal EvalSpec

```yaml
version: 0.1
name: billing-api
owner: eval-team

target:
  type: api
  description: Billing API for orders, invoices, and refunds

interfaces:
  openapi: ./openapi.yaml
  base_url_env: API_BASE_URL
  auth:
    type: bearer
    token_env: API_TOKEN

fixtures:
  reset_command: npm run db:reset:test
  seed_command: npm run db:seed:test
  known_entities:
    user_basic: usr_123
    user_blocked: usr_456
    invoice_paid: inv_100

oracles:
  invariants:
    - id: invoice_total_equals_sum_of_lines
      description: Invoice total must equal sum of line items, tax, and discounts.
    - id: refund_never_exceeds_paid_amount
      description: Refund amount must never exceed captured payment.
    - id: blocked_users_cannot_create_orders
      description: Blocked users must receive a permission error when creating orders.
  forbidden:
    - stack_traces_in_response
    - internal_ids_in_public_response
    - mutation_on_get

datasets:
  splits:
    train: 60
    validation: 20
    holdout: 20
  categories:
    - happy_path
    - edge_case
    - negative_case
    - permission_case
    - regression_case
  per_endpoint_min: 5
  include_cross_endpoint_flows: true

eval_layers:
  e2e: required
  component: diagnostic
  trace: diagnostic
  guardrail: disqualifying

metrics:
  hard_gates:
    - schema_validity
    - invariant_pass
    - guardrail_pass
  scores:
    - status_code_correctness
    - response_correctness
    - state_correctness
    - regression_pass_rate
    - latency

budgets:
  max_case_generation_cost_usd: 10
  max_eval_run_cost_usd: 5
  max_latency_ms_p95: 1500
```

## Agent Workflow

An agent using EvalSpec should follow this workflow.

### Step 1: Load Interfaces

Read API schemas, CLI help, repository structure, docs, and existing tests.

Output:

```text
target inventory
known endpoints/commands
known data models
known side effects
known auth/permission surfaces
```

### Step 2: Build Behavior Map

Map actions to expected behavior.

For each action, identify:

- input schema
- preconditions
- side effects
- expected response
- failure states
- permission rules
- invariants touched

### Step 3: Generate Candidate Cases

Generate cases across categories:

- happy path
- edge case
- negative case
- permission case
- regression case
- cross-endpoint flow
- state-machine transition
- security/abuse case

### Step 4: Attach Oracles

Every case needs at least one oracle.

Good:

```text
After refund, invoice.refunded_amount <= invoice.paid_amount
```

Bad:

```text
Response should look good
```

### Step 5: Assign Split

The agent may generate all cases, but the optimizer must not see all cases.

Recommended:

- train: visible to mutator
- validation: used for candidate selection
- holdout: final gate only
- redteam: final gate and periodic regression

### Step 6: Produce Artifacts

Expected generated files:

```text
evalspec.yaml
datasets/train.jsonl
datasets/validation.jsonl
datasets/holdout.jsonl
datasets/redteam.jsonl
oracles/invariants.yaml
reports/case_generation_report.md
```

## Coding-Agent EvalSpec

For coding-agent patch loops, EvalSpec describes a repo task.

```yaml
version: 0.1
name: checkout-service-patch-loop

target:
  type: coding_agent
  repo: ./target-repo
  task_source: ./tasks

sandbox:
  allowed_paths:
    - src/**
    - tests/**
  forbidden_paths:
    - eval/**
    - hidden_tests/**
    - .env
    - .github/workflows/**
  network: disabled

commands:
  setup: npm install
  visible_tests: npm test
  lint: npm run lint
  typecheck: npm run typecheck

hidden_eval:
  command: ./eval/run_hidden.sh
  checksum_sha256: "..."
  read_only: true

patch_policy:
  max_files_changed: 8
  max_diff_lines: 400
  require_tests_before_patch: true
  rollback_on_unit_failure: true
  rollback_on_regression: true

promotion:
  required:
    - visible_tests_pass
    - hidden_tests_pass
    - no_guardrail_violation
    - regression_count_not_increased
  prefer:
    - smaller_diff
    - lower_latency
    - lower_cost
```

## RAG EvalSpec

```yaml
version: 0.1
name: legal-rag

target:
  type: rag
  adapter: adapters.legal_rag.Adapter

fixtures:
  corpus: ./fixtures/legal_docs
  reset_command: ./scripts/reset_index.sh
  seed_command: ./scripts/build_index.sh

oracles:
  invariants:
    - answer_must_cite_current_law
    - answer_must_not_invent_sources
    - answer_must_state_uncertainty_when_context_missing

e2e_cases:
  - id: current_tax_deadline
    input: "What is the current filing deadline for X?"
    expect:
      required_claims:
        - deadline_date
      required_citations:
        - doc_tax_2026
      forbidden_claims:
        - outdated_2024_deadline

component_checks:
  - component: retriever
    expect:
      recall_at_k:
        k: 5
        required_doc_ids:
          - doc_tax_2026

trace_checks:
  - must_have_events:
      - retrieval_started
      - citation_verification_started
      - answer_synthesized
```

## API EvalSpec

```yaml
version: 0.1
name: inventory-api

target:
  type: api

interfaces:
  openapi: ./openapi.yaml
  base_url_env: API_BASE_URL

fixtures:
  reset_command: ./scripts/reset_test_db.sh
  seed_command: ./scripts/seed_inventory.sh

oracles:
  invariants:
    - stock_never_negative
    - reservation_expires_after_ttl
    - get_requests_do_not_mutate_state

e2e_generation:
  per_endpoint_min: 5
  include_permission_cases: true
  include_cross_endpoint_flows: true

trace_checks:
  - id: no_external_payment_call_in_test
    must_not_call_tool: payment_provider_live
```

## Quality Rules for Generated Cases

For the full audit checklist, use [Dataset_Quality_Checklist.md](Dataset_Quality_Checklist.md).

A generated case is acceptable only if it has:

- clear purpose
- deterministic setup
- deterministic oracle
- expected output or expected state
- category and difficulty
- split assignment
- no dependency on production secrets
- no dependency on live external systems unless explicitly marked
- clear failure message

Reject generated cases that:

- only assert HTTP 200
- only compare snapshots without semantic oracle
- require live production data
- contain personal data
- depend on current time without freezing time
- duplicate an existing case without adding coverage
- expose hidden holdout logic to the mutator

## Split Policy

The same agent may generate the datasets, but the optimizer must not see all generated cases.

Recommended process:

1. Generate many candidate cases.
2. Human or verifier agent reviews them.
3. Assign splits.
4. Freeze holdout and redteam.
5. Hide holdout/redteam from mutator prompts, traces, and visible feedback.
6. Rotate redteam periodically.
7. Version datasets like code.

## What To Avoid

Avoid making internal component checks too strict too early.

Bad:

```text
The planner must produce exactly these five steps.
```

Better:

```text
The final output must be correct.
The trace must show that legal-date verification happened before final answer.
```

Avoid endpoint-only test generation.

Bad:

```text
Call every endpoint once and assert 2xx.
```

Better:

```text
Generate flows that exercise invariants, side effects, permissions, and regressions.
```

Avoid treating the composite score as truth.

Good candidates should pass hard gates first, then be compared by score vectors and Pareto tradeoffs.

## Implementation Notes

EvalSpec should be stable and framework-neutral.

No third-party framework types should appear in:

- EvalSpec schema
- TaskAdapter contract
- archive schema
- promotion/rollback rules

Adapters may call external tools. Specs should not depend on them.
