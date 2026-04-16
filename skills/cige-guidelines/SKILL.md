# CIGE: Agentic Test Case Authoring

Use this skill when writing, reviewing, or refactoring AI agent test cases. It enforces the CIGE standard — a structured format that separates stable test intent from adaptive execution, enabling self-healing agentic tests.

## When to invoke

- Authoring a new agentic test case from scratch
- Reviewing an existing test for brittleness or missing structure
- Classifying a test failure and deciding on recovery action
- Designing test guardrails for a new agent workflow

---

## The CIGE Format

Every agentic test case must be expressed in this structure:

```json
{
  "Context": {
    "system": "<app or service under test, version if relevant>",
    "environment": "<staging | dev | ephemeral | ...>",
    "tools": ["<tool 1>", "<tool 2>"],
    "preconditions": ["<seed data>", "<auth state>", "<feature flags>"],
    "specRef": "<path or URL to the BRD / product spec that defines expected behavior>"
  },
  "Intent": "<single outcome-based objective — what success looks like>",
  "Guardrails": [
    "<constraint the agent must never violate>",
    "<scope boundary or irreversibility limit>"
  ],
  "Execution": [
    "<adaptive step 1 — guidance, not script>",
    "<adaptive step 2>",
    "<verification: confirm intent was achieved>"
  ]
}
```

`specRef` is required. It is the ground truth that self-healing agents use to distinguish a product defect from an intentional product change. Without it, failure classification cannot be completed.

---

## How to apply each field

### Context
Answer these before writing any execution:
- What system is being tested? (app, service, version, environment)
- What is the agent's starting state? (logged in, seeded data, feature flags)
- What tools does the agent have access to?
- What external dependencies must be available?
- Where is the product specification or BRD for this system? (`specRef`)

`specRef` can be a file path, a URL, a Notion/Confluence page ID, or any resolvable pointer to the document that defines expected system behavior. It must be kept up to date as the product evolves — it is what self-healing agents read to determine whether a failure is a bug or an intentional change.

**Rule:** If context is ambiguous, ask — do not assume defaults. If `specRef` is missing, flag it before writing execution steps.

### Intent
Write one sentence that answers: *"What outcome must be true for this test to pass?"*

- Use outcome language, not procedural language
- Intent must survive: UI redesigns, API refactors, workflow changes
- Bad: "Click checkout, fill address, submit order"
- Good: "User can successfully place an order and receive a confirmation"

**Rule:** Intent is the anchor. If execution changes but the intent sentence still holds, the test is still valid.

### Guardrails
Declare explicit constraints before execution begins:
- No mutations to production data
- No exposure of PII or secrets
- No irreversible operations (deletions, payments, emails) without scoped test doubles
- No execution outside the designated test environment

**Rule:** If a guardrail would be violated by any execution path, surface it and halt — do not route around it.

### Execution
Write steps as adaptive guidance, not rigid assertions:
- Steps are starting points; the agent adapts when the system changes
- End every execution block with an intent verification step
- Use progressive context disclosure: load steps on demand, not all upfront

**Rule:** A passing test means intent was confirmed safely. A failing test requires classification before any fix.

---

## Failure Classification and Self-Healing Agents

Every test failure must be classified before any recovery action is taken. Classification determines which self-healing agent runs and — critically — which CIGE fields that agent is allowed to touch.

### Field Mutation Rules (non-negotiable)

| CIGE Field | EnvironmentRecoveryAgent | StaleExecutionAgent | ProductDefectAgent |
|---|---|---|---|
| `Context` | Read-only (uses to restore env) | Read-only | Read-only (uses `specRef`) |
| `Intent` | Never touch | Never touch | May *propose* update — **human approval required** |
| `Guardrails` | Never touch | Never touch | Never touch |
| `Execution[]` | Never touch | **Writes** — human approval required | Never touch |

No agent may modify `Intent` autonomously. No agent may modify `Guardrails` at all.

