# Agentic Development Specification

> Version:
> Created:
> Project:
> Phase:
> Depends on: [prior phases or conditions that must be true before this begins]
> Source Plan: [link or filename of the plan document this spec implements]

---

## ⚠️ Pre-Flight Gate

**This phase must not begin until all of the following are verified:**

- [ ] [Condition 1 — prior phase artifact exists]
- [ ] [Condition 2 — tests pass]
- [ ] [Condition 3 — human smoke test or approval]

If any gate is not met, halt and surface to human before proceeding.

> **Note:** Delete this section for the first phase of a project or when there are no prior dependencies.

---

## Why This Phase Is Different

*Optional but recommended for any phase that introduces new risk dimensions — a security boundary, a cross-language contract, a new build system, platform-specific behavior, or an irreversible operation. A short paragraph here shapes how the agent approaches the work before it reads anything else.*

---

# 1. Objective

## 1.1 Business Outcome

Describe the measurable outcome this phase must achieve. One to three sentences. What is different about the system when this is done?

## 1.2 Success Metric & Threshold

Define the quantitative bar that constitutes success. Both the agent and the human reviewer must have a clear pass/fail reference.

| Metric | Baseline (Current State) | Target Threshold | Verification Method |
|--------|--------------------------|------------------|---------------------|
|        |                          |                  |                     |

## 1.3 Current State Baseline

Describe the system state before this phase runs. This anchors post-implementation validation and makes regression detection unambiguous.

- What exists today that this phase changes:
- What must continue to work unchanged:
- Known constraints or context the agent needs:

---

# 2. Agent Persona & Operating Discipline

Define the agent's role and behavioral frame before it reads anything else. This shapes plan quality before implementation begins. The persona statement is not flavor text — it encodes the risk posture appropriate to this phase.

```
You are a [domain] agent working on [project context].
Your operating discipline is: read first, plan before writing, fail closed on ambiguity.

You do not [key prohibition specific to this phase's risk profile].
You do not make decisions the spec left open — you surface them to the human.
You emit a written plan before touching any file.
```

*Calibrate the persona to the phase type:*
- *Pure refactor:* "preserve all existing behavior exactly, fail closed on any ambiguity about what the original code does"
- *Security-sensitive:* "security-first, never guess at platform behavior, defensive error handling on all privileged paths"
- *Cross-language boundary:* "read-first, confirm contracts before implementing either side, never assume type serialization"
- *Frontend / visual:* "preserve all logic, change only what the spec explicitly authorizes, escape all user-controlled data before DOM insertion"
- *Build / delivery:* "narrow scope, surface errors rather than fixing them autonomously, do not modify source files"

---

# 3. Invariants (Must Always Be True)

These constraints are non-negotiable and must hold after this phase is complete. The agent must verify each before marking the task done.

- [ ] [Behavioral invariant — e.g. "existing CLI behavior is identical before and after"]
- [ ] [Scope invariant — e.g. "no files outside the declared allowed operations table are modified"]
- [ ] [Safety invariant — e.g. "ProgressFunc is nilable — calling with nil must not panic"]
- [ ] [Test invariant — e.g. "go test ./... passes throughout — no regressions at any point"]
- [ ] [Contract invariant — e.g. "all exported methods intended to be JS-callable are capitalized"]

*Add phase-specific invariants. Delete the examples above.*

---

# 4. Explicit Constraints (What Is Forbidden)

These are hard stops, not guidelines. The agent must not cross these boundaries regardless of what seems locally reasonable.

- [ ] Do not modify [files / packages / systems outside declared scope]
- [ ] Do not [action that seems helpful but is out of phase scope]
- [ ] Do not attempt to fix errors outside this phase autonomously — surface them to human
- [ ] Do not add dependencies not already declared in this spec
- [ ] Do not proceed under ambiguity — halt and surface to human

## 4.1 Declared Allowed File Operations

Every file the agent may create or modify must be listed here. Operations on any unlisted file are forbidden.

| Operation | Target | Justification |
|-----------|--------|---------------|
| Create    |        |               |
| Modify    |        |               |
| Read      |        |               |
| Delete    |        |               |

---

# 5. Inputs

List all inputs this phase requires, their types, and validation rules.

| Input | Type | Required | Validation Rule |
|-------|------|----------|-----------------|
|       |      |          |                 |

- Configuration or environment assumptions:
- Secrets handling (reference by name only — never inline secrets in specs):
- Preconditions on input state:

> **Delete this section** if the phase produces no inputs distinct from the codebase itself (e.g. a pure refactor with no runtime inputs).

---

# 6. Outputs

Define exact output shapes, destinations, and error contracts.

## 6.1 Success Output

*Describe the artifact, return type, file, or observable state change that constitutes successful output. Use a code block or schema if the output has a defined structure.*

```
[output schema or description]
```

## 6.2 Error Output

*Describe what the agent produces on failure — error type, wrapping convention, partial result handling.*

- On failure: [what is returned or emitted]
- Partial results: [returned or not returned]
- Error wrapping convention: [e.g. "errors are wrapped with stage context: 'stage failed: <underlying error>'"]

