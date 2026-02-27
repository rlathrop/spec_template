# Agentic Development Specification
## IoT Scout — Phase 0: Extract Shared Scan Pipeline

> Version: 1.0  
> Created: 2026-02-23  
> Project: iot-scout  
> Phase: 0 of 4  
> Source Plan: 2026-02-12-desktop-ui-transformation-v2.md

---

# 1. Objective

## 1.1 Business Outcome

Extract the inline scan logic from `src/cmd/iot-scout/main.go` into a reusable `pkg/scanner/pipeline.go` package so that both the existing CLI and the forthcoming Wails desktop GUI can call a single shared pipeline without code duplication.

## 1.2 Success Metric & Threshold

| Metric | Baseline (Current State) | Target Threshold | Measurement Window |
|--------|--------------------------|------------------|--------------------|
| CLI scan behavior | Inline in `main.go:44-218` | Identical output before and after refactor | Single test run post-refactor |
| Test coverage on pipeline | 0% (no package exists) | All 4 pipeline stages covered by unit tests | Post-implementation |
| CLI regression | N/A | `go test ./...` passes with zero failures | Post-refactor |
| Binary behavior | `iot-scout scan --subnet x.x.x.x/24` produces JSON + HTML reports | Identical output, identical flags, identical report structure | Manual verification run |

This phase is complete when the CLI produces byte-for-byte equivalent output to today's behavior, `go test ./...` passes, and `pipeline.go` compiles cleanly with no references to `main.go` internals.

## 1.3 Current State Baseline

- Scan logic currently lives inline in `main.go` lines 44–218, inside `scanCmd.RunE`
- Pipeline stages (discovery → fingerprint → classify → security check) are not independently callable
- No shared abstraction exists — if GUI code were written today, it would duplicate this logic
- Existing packages (`discovery`, `fingerprint`, `classifier`, `checks`) are already well-separated; only the orchestration layer is missing
- All existing tests pass as of plan creation date

---

# 2. Agent Persona & Operating Discipline

```
You are a Go refactoring agent working on an open-source IoT security tool.
Your operating discipline is: read the existing code before writing anything,
preserve all existing behavior exactly, fail closed on any ambiguity about
what the original code does.

You do not rename exported types in existing packages.
You do not change CLI flag behavior or output format.
You do not add features — this phase is extraction only.
You emit a summary of what you read before writing any file.
```

---

# 3. Invariants (Must Always Be True)

These constraints must hold after this phase is complete. The agent must verify each before marking the task done.

- [ ] `iot-scout scan` CLI behavior is identical before and after — same flags, same stdout output, same JSON/HTML report structure
- [ ] No existing package APIs are changed (discovery, fingerprint, classifier, checks, report)
- [ ] `pkg/scanner/pipeline.go` has zero direct imports from `cmd/` — the pipeline is a library, not a binary
- [ ] `main.go` scan logic is fully delegated to `scanner.Run()` — no residual inline discovery/fingerprint/classify/check logic remains
- [ ] All existing tests continue to pass — `go test ./...` green
- [ ] The new pipeline compiles on all three target platforms (linux/amd64, darwin/amd64, windows/amd64)
- [ ] `ProgressFunc` is nilable — calling `scanner.Run()` with `nil` progress must not panic

---

# 4. Explicit Constraints (What Is Forbidden)

- [ ] Do not modify any file under `pkg/discovery`, `pkg/fingerprint`, `pkg/classifier`, `pkg/checks`, or `pkg/report`
- [ ] Do not change any cobra flag definitions in `main.go`
- [ ] Do not change stdout output format — the existing device printing loop stays in `main.go`
- [ ] Do not add features to the pipeline beyond what exists inline today (no new stages, no new options)
- [ ] Do not introduce any new external dependencies — use only packages already in `go.mod`
- [ ] Do not move or rename `main.go` — only replace the inline scan block
- [ ] Do not add a GUI, Wails, or elevation package in this phase — that is Phases 1–3

## 4.1 Declared Allowed File Operations