---

### Agent 1: EnvironmentRecoveryAgent

**Triggers on:** Infrastructure Failure — the test environment, authentication, or a dependency is unavailable or misconfigured.

**Signal:** Execution fails before reaching the system under test (auth error, service unreachable, missing seed data, ephemeral environment not bootstrapped).

**What it does:**
1. Reads `Context` to identify what needs restoring (environment type, preconditions, dependent services)
2. Attempts recovery: re-authenticates, re-seeds data, restarts dependencies, or re-provisions the ephemeral environment
3. Re-runs the test in full after recovery
4. If recovery fails after 3 attempts, escalates with an environment diagnostic — it does **not** alter the test definition

**What it never touches:** Intent, Guardrails, Execution — the test is not the problem.

**Human gate:** Not required for recovery attempts. Required only when escalating a persistent environment failure.

---

### Agent 2: StaleExecutionAgent

**Triggers on:** Outdated Test Logic — the system under test changed (UI restructured, API path updated, workflow reordered) but the Intent is still valid and the product spec (`specRef`) reflects the same expected outcome.

**Signal:** Execution steps fail mid-flow (element not found, endpoint returns 404, workflow step no longer exists), but fetching `specRef` confirms the intended outcome is still a valid product requirement.

**What it does:**
1. Reads `Intent` (the goal it must preserve) and `Guardrails` (constraints that apply during repair)
2. Fetches `specRef` to confirm the intended outcome is still a stated product requirement
3. Inspects the current system state (live UI, API schema, current workflow) to understand what changed
4. Proposes new `Execution[]` steps that achieve `Intent` within `Guardrails`
5. Dry-runs the proposed steps to verify `Intent` can be confirmed
6. Surfaces the proposed diff to a human for approval — **does not commit without sign-off**

**What it never touches:** Intent, Context, Guardrails.

**Human gate:** Required before committing updated `Execution[]`. The agent proposes; a human approves. This prevents silent intent drift disguised as execution repair.

**Rule:** If the agent cannot find a path to confirm `Intent` within `Guardrails` using the current system, it must escalate — the failure may be misclassified and is actually a product defect.

---

### Agent 3: ProductDefectAgent

**Triggers on:** Product Defect — the system behaved in a way that contradicts the stated Intent, and the root cause is not environment or stale execution.

**Signal:** Execution reached the system under test, environment is healthy, execution steps are current — but the outcome does not match `Intent`.

**What it does:**
1. Reads `Intent` (what was expected) and captures actual system behavior (what happened)
2. Fetches `specRef` (the BRD / product spec) to determine ground truth:

   **Case A — `specRef` unchanged, system diverged:**
   The system broke. This is a confirmed product defect.
   → File a bug report with: Intent, actual behavior, specRef snapshot, execution trace
   → Do not modify the test definition

   **Case B — `specRef` was recently updated, system matches new spec:**
   The product intentionally changed. The Intent in the test is now stale.
   → Propose an Intent update that aligns with the new specRef
   → Surface the proposed change for **human approval before any write**
   → Guardrails are never touched, even if the spec changed

   **Case C — `specRef` is unreachable or missing:**
   Classification cannot be completed.
   → Halt and flag the missing specRef as a test authoring gap
   → Do not guess — require the specRef to be provided before proceeding

**What it never touches:** Guardrails, Execution. Only Intent may be proposed for update, and only under Case B with human approval.

**Human gate:** Always required. No autonomous writes to Intent under any condition.

---

### When Intent May Change (and Only Then)

Intent is the most protected field in CIGE. The only condition under which Intent update is permissible:

1. `specRef` was updated by a product owner or author
2. The new spec explicitly describes a different expected outcome
3. ProductDefectAgent proposes the Intent update with a diff showing old Intent, new spec language, and proposed new Intent
4. A human reviews and approves the proposed change

This is not self-healing. It is **supervised intent evolution** — acknowledging that the product changed, not that the test failed.