## 6.3 Output Destination

- Where does output land:
- Who or what consumes it:

> **Delete this section** for phases whose output is purely a code change with no distinct runtime output contract (e.g. a visual enhancement or pure refactor). Keep it for any phase that produces a function return value, file, event, API response, or structured artifact consumed downstream.

---

# 7. Execution Model

Describe how the agent must sequence its work. Execution is always staged — no monolithic runs.

- [ ] Planning phase required — agent must produce a written plan artifact before any file is written or action taken
- [ ] Tasks execute in declared order — agent must not reorder
- [ ] Each task must verify its gate conditions before proceeding to the next
- [ ] Human approval gate required before: [list any stages requiring human approval, or "none"]
- [ ] On failure: surface to human rather than attempting autonomous fixes outside phase scope

## 7.1 Task Sequence

| Task | Description | Gate to Start | Gate to End | Human Approval? |
|------|-------------|---------------|-------------|:---------------:|
| 1    |             |               |             |                 |
| 2    |             |               |             |                 |

> **Simplify or delete this section** for single-task phases. Keep it whenever the phase has meaningful internal sequencing or multiple human gates.

---

# 8. Rollback Requirements

Every forward action must have a defined inverse. The agent must know the rollback sequence before it takes the first action.

- [ ] Each forward action has a defined inverse before execution begins
- [ ] Rollback executes in reverse dependency order
- [ ] Rollback is idempotent
- [ ] If rollback fails, agent halts and surfaces to human — no silent partial rollbacks
- [ ] Post-rollback state is verified

## 8.1 Rollback Map

| Forward Action | Inverse Action | Dependency Order |
|----------------|----------------|------------------|
|                |                |                  |

---

# 9. Verification & Observability

Define how correctness is confirmed. Calibrate to the phase type — not every phase needs telemetry, but every phase needs a defined correctness signal.

## 9.1 Automated Checks

```bash
# List the exact commands that must pass before the phase is considered complete
```

## 9.2 Behavioral Equivalence Check

*Include for refactor phases. Describe how to confirm behavior is unchanged before and after.*

> **Delete this section** for non-refactor phases.

## 9.3 Human Verification Required

*List anything that cannot be verified automatically — OS-level smoke tests, visual checks, integration tests that require a running environment.*

| Check | Steps | Pass Condition |
|-------|-------|----------------|

> **Delete this section** if everything is automated.

## 9.4 Observability (If Applicable)

*Include for phases that produce runtime artifacts with logging or telemetry requirements.*

- Execution ID: [how it is generated and carried through stages]
- Pre/post state capture: [what is captured and where]
- Log format: [structured JSON or other]
- Log destination: [file, stdout, service]

> **Delete this section** for implementation phases with no distinct runtime observability requirements.

---

# 10. Validation Scenarios

**This is a first-class section, not an afterthought.** Scenarios are defined before implementation begins. The agent sees the visible set. The holdout set is withheld until post-implementation evaluation.

## 10.1 Visible Scenarios

These scenarios are available to the agent during implementation for self-validation.

| ID | Name | Preconditions | Action | Expected Outcome | Satisfaction Criteria |
|----|------|---------------|--------|------------------|-----------------------|
| VS-001 | | | | | |

## 10.2 Holdout Scenarios (Human-Controlled)

**These are not shared with the agent.** Evaluated by human after implementation. If the agent sees this section, something went wrong with how the spec was provided.

| ID | Name | Preconditions | Action | Expected Outcome | What Human Verifies |
|----|------|---------------|--------|------------------|---------------------|
| HS-001 | | | | | |

## 10.3 Satisfaction Metric

> Target: ___% of visible scenarios pass. ___% of holdout scenarios pass.
>
> *For refactor phases: 100% is the only acceptable number — behavioral equivalence is binary.*
> *For new feature phases: define the fraction and the rationale.*

---

# 11. Testing Philosophy

Testing validates behavior and invariants — not just whether a test suite passes.

- [ ] Tests validate invariants explicitly, not just happy paths
- [ ] Tests validate idempotency where applicable — run twice, confirm same end state
- [ ] Tests validate failure handling at each declared failure mode
- [ ] Tests validate rollback correctness and post-rollback state where applicable
- [ ] Do not optimize implementation to pass only the visible scenarios
- [ ] Holdout scenarios exist and will be evaluated separately
- [ ] A passing test suite is necessary but not sufficient — satisfaction rate (Section 10.3) is the primary bar

---

# 12. Failure Modes & Risk Assessment

For each failure class, define detection and agent response. Pre-seed with the classes most likely for this phase type, then add task-specific modes.

## 12.1 Phase-Type Failure Classes

*Select the pre-seeded set appropriate to this phase. Delete inapplicable rows. Add task-specific rows in 12.2.*

**Refactor phases:**

| Failure Class | Description | Detection | Agent Response | Escalate? |
|---------------|-------------|-----------|----------------|:---------:|
| Behavior regression | Post-refactor output differs from baseline | Behavioral equivalence check | Diagnose delta, restore from git if needed | Yes |
| Import boundary violation | New package accidentally imports from wrong layer | Build error or grep check | Fix import before proceeding | No |
| Test suite breaks | Pre-existing tests fail after modification | go test ./... or equivalent | Restore modified file from git, diagnose | Yes |
| Partial refactor state | File left in mid-replacement state | Code review | Complete or fully revert — no partial states | No |

