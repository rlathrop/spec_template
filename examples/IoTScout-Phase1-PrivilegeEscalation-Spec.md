# Agentic Development Specification
## IoT Scout — Phase 1: Privilege Escalation Helper

> Version: 1.0  
> Created: 2026-02-23  
> Project: iot-scout  
> Phase: 1 of 4  
> Depends on: Phase 0 complete and merged  
> Source Plan: 2026-02-12-desktop-ui-transformation-v2.md

---

## ⚠️ Pre-Flight Gate

**This phase must not begin until Phase 0 is verified complete:**

- [ ] `src/pkg/scanner/pipeline.go` exists and compiles
- [ ] `go test ./pkg/scanner/...` passes
- [ ] `go test ./...` passes with no regressions
- [ ] `iot-scout scan --subnet x.x.x.x/24` produces correct output

If any gate above is not met, halt and surface to human before proceeding.

---

## Why Phase 1 Is Different From Phase 0

Phase 0 was a pure refactor with a binary correctness condition. Phase 1 introduces three properties that require a different approach:

**1. Security boundary.** The subprocess model passes scan config as JSON args to an elevated child process. Getting this wrong in a security tool doesn't just break a feature — it creates a privilege escalation vulnerability. The spec must be more conservative than the plan in several places.

**2. Platform fork.** Three OS-specific elevation mechanisms (macOS `osascript`, Linux `pkexec`, Windows UAC) means three distinct failure modes. Unit tests cannot validate the OS dialog behavior — they can only validate the surrounding logic. The human gate for this phase is OS-level smoke testing, not just `go test`.

**3. Known ambiguity in the plan.** The plan explicitly notes: *"Windows elevation is trickier. Real implementation may need `golang.org/x/sys/windows` for proper `ShellExecuteW` call."* This spec resolves that ambiguity before implementation begins rather than deferring it to the agent.

**Approach:** Phase 1 is split into two sub-specs:
- **1A: The IPC contract** — `elevated.go`, the hidden subprocess command and temp file protocol
- **1B: The elevation mechanics** — the `pkg/elevate` package, platform by platform

This split means 1A can be fully unit-tested and verified before any platform-specific code is written. If 1A is wrong, 1B doesn't matter.

---

# 1. Objective

## 1.1 Business Outcome

Enable the desktop GUI (and optionally the CLI) to run ARP/ICMP scans without launching the entire application as root. The user sees a standard OS-native password/authorization dialog at scan time. The rest of the application runs unprivileged.

## 1.2 Success Metric & Threshold

| Metric | Baseline | Target | Verification |
|--------|----------|--------|--------------|
| Scan works when launched unprivileged | Not possible today without sudo | Full scan completes after OS dialog | Human smoke test on each target OS |
| Scan works when already root | Passes today with sudo | Continues to work — no double-elevation | Automated test + smoke test |
| No privilege retained after scan | N/A | Elevated subprocess exits after scan; parent remains unprivileged | Process inspection post-scan |
| `go test ./...` regression | Passes | Still passes | CI |
| Windows builds | Binary builds for windows/amd64 | Builds without errors | `GOOS=windows go build` |

## 1.3 Current State Baseline

- Scanning requires the user to launch `iot-scout` with `sudo` or as root
- No elevation mechanism exists
- `pkg/scanner/pipeline.go` exists (Phase 0 complete)
- No `pkg/elevate/` package exists
- No hidden subprocess command exists
- Windows elevation mechanism is unspecified in the plan — **this spec decides it** (see Section 6)

---

# 2. Agent Persona & Operating Discipline

```
You are a Go security engineering agent working on a cross-platform security tool.
Your operating discipline is: security-first, read before writing, never guess at
platform behavior.

For security-sensitive code you write defensively:
- Temp files must use restrictive permissions (0600)
- Config passed to subprocess must be validated on receipt, not just on send
- Error paths must clean up temp files even on failure
- No sensitive data in log output or error strings

When the plan is ambiguous (especially on Windows), you surface the ambiguity
and apply the resolution defined in this spec. You do not invent your own resolution.

You implement 1A fully before touching 1B.
```

---

# 3. Invariants (Must Always Be True)

These must hold after Phase 1 is complete.

