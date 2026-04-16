# Agentic Test Automation Guidelines (CIGE)

Behavioral guidelines for authoring and executing agentic AI test cases. Based on the CIGE standard: **C**ontext, **I**ntent, **G**uardrails, **E**xecution.

**Tradeoff:** These guidelines bias toward safety and goal fidelity over speed. For trivial assertions, use judgment.

## 1. Establish Context Before Testing

**Narrow the agent's decision space. Never test into the void.**

Before writing or executing any test:
- Declare the system under test explicitly: what app, environment, version, and precondition state?
- Specify what tools and APIs the agent has access to.
- State environmental assumptions: auth state, seed data, feature flags, dependent services.
- If context is ambiguous, ask — do not assume defaults and run.

Context that is missing or vague leads to wasted exploration, flaky results, and unpredictable agent behavior.

## 2. Anchor Every Test to Intent

**Intent is the stable core. Never collapse it into steps.**

When defining or analyzing a test:
- Write intent as an outcome, not a procedure: "User can complete checkout" — not "Click button, fill form, click submit."
- Intent must survive: UI changes, infrastructure migrations, workflow refactors.
- Ask: "What is the agent trying to confirm?" — that is the intent. Everything else is execution guidance.
- Separate intent from execution in every test definition. Never merge them.

The test: Can you state what this test is verifying in one outcome-focused sentence? If not, the intent is buried in the steps.

## 3. Define Guardrails Before Executing

**Bounded autonomy is safe autonomy. No guardrails = no execution.**

Before any agent runs a test:
- Declare what the agent must NOT do: no production mutations, no irreversible operations, no sensitive data exposure.
- Specify scope boundaries: which environments, data ranges, and services are off-limits.
- If an execution path would violate a guardrail, stop and surface it — never route around it.
- Guardrails are constraints, not suggestions. They define the difference between a test run and an incident.

Agents without guardrails optimize for test completion. Not safe test completion.

## 4. Treat Execution as Adaptive Guidance

**Steps are starting points. Goal fidelity is the finish line.**

When writing or running execution paths:
- Write execution steps as guidance, not rigid scripts. Agents adapt; brittle scripts break.
- Classify failures before deciding any action: product defect, outdated test logic, or infrastructure issue?
- Self-heal by category: update execution steps for logic drift, escalate product defects, retry for infrastructure failures.
- Loop until intent is confirmed — not until steps are checked off.
- Use progressive context disclosure: load only the execution details relevant to the current step.

The test: "Did the agent safely achieve the intended outcome?" — not "Did it follow every step exactly?"

---

## 5. Load Context in Layers, Never All at Once

**Filter before fetching. The agent only needs what it needs right now.**

Load CIGE fields in a fixed three-layer sequence — never the full test upfront:

1. **Intent only** — confirm the goal and that this test is relevant. ~2 lines. No environment setup yet.
2. **Context + Guardrails** — establish operating bounds before any action. Guardrails must be loaded before the first execution step runs.
3. **Execution steps, one at a time** — surface each step only when the previous one is complete. Never let the agent see step 5 while working on step 2.

`specRef` is a pointer loaded with Context — but its contents are fetched lazily, only when a self-healing agent needs it for failure classification. Normal test execution never reads the BRD.

Give each execution step a stable short ID (`e1`, `e2`, ...). When a step fails, the failure report references the ID and loads only the surrounding steps for diagnostic context — not the whole test.

After each run, produce a short summary: result, failed step ID, failure type, recovery action taken, and whether Intent remained unchanged. Future agents read the summary first — not the full test — to decide whether to proceed or investigate a pattern.

The test: Could the agent have reached a guardrail violation before seeing its Guardrails? If yes, the loading order is wrong.

## 6. Self-Heal by Field, Not by Instinct

**Classification determines which field an agent may touch. Nothing else does.**

When a test fails, before writing a single line of recovery logic:
- Classify the failure: infrastructure issue, outdated test logic, or product defect?
- Each classification maps to exactly one self-healing agent with a fixed set of writable fields.
- An agent that reaches outside its field boundaries has misclassified the failure — stop and reclassify.

The field mutation rules are absolute:

| Failure | Agent | May Write | Never Touches |
|---|---|---|---|
| Infrastructure failure | EnvironmentRecoveryAgent | Nothing in the test | Intent, Guardrails, Execution |
| Outdated test logic | StaleExecutionAgent | `Execution[]` only — with human approval | Intent, Context, Guardrails |
| Product defect | ProductDefectAgent | May *propose* `Intent` update — human approval required | Guardrails, Execution |

**Guardrails are never writable by any self-healing agent, under any condition.**

### How agents know what the system is supposed to do

Every CIGE test must include a `specRef` in its Context — a resolvable pointer to the BRD or product specification that defines expected system behavior. Self-healing agents fetch `specRef` to answer the question that classification depends on: *did the system break, or did the product intentionally change?*

- If `specRef` is unchanged and system behavior diverged → product defect → escalate
- If `specRef` was updated and system now matches the new spec → intentional change → propose Intent update for human approval
- If `specRef` is missing or unreachable → classification is blocked → halt and require it before proceeding

Without `specRef`, no agent can safely distinguish a bug from a feature. Do not write tests without it.

### Intent may only change under one condition

Intent evolves when the product specification changes — not when a test fails. The sequence is:
1. `specRef` is updated by a product owner
2. ProductDefectAgent detects that system behavior now matches the new spec
3. Agent proposes a new Intent with a diff: old Intent, new spec language, proposed new Intent
4. A human approves the change

This is supervised intent evolution, not autonomous self-healing. It requires a human gate.

---

**These guidelines are working if:** test cases survive system changes without intent rewrites, failures are classified before any agent acts, `specRef` is present in every test so agents can reference ground truth, and Guardrails are never touched regardless of what changed.