| Operation | Target | Justification |
|-----------|--------|---------------|
| Create | `src/pkg/scanner/pipeline.go` | New pipeline package |
| Create | `src/pkg/scanner/pipeline_test.go` | Unit tests for pipeline |
| Modify | `src/cmd/iot-scout/main.go` | Replace inline scan block with `scanner.Run()` call |
| Read | `src/cmd/iot-scout/main.go` | Read before writing |
| Read | `src/pkg/*/` | Understand existing package APIs before referencing them |

No other files may be created or modified in this phase.

---

# 5. Inputs

| Input | Type | Required | Validation Rule |
|-------|------|----------|-----------------|
| `cfg.Subnet` | `string` | Yes | Must be valid CIDR notation (e.g. `192.168.1.0/24`) |
| `cfg.Ports` | `[]int` | Yes | Must be non-empty; all values 1–65535 |
| `cfg.Timeout` | `time.Duration` | Yes | Must be > 0 |
| `cfg.Concurrency` | `int` | Yes | Must be > 0 |
| `cfg.Verbose` | `bool` | No | Defaults false |
| `cfg.AllowDefaultCredsCheck` | `bool` | No | Defaults false; gates opt-in credential checks |
| `progress` | `ProgressFunc` | No | May be nil — must be nil-safe |

- All inputs originate from cobra flags already validated in `main.go` — pipeline does not re-validate, but must not panic on malformed input that passes through
- No secrets or credentials are accepted as direct inputs
- Context (`ctx`) must be respected for cancellation throughout all stages

---

# 6. Outputs

## 6.1 Success Output

`scanner.Run()` returns `(*Result, error)`.

```go
type Result struct {
    Devices   []models.Device  // fully populated after all 4 stages
    StartTime time.Time        // captured at entry to Run()
    Duration  time.Duration    // time.Since(StartTime) at return
}
```

`main.go` continues to own report generation and stdout printing — the pipeline returns data only, not formatted output.

## 6.2 Error Output

`scanner.Run()` returns a non-nil `error` if any stage fails fatally. Errors are wrapped with stage context:

```
"discovery failed: <underlying error>"
"fingerprint failed: <underlying error>"  // if future stages gain hard-fail paths
```

Partial results are not returned on error — caller receives `nil, err`. Callers must check error before using result.

## 6.3 Output Destination

The pipeline writes no files and emits no stdout. All output is returned to the caller. Report generation and printing remain in `main.go`.

## 6.4 Structured Logging Requirements

This phase does not add structured logging — the existing `Verbose` flag behavior is preserved via the existing discovery package. Progress reporting is via the `ProgressFunc` callback only.

---

# 7. Execution Model

The pipeline executes four stages in strict sequential order. There is no parallelism between stages (parallelism within stages, e.g. concurrent port scanning, is handled by the existing packages and is unchanged).

- [ ] Stages execute in order: Discovery → Fingerprint → Classify → SecurityCheck
- [ ] Each stage calls `progress()` at entry (0%) and exit (100%) if `progress` is non-nil
- [ ] Context cancellation (`ctx.Done()`) must be respected — stages that accept context must forward it
- [ ] If discovery returns zero devices, subsequent stages run over an empty slice (not an error)
- [ ] If discovery returns an error, pipeline returns immediately — no subsequent stages run

## 7.1 Stage Sequence

| Stage | Description | Human Gate Required | Rollback Defined |
|-------|-------------|:-------------------:|:----------------:|
| 1. Read existing code | Agent reads `main.go:44-218` and all referenced packages before writing | No | N/A |
| 2. Create `pipeline.go` | Write new file per spec | No | Delete file |
| 3. Create `pipeline_test.go` | Write unit tests | No | Delete file |
| 4. Modify `main.go` | Replace inline scan block | No | Restore from git |
| 5. `go test ./...` | Run full test suite | No | Restore from git |
| 6. Manual smoke test | Build and run `iot-scout scan` | **Yes — human verifies output** | Restore from git |

---

# 8. Rollback Requirements

This phase touches only Go source files under version control. Rollback is `git checkout` of modified files and `git rm` of new files.

- [ ] If `go test ./...` fails after modifying `main.go`, restore `main.go` from git before surfacing error
- [ ] If the pipeline package fails to compile, delete `pipeline.go` and `pipeline_test.go` before surfacing error
- [ ] Do not leave `main.go` in a partially-refactored state — either the replacement is complete or it is fully reverted