- [ ] The parent process never runs the scan pipeline with elevated privileges — only the subprocess does
- [ ] Temp files created for IPC are created with mode `0600` (owner read/write only)
- [ ] Temp files are deleted by the parent after reading, even on error paths
- [ ] The hidden `--internal-elevated-scan` command is not shown in `--help` output
- [ ] Config passed to the subprocess is re-validated inside the subprocess before use — no blind trust of args
- [ ] If elevation is declined by the user (OS dialog cancelled), the error is surfaced cleanly — no crash, no hang
- [ ] If already running as root/admin, `NeedsElevation()` returns false and the scan runs directly without spawning a subprocess
- [ ] `go test ./...` continues to pass on the development platform
- [ ] The binary still compiles for linux/amd64, darwin/amd64, and windows/amd64

---

# 4. Explicit Constraints (What Is Forbidden)

- [ ] Do not pass credentials, API keys, or secrets through subprocess args or temp files
- [ ] Do not run the entire application as root — elevation is scoped to the scan subprocess only
- [ ] Do not show the `--internal-elevated-scan` flag in help output (`Hidden: true` on the cobra command)
- [ ] Do not use world-readable temp files — mode must be `0600`
- [ ] Do not implement a custom password dialog — use OS-native mechanisms only
- [ ] Do not add the `golang.org/x/sys` dependency unless strictly required (see Windows resolution, Section 6)
- [ ] Do not modify `pkg/scanner/pipeline.go` — it is complete and must not be changed in this phase
- [ ] Do not implement the GUI in this phase — `app.go` and `gui.go` are Phase 2

## 4.1 Declared Allowed File Operations

| Operation | Target | Justification |
|-----------|--------|---------------|
| Create | `src/pkg/elevate/elevate.go` | Platform-agnostic interface |
| Create | `src/pkg/elevate/elevate_darwin.go` | macOS elevation |
| Create | `src/pkg/elevate/elevate_linux.go` | Linux elevation |
| Create | `src/pkg/elevate/elevate_windows.go` | Windows elevation |
| Create | `src/pkg/elevate/elevate_test.go` | Unit tests |
| Create | `src/cmd/iot-scout/elevated.go` | Hidden subprocess command |
| Modify | `src/go.mod` / `src/go.sum` | Only if `golang.org/x/sys` is required for Windows (see Section 6) |

No other files may be created or modified.

---

# 5. Sub-Spec 1A: IPC Contract (elevated.go)

Implement and verify this before writing any platform-specific elevation code.

## 5.1 What It Does

`elevated.go` is a hidden cobra command that runs inside the elevated subprocess. It:
1. Receives scan config as JSON via `--scan-config` flag
2. Validates the config
3. Runs `scanner.Run()` with a 60-second timeout
4. Writes the result as JSON to the path specified by `--result-file`
5. Exits 0 on success, non-zero on any failure

The parent process in `elevate.go` reads the result file after the subprocess exits.

## 5.2 IPC Protocol Definition

This is the contract between parent and child. Both sides must implement it exactly.

**Parent → Child (via args):**
```
--internal-elevated-scan
--scan-config <json-encoded scanner.Config>
--result-file <absolute path to temp file>
```

**Child → Parent (via temp file):**
```json
{
  "Devices": [...],
  "StartTime": "RFC3339",
  "Duration": <nanoseconds as int64>
}
```

**Error case:** If the subprocess exits non-zero, the parent must treat it as an elevation or scan failure and return an error. The parent must NOT attempt to read the result file if the subprocess exited non-zero.

## 5.3 Config Validation in Subprocess

The subprocess must validate the received config before running the scan. Reject and exit non-zero if:
- `Subnet` is empty
- `Ports` is empty or nil
- `Timeout` is zero or negative
- `Concurrency` is zero or negative

This prevents a malformed or malicious config from being passed to the scanner in an elevated context.

## 5.4 Temp File Security Requirements

- Parent creates temp file with `os.CreateTemp("", "iot-scout-scan-*.json")`
- Parent closes the file handle immediately after creation (before passing path to child)
- Child writes to the path with `os.WriteFile(path, data, 0600)`
- Parent defers `os.Remove(tmpPath)` immediately after creating the temp file — runs even on error
- Parent reads the file only after subprocess exits 0

