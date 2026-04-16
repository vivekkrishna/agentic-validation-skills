# Agentic Validation Skills

> Guidelines and skills for agentic AI test automation, based on the [CIGE standard](https://medium.com/@vivek-k-c/cige-an-agentic-ai-test-case-standard-proposed-66d2cdca766f) — a structured test case format designed for self-healing, goal-driven AI agents.

A `CLAUDE.md` file and Claude Code skill that improve how AI agents author, execute, and recover from agentic test cases.

## The Problem

Traditional test formats (BDD, POM, AAA) were designed for deterministic systems with rigid step sequences. When an AI agent runs these tests:

- **Intent collapses into procedures** — when the UI changes, the whole test breaks even if the goal hasn't changed
- **No guardrails** — agents optimize for test completion, not safe test completion
- **Failure is binary** — pass/fail with no classification, so agents can't decide whether to self-heal, escalate, or retry
- **Context is assumed** — agents explore blindly instead of operating within a defined decision space
- **Token waste** — loading the full test upfront dilutes attention and increases cost

## The Solution: CIGE

The **CIGE standard** separates four concerns that traditional test formats conflate:

| Component | Role | Stability |
|---|---|---|
| **Context** | System under test, environment, tools, preconditions | Changes with environment |
| **Intent** | The single outcome the test must confirm | Stable — survives system changes |
| **Guardrails** | Explicit constraints on agent behavior | Stable — safety boundaries |
| **Execution** | Adaptive guidance for reaching the goal | Mutable — self-heals with the system |

Every agentic test case is expressed as:

```json
{
  "Context": "E-commerce checkout service, staging environment, user pre-authenticated with test account, payment gateway mocked",
  "Intent": "User can successfully place an order and receive an order confirmation",
  "Guardrails": [
    "Do not submit orders with real payment credentials",
    "Do not mutate production order records",
    "Halt if checkout flow exits the staging domain"
  ],
  "Execution": [
    "Navigate to cart with at least one item",
    "Proceed through checkout flow to order summary",
    "Submit order using test payment credentials",
    "Verify: order confirmation page is displayed with a valid order ID"
  ]
}
```

## The Four Principles

### 1. Establish Context Before Testing

**Narrow the agent's decision space. Never test into the void.**

Declare system, environment, tools, and state before writing a single execution step. Missing context causes wasted exploration and flaky results.

### 2. Anchor Every Test to Intent

**Intent is the stable core. Never collapse it into steps.**

Write intent as one outcome-focused sentence. It must survive UI changes, infrastructure migrations, and workflow refactors. The question is always: *"Did the agent safely achieve the intended outcome?"* — not *"Did it follow every step?"*

### 3. Define Guardrails Before Executing

**Bounded autonomy is safe autonomy. No guardrails = no execution.**

Declare what the agent must never do before any test runs. Guardrails are constraints, not suggestions. An agent that violates a guardrail to complete a test has not passed the test.

### 4. Treat Execution as Adaptive Guidance

**Steps are starting points. Goal fidelity is the finish line.**

Execution steps are guidance, not scripts. When the system changes, execution self-heals. But only after classifying the failure:

| Failure Type | Recovery |
|---|---|
| **Product Defect** | Escalate — preserve intent |
| **Outdated Test Logic** | Update execution only — intent must not change |
| **Infrastructure Failure** | Restore context — do not alter test definition |

Never rewrite intent to make a failing test pass.

## Reported Impact

From production use of the CIGE standard:

- ~90% reduction in test maintenance effort
- ~38% reduction in false positives
- ~62% reduction in average test execution time

## Install

**Option A: Claude Code Plugin (recommended)**

```
/plugin marketplace add vivekkrishna/agentic-validation-skills
```

```
/plugin install agentic-validation-skills
```

Once installed, invoke the skill with:
```
/agentic-validation-skills:cige-guidelines
```

**Option B: CLAUDE.md (per-project)**

New project:
```bash
curl -o CLAUDE.md https://raw.githubusercontent.com/vivekkrishna/agentic-validation-skills/main/CLAUDE.md
```

Existing project (append):
```bash
echo "" >> CLAUDE.md
curl https://raw.githubusercontent.com/vivekkrishna/agentic-validation-skills/main/CLAUDE.md >> CLAUDE.md
```

## How to Know It's Working

These guidelines are working if you see:

- **Tests survive system changes** — execution updates but intent never rewrites
- **Failures are classified, not just reported** — agents know whether to self-heal, escalate, or retry
- **Guardrail violations surface before execution** — not discovered mid-run
- **Context is always explicit** — no blind exploration, no assumed state
- **Shorter, more focused test runs** — progressive context disclosure reduces token waste

## Customization

Merge `CLAUDE.md` with your project-specific test guidelines:

```markdown
## Project-Specific Test Context

- Staging URL: https://staging.example.com
- Test account credentials are in Vault at secret/test-accounts
- Payment gateway mock is enabled by default in staging
- Do not test against the orders-v1 endpoint — deprecated, use orders-v2
```

## Background

CIGE was proposed as an alternative to free-form agentic test definitions. The full rationale is in the original article: [CIGE: An Agentic AI Test Case Standard Proposed](https://medium.com/@vivek-k-c/cige-an-agentic-ai-test-case-standard-proposed-66d2cdca766f).

The core insight: modern AI agents operate contextually, not mechanically — just like human testers. Test definitions should reflect that. Rigid step sequences served deterministic automation. Structured, intent-driven formats serve agentic automation.

## License

MIT