**Security-sensitive phases:**

| Failure Class | Description | Detection | Agent Response | Escalate? |
|---------------|-------------|-----------|----------------|:---------:|
| Privilege boundary violation | Agent takes action outside declared scope | Pre-flight permission check | Fail closed, log, halt | Yes |
| Insecure temp file | World-readable file created with sensitive data | Permission check post-creation | Delete, recreate with correct permissions | Yes |
| Input not validated on receipt | Subprocess or handler trusts caller input blindly | Code review | Add validation on receipt, not just on send | No |
| Error path skips cleanup | Failure leaves sensitive artifacts behind | Code review of all error returns | Add deferred cleanup to all error paths | No |

**Cross-language / API boundary phases:**

| Failure Class | Description | Detection | Agent Response | Escalate? |
|---------------|-------------|-----------|----------------|:---------:|
| Contract field name mismatch | Field name differs between producer and consumer | Runtime failure / undefined | Read both sides before implementing either | No |
| Silent type coercion | Numeric or boolean type treated differently across boundary | Runtime failure | Verify serialization round-trip before wiring | No |
| Exported/unexported confusion | Method invisible to binding consumer | Runtime undefined error | Confirm export requirements before naming | No |

**Frontend / visual phases:**

| Failure Class | Description | Detection | Agent Response | Escalate? |
|---------------|-------------|-----------|----------------|:---------:|
| XSS via unescaped data | Network-sourced string injected into innerHTML raw | Code review / manual test | Wrap in escapeHtml before every innerHTML assignment | No |
| Runtime timing violation | API called before host environment has injected it | Runtime error in target environment | Use correct initialization event for the platform | No |
| Scope creep into logic | Visual change modifies behavior unintentionally | Regression test | Revert logic change, limit to presentation only | No |
| Element ID / class rename breaks JS | CSS or HTML change breaks JS references | Runtime failure | Treat all JS-referenced IDs and classes as invariants | No |

## 12.2 Task-Specific Failure Modes

| Failure Class | Description | Detection | Agent Response | Escalate? |
|---------------|-------------|-----------|----------------|:---------:|
|               |             |           |                |           |

---

# 13. Non-Goals

Explicitly state what is out of scope. Prevents the agent from doing things that seem helpful but belong in a different phase.

- [Feature or capability that is intentionally deferred]
- [Adjacent improvement the agent might reasonably attempt]
- [Known open question that is explicitly not resolved in this phase]

---

# 14. Agent Planning Instructions

**The agent must complete all steps below and produce a written plan artifact before writing any file or taking any action. No exceptions.**

1. **Read** [specific files the agent must read before writing anything — be exact about filenames and what to confirm]
2. **Confirm** [specific contracts, signatures, or assumptions that must be verified from the code before proceeding]
3. **State** [the specific decision or approach before implementing it — e.g. "state the IPC protocol", "state the exact replacement block for main.go"]
4. **List** all invariants from Section 3 and confirm each is achievable given what you have read
5. **List** all failure modes from Section 12 and state how each is addressed
6. **State implementation order** — which files are written first and why
7. **Only then** begin implementation

> If reading the codebase reveals that any assumption this spec makes is incorrect — a function signature doesn't match, a file doesn't exist, a contract is different than described — **halt and surface the discrepancy to the human before proceeding.** Do not work around it.

*The planning instructions above are generic. Replace steps 1–3 with the specific reading and confirmation steps for this phase. The more specific these are, the more they prevent the agent from skipping assumptions.*

---

# 15. Compliance & Audit Context

*Include for phases operating in regulated environments (PCI, HIPAA, SOC2, internal compliance frameworks). Delete for standard engineering phases.*

- Applicable standards or frameworks:
- Data classification of artifacts produced:
- Audit scope — what must be logged and retained:
- Approval or sign-off required before execution:
- Change control reference:

---

# 16. Known Shortcuts (Explicitly Documented)

*Include for rapid iteration or prototype phases where you are intentionally accepting technical debt. Explicitly documenting shortcuts prevents them from becoming permanent by accident. Delete for production phases.*

| Shortcut | What Is Deferred | Acceptable Because | Must Be Resolved Before |
|----------|------------------|--------------------|-------------------------|
|          |                  |                    |                         |

---

# 17. Open Questions

*List any decisions that were not resolved before this spec was written. These must be resolved before the agent begins. Either answer them here, or make them explicit pre-flight blockers.*

| Question | Why It Matters | Resolution | Resolved By |
|----------|---------------|------------|-------------|
|          |               |            |             |

> **An open question in this section is a blocker.** The agent must not proceed past the pre-flight gate with unresolved questions that affect implementation decisions.

---

# 18. Change Log

| Date | Version | Change | Author |
|------|---------|--------|--------|
|      | 1.0     | Initial spec | |
|      |         |        |        |

---

*End of Specification*