## 5.5 Subprocess Timeout

The subprocess runs with a 60-second context timeout. If the scan exceeds this, the subprocess exits with an error. This prevents an elevated subprocess from hanging indefinitely.

This timeout is hardcoded in the subprocess, not configurable via args. It is not the same as `scanner.Config.Timeout` which governs per-probe network timeouts.

---

# 6. Sub-Spec 1B: Platform Elevation Mechanics

**Prerequisite: Sub-Spec 1A implemented and `go test ./...` passing.**

## 6.1 Platform Decision: Windows

The plan notes the `cmd`/`runas` approach is simplified and may need `golang.org/x/sys/windows` for `ShellExecuteW`. **This spec resolves that:**

**Decision: Use `golang.org/x/sys/windows` for Windows elevation.**

Rationale: The `cmd /C exe args` approach does not reliably trigger UAC for GUI-aware applications and has well-known quoting vulnerabilities when args contain spaces or special characters (which JSON config strings do). `ShellExecuteW` with the `"runas"` verb is the correct and well-tested mechanism.

The agent must:
1. Add `golang.org/x/sys` to `go.mod` via `go get golang.org/x/sys`
2. Use `ShellExecuteW` in `elevate_windows.go`
3. Note: `ShellExecuteW` is async — the parent must wait for the subprocess to exit before reading the result file. Use a mechanism appropriate to the Windows process model (e.g. `WaitForSingleObject` via `golang.org/x/sys/windows`)

If the agent encounters unexpected complexity with the Windows implementation that cannot be resolved without further human input, it must halt and surface the specific blocker rather than substituting a weaker implementation.

## 6.2 macOS (`elevate_darwin.go`)

Use `osascript` with `do shell script ... with administrator privileges`. This is the standard macOS mechanism and shows the native authorization dialog.

**Known issue with the plan's code:** The plan's `osascript` quoting is incorrect — it would break on paths or args containing spaces. The agent must implement proper shell quoting. Use `fmt.Sprintf` with `%q` or shell-escape each argument individually. The correct pattern is:

```
do shell script "quoted-exe-path arg1 arg2 ..." with administrator privileges
```

Where the entire command string is safely constructed before being passed to `osascript -e`.

## 6.3 Linux (`elevate_linux.go`)

Use `pkexec`. This is straightforward and the plan's code is correct. One addition: `pkexec` may not be available on all Linux systems. The agent must check for `pkexec` availability with `exec.LookPath("pkexec")` before invoking it, and return a clear error if not found:

```
"pkexec not found — install policykit-1 or run iot-scout with sudo"
```

## 6.4 `NeedsElevation()` — Platform Behavior

- **Linux/macOS:** `os.Geteuid() != 0` — already correct in the plan
- **Windows:** `os.Geteuid()` always returns `-1` on Windows and is not meaningful. The Windows implementation must use `golang.org/x/sys/windows` to check if the current process has elevated privileges. The agent must implement this correctly and not use `Geteuid()` in `elevate_windows.go`.

---

# 7. Inputs

| Input | Type | Required | Notes |
|-------|------|----------|-------|
| `cfg scanner.Config` | struct | Yes | Full config from caller; validated in subprocess |
| `progress scanner.ProgressFunc` | func | No | Nil-safe; NOTE: progress callbacks do NOT work in subprocess mode — parent receives no progress events during elevated scan. This is a known limitation. |

**Progress limitation:** When `NeedsElevation()` is true, the subprocess runs `scanner.Run` with `nil` progress. The caller's `ProgressFunc` is not called during the scan. The GUI (Phase 2) must handle this gracefully — show a spinner or indeterminate state, not a progress bar, during elevated scans.

This limitation must be documented in a comment in `elevate.go` and is explicitly **not** solved in this phase. The plan notes this as a remaining question for a future release.

---

# 8. Outputs

`RunElevatedScan` returns `(*scanner.Result, error)` — identical signature to `scanner.Run`.

Callers cannot distinguish between a directly-run scan and an elevated scan from the return value. This is intentional.

**Error cases the caller must handle:**
- User cancelled the OS dialog → `"elevation cancelled"` or OS-specific error
- `pkexec` not found on Linux → clear message (see 6.3)
- Subprocess scan timeout → `"scan timed out"`  
- Subprocess exited non-zero → wrapped error with exit code
- Temp file unreadable or invalid JSON → wrapped error