**Rule:** Never modify intent to make a failing test pass. That is intent corruption, not self-healing.

---

## Progressive Context Disclosure

Never load the full CIGE test case into an agent's context at once. Load fields in stages, in order, stopping at each layer until the current layer's work is complete before loading the next.

This follows the same principle as efficient memory retrieval systems: **filter before fetching**. Loading all execution steps upfront dilutes attention, increases token cost, and allows agents to "look ahead" and take shortcuts that bypass guardrail checks.

### The 3-Layer Loading Protocol

```
Layer 1 — ORIENT    →  Load: Intent only
Layer 2 — BOUND     →  Load: Context + Guardrails
Layer 3 — EXECUTE   →  Load: Execution[], one step at a time
```

**Layer 1 — Orient (Intent only)**
Load just the Intent field first. At ~1-2 sentences, this costs almost nothing. The agent uses it to:
- Confirm this test is relevant to the current task
- Establish the goal before seeing any constraints or steps
- Decide whether to proceed at all

Do not load Context or Execution until Intent is confirmed and the test is selected for running.

**Layer 2 — Bound (Context + Guardrails)**
Once Intent is confirmed, load Context and Guardrails together. The agent now knows:
- What environment to set up and what tools are available (Context)
- What it must never do before taking a single action (Guardrails)

Guardrails must be loaded before any Execution step. An agent that begins executing before seeing its Guardrails is operating without safety bounds.

`specRef` is part of Context but its **contents are not fetched at this layer**. The pointer is loaded; the document is not. The spec is fetched lazily — only when a self-healing agent needs it for failure classification. Normal test execution never reads the BRD.

**Layer 3 — Execute (Execution steps, one at a time)**
Load execution steps individually, on demand, as the agent progresses. Do not surface step 4 while the agent is working on step 2.

Each step should carry a short stable ID so failure reports can reference it precisely without repeating full step text:

```json
"Execution": [
  { "id": "e1", "step": "Navigate to cart with at least one item" },
  { "id": "e2", "step": "Proceed through checkout to order summary" },
  { "id": "e3", "step": "Submit using test payment credentials" },
  { "id": "e4", "step": "Verify: order confirmation page shows a valid order ID" }
]
```

When a step fails, a self-healing agent references the failure as `e3 failed` and loads the N steps before it for diagnostic context — not the entire test.

### specRef: Lazy Fetch on Demand

```
Normal execution:   Context loads → specRef pointer is known, contents NOT fetched
Failure occurs:     Self-healing agent fetches specRef contents for classification
Classification done: specRef contents discarded from context
```

This is the same pattern as a citation system: you know the reference exists from the moment Context loads, but you only pull the full document when you actually need to reason about it. In most test runs, specRef is never fetched at all.

### Run Summaries

After each test run, compress the execution trace into a short summary observation:

```json
{
  "testId": "checkout-happy-path",
  "runDate": "2026-04-16",
  "result": "pass | fail | self-healed",
  "failedStep": "e3",
  "failureType": "outdated-test-logic | product-defect | infrastructure",
  "recoveryAction": "StaleExecutionAgent updated e2, e3 — approved by human",
  "intentUnchanged": true
}
```

Future agents can load this summary (a few lines) before deciding whether to run a full test or to first check whether recent runs suggest a pattern. This is progressive disclosure applied across sessions, not just within a single run.

---

## Quality Check

A well-formed CIGE test satisfies all of these:

- [ ] Context answers: what system, what environment, what tools, what state
- [ ] Context includes a resolvable `specRef` pointer
- [ ] Intent is one outcome-focused sentence, free of procedural language
- [ ] Guardrails enumerate at least the irreversibility and scope boundaries
- [ ] Execution steps each carry a stable short ID
- [ ] Execution ends with an explicit intent verification step
- [ ] Test can be read by a human and understood without looking at the system

If any box is unchecked, the test is not ready to run.