## 8.1 Rollback Map

| Forward Action | Inverse Action | Order |
|----------------|----------------|-------|
| Create `pipeline.go` | `git rm src/pkg/scanner/pipeline.go` | 1 |
| Create `pipeline_test.go` | `git rm src/pkg/scanner/pipeline_test.go` | 2 |
| Modify `main.go` | `git checkout src/cmd/iot-scout/main.go` | 3 |

---

# 9. Observability Requirements

This is a pure refactor phase with no runtime observability changes. The following confirms the extraction was clean:

## 9.1 Verification Checkpoints

- `git diff --stat` after completion should show exactly 3 files changed/created: `pipeline.go`, `pipeline_test.go`, `main.go`
- `go vet ./...` must produce zero warnings
- `go test ./...` must produce zero failures
- `go build ./cmd/iot-scout` must succeed

## 9.2 Behavioral Equivalence Check

Run before and after the refactor and diff the outputs:

```bash
# Before: capture baseline output
./iot-scout scan --subnet 192.168.1.0/24 --output before.json

# After refactor: capture new output  
./iot-scout scan --subnet 192.168.1.0/24 --output after.json

# Diff (devices and findings should be identical; timestamps will differ)
jq 'del(.scan_time, .duration)' before.json > before_clean.json
jq 'del(.scan_time, .duration)' after.json > after_clean.json
diff before_clean.json after_clean.json
```

Expected: zero diff on device data, findings, and report structure.

---

# 10. Validation Scenarios

## 10.1 Visible Scenarios

| ID | Name | Preconditions | Action | Expected Outcome | Satisfaction Criteria |
|----|------|---------------|--------|------------------|-----------------------|
| VS-001 | CLI scan completes successfully | Valid subnet with at least 1 device | `iot-scout scan --subnet x.x.x.x/24` | Devices listed to stdout, JSON report written | stdout output matches pre-refactor format; report.json has same structure |
| VS-002 | nil progress does not panic | Any valid Config | `scanner.Run(ctx, cfg, nil)` | Returns result, no panic | No panic; result.Devices populated |
| VS-003 | Context cancellation is respected | Valid Config | Cancel ctx mid-scan | Returns ctx error, no hang | `ctx.Err()` returned within 1s of cancel |
| VS-004 | Empty subnet returns empty result | Valid CIDR with no live hosts | `scanner.Run(ctx, cfg, nil)` | Returns Result with empty Devices slice, no error | `len(result.Devices) == 0`, `err == nil` |
| VS-005 | All 4 progress stages fire | Valid Config with non-nil progress | `scanner.Run(ctx, cfg, progressFn)` | progressFn called for all 4 stages | Each of discovery, fingerprint, classify, security stages fires at 0% and 100% |

## 10.2 Holdout Scenarios (Human-Controlled)

These are not shared with the agent during implementation.

| ID | Name | Preconditions | Action | Expected Outcome | Satisfaction Criteria |
|----|------|---------------|--------|------------------|-----------------------|
| HS-001 | Default creds check opt-in gate | `AllowDefaultCredsCheck: false` | Run scan against device that would trigger creds check | No credential check attempted | No network auth attempts observed |
| HS-002 | Verbose flag still passes through | `Verbose: true` | `iot-scout scan --subnet x.x.x.x/24 --verbose` | Verbose output still appears | Pre- and post-refactor verbose output identical |
| HS-003 | Pipeline import boundary | N/A | `grep -r "cmd/iot-scout" src/pkg/scanner/` | Zero results | Pipeline package has no imports from cmd/ |

## 10.3 Satisfaction Metric

> Target: 100% of visible scenarios pass, 100% of holdout scenarios pass. This is a refactor — behavioral equivalence is binary, not probabilistic. Any deviation from pre-refactor behavior is a failure.

---

# 11. Testing Philosophy