---

# 9. Execution Model

**Mandatory order:**

1. Implement `elevated.go` (Sub-Spec 1A)
2. Write `elevate_test.go` unit tests for 1A-testable behavior
3. Run `go test ./...` — must pass before continuing
4. Implement `elevate.go` + platform files (Sub-Spec 1B)
5. Build for all three platforms
6. Human smoke test (Section 11.2)

The agent must not implement 1B before 1A is complete and tested.

---

# 10. Rollback Requirements

- [ ] Each new file can be individually deleted — no existing files are modified except optionally `go.mod`
- [ ] If `go.mod` is modified (Windows `golang.org/x/sys`), rollback is `go mod tidy` after removing the import

## 10.1 Rollback Map

| Forward Action | Inverse Action | Order |
|----------------|----------------|-------|
| Create `elevated.go` | `git rm src/cmd/iot-scout/elevated.go` | 1 |
| Create `elevate.go` | `git rm src/pkg/elevate/elevate.go` | 2 |
| Create platform files | `git rm src/pkg/elevate/elevate_*.go` | 3 |
| Create `elevate_test.go` | `git rm src/pkg/elevate/elevate_test.go` | 4 |
| Modify `go.mod` | `git checkout src/go.mod src/go.sum` + `go mod tidy` | 5 |

---

# 11. Observability & Verification

## 11.1 Automated Checks

```bash
# All platforms — unit tests
go test ./pkg/elevate/... -v
go test ./... 

# Cross-platform build verification
GOOS=linux   GOARCH=amd64 go build ./cmd/iot-scout
GOOS=darwin  GOARCH=amd64 go build ./cmd/iot-scout
GOOS=windows GOARCH=amd64 go build ./cmd/iot-scout

# Verify hidden command not in help
./iot-scout --help | grep -c "internal-elevated-scan"
# Expected: 0
```

## 11.2 Human Smoke Tests (Cannot Be Automated)

These require actual OS execution. The human must verify on each target platform:

| Test | Steps | Pass Condition |
|------|-------|----------------|
| Unprivileged scan triggers OS dialog | Launch `iot-scout gui` (Phase 2) or call `elevate.RunElevatedScan` from a test binary as non-root | OS-native password/auth dialog appears |
| User cancels dialog | Click Cancel on OS dialog | Returns error, no crash, no hang |
| User approves dialog | Enter correct password | Scan completes, results returned correctly |
| Already-root path | Run `sudo ./iot-scout scan ...` | Scan runs without spawning subprocess, no dialog |
| Temp file cleanup | Run scan, check `/tmp` for `iot-scout-scan-*` | No temp files remain after scan completes |

**Platform matrix:**

| Platform | Dialog Mechanism | Smoke Test Required |
|----------|-----------------|:-------------------:|
| macOS | `osascript` authorization dialog | Yes |
| Linux | `pkexec` polkit dialog | Yes |
| Windows | UAC / `ShellExecuteW runas` | Yes |

The human gate for this phase is completion of all rows of the platform matrix. Phase 2 must not begin until all smoke tests pass on the development platform.

---

# 12. Validation Scenarios

## 12.1 Visible Scenarios (Automated)

| ID | Name | Preconditions | Action | Expected Outcome |
|----|------|---------------|--------|-----------------|
| VS-001 | `NeedsElevation` returns false when root | Running as root/uid 0 | Call `NeedsElevation()` | Returns `false` |
| VS-002 | `NeedsElevation` returns true when unprivileged | Running as non-root | Call `NeedsElevation()` | Returns `true` |
| VS-003 | Config validation rejects empty subnet | `elevated.go` subprocess path | Pass `scanner.Config{Subnet: ""}` as JSON | Subprocess exits non-zero |
| VS-004 | Config validation rejects zero timeout | `elevated.go` subprocess path | Pass `scanner.Config{Timeout: 0}` | Subprocess exits non-zero |
| VS-005 | Hidden command absent from help | Binary built | `./iot-scout --help` | `internal-elevated-scan` not in output |
| VS-006 | Already-root takes direct path | Running as root | Call `RunElevatedScan` | `NeedsElevation()` false, scanner.Run called directly, no subprocess |

## 12.2 Holdout Scenarios (Human-Controlled)

| ID | Name | What Human Verifies |
|----|------|---------------------|
| HS-001 | Temp file permissions | After scan, inspect created/deleted temp file — mode was `0600`, not world-readable |
| HS-002 | Subprocess exits after scan | Process list shows no lingering elevated subprocess after scan completes |
| HS-003 | macOS quoting handles spaces in path | Install binary in path with spaces (e.g. `/Users/robert/My Tools/iot-scout`), run scan — osascript does not fail |
| HS-004 | Linux pkexec not found | Remove or rename `pkexec`, attempt scan — error message is human-readable, not a panic |
| HS-005 | JSON config with special characters | Subnet or other config containing `"`, `\`, or spaces — subprocess receives correct unescaped values |

## 12.3 Satisfaction Metric

> Security-critical path: all visible scenarios must pass (100%). All holdout scenarios must be verified by human on the development platform before Phase 2 begins.

---

# 13. Failure Modes & Risk Assessment

| Failure Class | Description | Detection | Agent Response | Escalate? |
|---------------|-------------|-----------|----------------|:---------:|
| Shell injection via JSON args | Malformed config JSON interpreted as shell commands in `osascript` | Code review of quoting logic | Use proper shell quoting, never interpolate raw strings | Yes |
| Temp file race condition | Another process reads temp file before subprocess writes | Unlikely in practice; temp file name is random | Acceptable risk; document limitation | No |
| Subprocess hang | Elevated subprocess never exits | 60-second timeout | Timeout kills subprocess; parent returns error | No |
| `pkexec` absent on Linux | Common on minimal/embedded Linux | `exec.LookPath` check | Return clear user-facing error | No |
| Windows UAC unavailable | UAC disabled by policy | `ShellExecuteW` error | Return error with message to run as Administrator | No |
| Double elevation | `NeedsElevation` returns true when already elevated on Windows | Wrong `NeedsElevation` implementation | Use `golang.org/x/sys/windows` token check, not `Geteuid` | Yes |
| Leftover temp files on crash | Parent crashes before `defer os.Remove` runs | N/A | Document as known limitation; temp files have random names and are small | No |

---

# 14. Non-Goals

- Real-time progress events from the elevated subprocess (noted in plan as future work)
- Persistent elevation (the subprocess exits after each scan)
- Code signing or notarization (deferred to future release per plan)
- The GUI (`app.go`, `gui.go`) — that is Phase 2
- Supporting `pkexec` alternatives on Linux (`sudo`, `gksudo`) — `pkexec` only for now
- Windows support for builds that don't have `golang.org/x/sys` available

---

# 15. Agent Planning Instructions

Before writing any file, the agent must produce a written plan covering:

1. **Read** `src/pkg/scanner/pipeline.go` — confirm `scanner.Config`, `scanner.Result`, and `scanner.Run` signatures match what this spec assumes
2. **Confirm** Phase 0 gate conditions are met (list them)
3. **State the IPC protocol** — write out exactly what the parent passes to the child and what the child writes to the temp file, before implementing either side
4. **State the Windows approach** — confirm `golang.org/x/sys/windows` will be used, identify the specific functions (`ShellExecuteW`, `WaitForSingleObject`, token elevation check)
5. **State the macOS quoting approach** — write out the exact quoting strategy before implementing `elevate_darwin.go`
6. **List all failure modes** from Section 13 and state how each is addressed
7. **Confirm the implementation order**: `elevated.go` first, then tests, then `go test ./...`, then elevation files
8. **Only then** begin implementation

> If reading `scanner.go` reveals that `scanner.Config` or `scanner.Result` are not JSON-serializable (e.g. contain unexported fields or non-serializable types), halt and surface to human before proceeding. The entire IPC contract depends on clean JSON round-trip of these types.

---

# 16. Change Log

| Date | Version | Change | Author |
|------|---------|--------|--------|
| 2026-02-23 | 1.0 | Initial spec — Phase 1 privilege escalation with 1A/1B split, Windows resolution, macOS quoting fix | |

---

*End of Specification — IoT Scout Phase 1*