- [ ] Tests must validate that `ProgressFunc` fires for all four stage transitions
- [ ] Tests must validate nil-safety of `ProgressFunc`
- [ ] Tests must validate that a cancelled context returns an error and does not hang
- [ ] Tests must validate that an empty device list flows through all stages without error
- [ ] Do not mock the existing packages — call them or stub at the network boundary only
- [ ] The test file is `pipeline_test.go` in `package scanner` (white-box) or `package scanner_test` (black-box) — agent's choice, but must be consistent
- [ ] `go test ./...` is the definitive pass/fail gate — not just `go test ./pkg/scanner/...`

---

# 12. Failure Modes & Risk Assessment

| Failure Class | Description | Detection Method | Agent Response | Human Escalation? |
|---------------|-------------|-----------------|----------------|:-----------------:|
| Import cycle | `pipeline.go` accidentally imports `cmd/` | `go build` error | Fix import, do not proceed | No |
| Broken inline extraction | `main.go` scan block not fully replaced — residual discovery/check calls remain | `grep -n "discovery\." main.go` after refactor | Complete the replacement | No |
| API mismatch | Pipeline calls existing package with wrong signature | `go build` compile error | Read package source before calling | No |
| Behavior regression | Post-refactor output differs from baseline | JSON diff (Section 9.2) | Diagnose delta, restore from git if needed | Yes |
| Test suite breaks | Pre-existing tests fail after `main.go` modification | `go test ./...` | Restore `main.go` from git, diagnose | Yes |
| ProgressFunc nil panic | Code calls `progress(...)` without nil check | Runtime panic in test VS-002 | Add nil guard before all progress calls | No |

---

# 13. Non-Goals

- Adding any new scan capabilities or security checks
- Implementing the Wails GUI or desktop window (Phase 1)
- Implementing privilege elevation (Phase 2)
- Adding structured logging or OpenTelemetry
- Changing JSON report schema
- Adding progress streaming over a pipe or socket (noted as a remaining question in the plan — explicitly deferred)
- Supporting any new platforms beyond the three already targeted

---

# 14. Agent Planning Instructions

Before writing a single line of code, the agent must produce a written plan that covers all of the following. Do not proceed to implementation until this plan is complete.

1. **Read** `src/cmd/iot-scout/main.go` lines 44–218 in full. Summarize what each inline stage does and which packages it calls.
2. **Read** the public APIs of `pkg/discovery`, `pkg/fingerprint`, `pkg/classifier`, `pkg/checks` — list every function signature the pipeline will call.
3. **Confirm** the `models.Device` struct fields referenced in the inline code — list them.
4. **List all invariants** from Section 3 and confirm each is achievable given what you've read.
5. **Identify all failure modes** from Section 12 and state how you will avoid each.
6. **State the exact replacement block** you will insert into `main.go` — write it out before touching the file.
7. **State the exact import changes** to `main.go` — what is added, what is removed.
8. **Only then** create `pipeline.go`, then `pipeline_test.go`, then modify `main.go`.
9. After modifying `main.go`, run `go build ./cmd/iot-scout` before running tests.
10. Run `go test ./...` and surface any failures before declaring done.

> If you read `main.go` and find that lines 44–218 do not match the plan's description, or if any package API does not exist as expected, **halt and surface the discrepancy** before proceeding.

---

# 15. Holdout Evaluation

## Implementation Phase
> Implement `pipeline.go` and `pipeline_test.go` per this spec. Modify `main.go` to call `scanner.Run()`. Verify visible scenarios (Section 10.1) pass. Run `go test ./...`. Run the behavioral equivalence check (Section 9.2).

## Holdout Evaluation Phase
> Human will verify holdout scenarios (Section 10.2) after implementation. Specifically:
> - Confirm no credential checks fire when `AllowDefaultCredsCheck` is false (HS-001)
> - Confirm verbose output is unchanged (HS-002)
> - Confirm pipeline package has no imports from cmd/ (HS-003)
>
> If any holdout scenario fails: identify the root cause, fix without weakening invariants, re-run `go test ./...`, re-verify behavioral equivalence. Do not mark Phase 0 complete until all holdout scenarios pass.

---

# 16. Change Log

| Date | Version | Change | Author |
|------|---------|--------|--------|
| 2026-02-23 | 1.0 | Initial spec for Phase 0 pipeline extraction | |

---

*End of Specification — IoT Scout Phase 0*
