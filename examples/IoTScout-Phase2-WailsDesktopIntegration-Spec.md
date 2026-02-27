# Agentic Development Specification
## IoT Scout — Phase 2: Wails Desktop Integration

> Version: 1.0  
> Created: 2026-02-23  
> Project: iot-scout  
> Phase: 2 of 4  
> Depends on: Phase 0 and Phase 1 complete and merged  
> Source Plan: 2026-02-12-desktop-ui-transformation-v2.md

---

## ⚠️ Pre-Flight Gate

**This phase must not begin until all of the following are verified:**

- [ ] `src/pkg/scanner/pipeline.go` exists, exports `Config`, `Result`, `Run`, `ProgressFunc`, `ProgressStage`
- [ ] `src/pkg/elevate/elevate.go` exists, exports `NeedsElevation()` and `RunElevatedScan(cfg, progress)`
- [ ] `src/cmd/iot-scout/elevated.go` exists with hidden `--internal-elevated-scan` command
- [ ] `go test ./...` passes with no failures
- [ ] `go build ./cmd/iot-scout` succeeds
- [ ] Phase 1 human smoke tests passed on the development platform

If any gate is not met, halt and surface to human.

---

## Why Phase 2 Has a Different Risk Profile

Phase 0 was pure Go with deterministic correctness. Phase 1 was security-sensitive with platform forks. Phase 2 adds three new dimensions:

**1. A new build system.** Once Wails is integrated, `go build` no longer produces a working binary for `iot-scout gui`. `wails build` must be used. The agent must understand this distinction and not conflate them. `go build` remains valid for verifying compilation and running tests; `wails build` is required for a working desktop binary.

**2. A cross-language API contract.** The exported methods on `DesktopApp` become JavaScript-callable via Wails bindings. Method names, parameter types, and return types must be exact — mismatches don't produce compile errors, they produce silent runtime failures in the UI. The binding contract is this spec's most important output.

**3. Three logically independent tasks with strict ordering.** Task 3 (dependency), Task 4 (Go backend), and Task 5 (model change) are sequenced so each gates the next. The agent must not reorder them.

---

# 1. Objective

## 1.1 Business Outcome

Add a `gui` subcommand to the existing `iot-scout` binary that opens a native desktop window. The CLI (`iot-scout scan`) must continue to work identically. The single binary serves both modes.

## 1.2 Success Metric & Threshold

| Metric | Baseline | Target | Verification |
|--------|----------|--------|--------------|
| `iot-scout scan` works identically post-integration | Works | Still works, behavior unchanged | `go test ./...` + human CLI smoke test |
| `iot-scout gui` opens a native window | Doesn't exist | Native OS window opens, title "IoT Scout" | Human smoke test |
| `DetectNetwork()` returns preferred subnet | Doesn't exist | Returns CIDR of primary private interface | Unit test + human smoke test |
| `StartScan()` returns report without error on a live network | Doesn't exist | Returns `ScanResult` with non-nil `Report` | Human smoke test |
| `Finding` model backward compatible | Has `Severity`, `Category`, `Message` | Adds `HumanMessage`, `RiskLevel` with `omitempty` | Unit test verifying JSON omission when empty |
| `go test ./...` regression | Passes | Still passes | CI |

## 1.3 Current State Baseline

- No Wails dependency in `go.mod`
- No `wails.json`, no `frontend/` directory
- No `gui.go`, no `app.go`
- `models.Finding` has three fields: `Severity`, `Category`, `Message`
- CLI scan pipeline is fully operational via Phase 0
- Elevation is fully operational via Phase 1

---

# 2. Agent Persona & Operating Discipline

```
You are a Go systems engineering agent integrating a desktop UI framework
into an existing CLI tool.

Your operating discipline: read-first, strict task ordering, never modify
existing packages. The CLI (main.go, scanner, elevate) is proven and must
not be touched. You are adding new files, not refactoring existing ones.

You understand two distinct build modes:
  - `go build ./cmd/iot-scout` → verifies Go compilation, runs tests
  - `wails build` → produces a working desktop binary with embedded frontend

You do not substitute one for the other. You do not claim "build succeeded"
if only `go build` passed and the task requires `wails build`.

You implement tasks in strict order: Task 3 → Task 4 → Task 5.
You do not begin a task until the previous task's verification step passes.
```

---

# 3. Invariants (Must Always Be True)

- [ ] `iot-scout scan --subnet x.x.x.x/24` produces identical output before and after this phase
- [ ] `go test ./...` passes throughout — no regressions at any point
- [ ] Existing packages (`pkg/scanner`, `pkg/elevate`, `pkg/discovery`, `internal/models`) are not modified except for the explicit `models.Finding` change in Task 5
- [ ] All `DesktopApp` methods that are intended to be JS-callable are exported (capitalized)
- [ ] All `DesktopApp` helper functions that must NOT be JS-callable are unexported (lowercase)
- [ ] The `startup` method signature matches exactly what Wails requires: `func (a *DesktopApp) startup(ctx context.Context)`
- [ ] `StartScan` never panics — errors are returned as `ScanResult{Error: "..."}`, not propagated as Go errors
- [ ] `StartScan` is concurrency-safe — the `sync.Mutex` guard prevents overlapping scans
- [ ] Progress events during elevated scans show an indeterminate state, not a determinate progress bar (Phase 1 known limitation — see Section 7.2)
- [ ] `HumanMessage` and `RiskLevel` fields on `Finding` are absent from JSON when empty (`omitempty`)

---

# 4. Explicit Constraints (What Is Forbidden)

- [ ] Do not modify `src/pkg/scanner/pipeline.go` — read-only this phase
- [ ] Do not modify `src/pkg/elevate/elevate.go` or its platform files — read-only this phase
- [ ] Do not modify `src/cmd/iot-scout/main.go` — read-only this phase
- [ ] Do not modify `src/cmd/iot-scout/elevated.go` — read-only this phase
- [ ] Do not add a Node.js build step — the frontend is vanilla JS with no build pipeline
- [ ] Do not embed the frontend via a path outside `src/cmd/iot-scout/frontend/` — the `//go:embed` directive must resolve relative to `gui.go`'s location
- [ ] Do not attempt to pass progress callbacks through to the elevated subprocess — this is a documented known limitation, not a bug to fix
- [ ] Do not add any JS frameworks (React, Vue, etc.) or npm packages — vanilla HTML/CSS/JS only
- [ ] Do not make `StartScan` return a Go `error` — it must return `*ScanResult` only, with errors embedded in the struct

## 4.1 Declared Allowed File Operations

| Operation | Target | Justification |
|-----------|--------|---------------|
| Modify | `src/go.mod` / `src/go.sum` | Add Wails v2 dependency |
| Create | `src/cmd/iot-scout/wails.json` | Wails build config |
| Create | `src/cmd/iot-scout/frontend/` | Empty directory placeholder for Task 3 |
| Create | `src/cmd/iot-scout/gui.go` | Cobra `gui` command + Wails init |
| Create | `src/cmd/iot-scout/app.go` | Wails-bound Go methods |
| Create | `src/cmd/iot-scout/app_test.go` | Unit tests for helpers |
| Modify | `src/internal/models/types.go` | Add `HumanMessage`, `RiskLevel` to `Finding` |
| Create/Modify | `src/internal/models/models_test.go` | Test `omitempty` behavior |

No other files may be created or modified.

---

# 5. Task 3: Wails Dependency and Config

**Gate to start this task:** Pre-flight gate passed.  
**Gate to end this task:** `go build ./cmd/iot-scout` succeeds with Wails in `go.mod`.

## 5.1 Add Wails Dependency

```bash
cd src && go get github.com/wailsapp/wails/v2@latest
```

Verify that `go.mod` now contains a `require github.com/wailsapp/wails/v2` entry.

## 5.2 Create wails.json

Path: `src/cmd/iot-scout/wails.json`

```json
{
  "$schema": "https://wails.io/schemas/config.v2.json",
  "name": "IoT Scout",
  "outputfilename": "iot-scout",
  "frontend:dir": "frontend",
  "frontend:install": "",
  "frontend:build": "",
  "author": {
    "name": "Robert"
  }
}
```

Empty `frontend:install` and `frontend:build` are intentional — no Node.js build step.

## 5.3 Create Empty Frontend Directory

Create `src/cmd/iot-scout/frontend/.gitkeep` (empty file) so the directory is tracked by git and the `//go:embed all:frontend` directive in `gui.go` does not fail on an empty directory.

## 5.4 Verification

```bash
cd src && go build ./cmd/iot-scout
# Must succeed. Binary may not fully work yet (gui command not added), but must compile.
```

## 5.5 Commit

```bash
git add src/go.mod src/go.sum src/cmd/iot-scout/wails.json src/cmd/iot-scout/frontend/.gitkeep
git commit -m "feat: add Wails v2 dependency and build config"
```

---

# 6. Task 4: Go Backend Binding Layer

**Gate to start this task:** Task 3 verification passed.  
**Gate to end this task:** `go test ./cmd/iot-scout/... -run TestIsPrivateIP -run TestPreferSubnet -run TestHumanizeFinding` passes AND `go build ./cmd/iot-scout` succeeds.

## 6.1 The Wails Binding Model

Wails uses reflection to expose exported methods of bound structs to JavaScript. The JavaScript caller invokes them as:

```javascript
window.go.main.DesktopApp.MethodName(arg1, arg2)
```

**Rules that apply to bound methods:**
- Must be exported (capitalized)
- May return `(T, error)` or just `T` — Wails handles both
- Must not use types that don't JSON-serialize cleanly (channels, function values, etc.)
- The `startup(ctx context.Context)` lifecycle hook is lowercase and handled specially by Wails via `OnStartup` — it is NOT JS-callable

**Unexported helpers on `DesktopApp` are not bound.** `isPrivateIP`, `preferSubnet`, and `humanizeFinding` must remain unexported.

## 6.2 Create gui.go

Path: `src/cmd/iot-scout/gui.go`

```go
// src/cmd/iot-scout/gui.go
package main

import (
	"embed"

	"github.com/spf13/cobra"
	"github.com/wailsapp/wails/v2"
	"github.com/wailsapp/wails/v2/pkg/options"
	"github.com/wailsapp/wails/v2/pkg/options/assetserver"
)

//go:embed all:frontend
var frontendAssets embed.FS

var guiCmd = &cobra.Command{
	Use:   "gui",
	Short: "Launch desktop UI",
	Long:  "Open IoT Scout as a native desktop application",
	RunE:  runGUI,
}

func init() {
	rootCmd.AddCommand(guiCmd)
}

func runGUI(cmd *cobra.Command, args []string) error {
	app := NewDesktopApp()

	return wails.Run(&options.App{
		Title:     "IoT Scout",
		Width:     1200,
		Height:    800,
		MinWidth:  800,
		MinHeight: 600,
		AssetServer: &assetserver.Options{
			Assets: frontendAssets,
		},
		BackgroundColour: &options.RGBA{R: 249, G: 250, B: 251, A: 1},
		OnStartup:        app.startup,
		Bind: []interface{}{
			app,
		},
	})
}
```

## 6.3 Create app.go

Path: `src/cmd/iot-scout/app.go`

Implement exactly as specified in the plan (lines 732–977 of the source plan). Key points the agent must verify match this spec:

**Types exported to JS:**

| Type | Fields | JSON keys |
|------|--------|-----------|
| `SubnetInfo` | `CIDR`, `InterfaceName`, `IP`, `IsPreferred` | `cidr`, `interfaceName`, `ip`, `isPreferred` |
| `NetworkInfo` | `Subnets []SubnetInfo`, `Preferred *SubnetInfo` | `subnets`, `preferred` |
| `ScanResult` | `Report *report.ScanReport`, `Error string` | `report`, `error` (omitempty) |

**Bound methods (JS-callable):**

| Method | Signature | Returns |
|--------|-----------|---------|
| `DetectNetwork` | `func (a *DesktopApp) DetectNetwork() (*NetworkInfo, error)` | NetworkInfo or error |
| `StartScan` | `func (a *DesktopApp) StartScan(subnet string, allowDefaultCreds bool) *ScanResult` | ScanResult always (errors embedded) |

**Note on `StartScan` return type:** `StartScan` returns `*ScanResult`, not `(*ScanResult, error)`. Errors are embedded in `ScanResult.Error`. This is intentional — it simplifies JavaScript error handling. The agent must not change this to a two-value return.

**Unexported helpers (not JS-callable):**

| Function | Signature |
|----------|-----------|
| `startup` | `func (a *DesktopApp) startup(ctx context.Context)` |
| `isPrivateIP` | `func isPrivateIP(ip net.IP) bool` |
| `preferSubnet` | `func preferSubnet(subnets []SubnetInfo) *SubnetInfo` |
| `humanizeFinding` | `func humanizeFinding(f models.Finding) models.Finding` |

## 6.4 Scan Config Defaults

`StartScan` hardcodes these scan defaults. The agent must not alter them:

```go
cfg := scanner.Config{
    Subnet:                 subnet,
    Ports:                  []int{22, 23, 80, 443, 554, 1883, 1900, 5353, 8000, 8080, 8443, 8883},
    Timeout:                5 * time.Second,
    Concurrency:            100,
    AllowDefaultCredsCheck: allowDefaultCreds,
}
```

## 6.5 Event Names

Wails events emitted by `StartScan` must use exactly these event names — the frontend JavaScript in Phase 3 depends on them:

| Event | Payload fields |
|-------|----------------|
| `scan:progress` | `{ progress: float64, deviceCount: int, message: string }` |

## 6.6 Elevated Scan Progress Behavior

When `elevate.NeedsElevation()` is true, `StartScan` must:
1. Emit one `scan:progress` event with `progress: 0.0`, `message: "Requesting permissions..."`
2. Call `elevate.RunElevatedScan(cfg, nil)` — nil progress is correct and expected
3. Emit one `scan:progress` event with `progress: 0.05`, `message: "Scanning (elevated)..."` — this is the only progress the UI receives during an elevated scan

The UI will show an indeterminate state. This is the documented known limitation from Phase 1 (Section 7.2 of the Phase 1 spec). **Do not attempt to add progress IPC between parent and subprocess.** That is future work.

## 6.7 Create app_test.go

Path: `src/cmd/iot-scout/app_test.go`

Implement exactly the tests specified in the plan (lines 984–1061). Tests cover:
- `TestIsPrivateIP` — all five IP cases
- `TestPreferSubnet` — named interface preference
- `TestPreferSubnetFallback` — unnamed interface fallback
- `TestHumanizeFinding` — all four severity cases

**Note:** `DetectNetwork` and `StartScan` cannot be unit-tested in this package because they depend on Wails runtime context. The helper function tests are sufficient for unit coverage. Human smoke tests (Section 10.2) cover the bound methods.

## 6.8 Verification

```bash
cd src
go test ./cmd/iot-scout/... -run TestIsPrivateIP -run TestPreferSubnet -run TestHumanizeFinding -v
go build ./cmd/iot-scout
go test ./...
```

All must pass.

## 6.9 Commit

```bash
git add src/cmd/iot-scout/gui.go src/cmd/iot-scout/app.go src/cmd/iot-scout/app_test.go
git commit -m "feat: add gui cobra command and Wails-bound DesktopApp backend"
```

---

# 7. Task 5: Add Humanized Fields to models.Finding

**Gate to start this task:** Task 4 verification passed.  
**Gate to end this task:** `go test ./...` passes with new `TestFindingOmitempty` test included.

## 7.1 Modify models/types.go

Add two fields to the `Finding` struct:

```go
type Finding struct {
    Severity     string `json:"severity"`
    Category     string `json:"category"`
    Message      string `json:"message"`
    HumanMessage string `json:"humanMessage,omitempty"`
    RiskLevel    string `json:"riskLevel,omitempty"`
}
```

**The `omitempty` tags are mandatory.** Without them, the CLI JSON output gains two empty fields on every finding, breaking backward compatibility for anyone parsing the existing output format.

## 7.2 Add/Update models_test.go

Add the following test. Create `src/internal/models/models_test.go` if it doesn't exist:

```go
package models_test

import (
    "bytes"
    "encoding/json"
    "testing"

    "github.com/robert/iot-scout/internal/models"
)

func TestFindingOmitempty(t *testing.T) {
    f := models.Finding{Severity: "HIGH", Category: "auth", Message: "bad"}
    data, err := json.Marshal(f)
    if err != nil {
        t.Fatal(err)
    }
    if bytes.Contains(data, []byte("humanMessage")) {
        t.Error("humanMessage should be omitted when empty")
    }
    if bytes.Contains(data, []byte("riskLevel")) {
        t.Error("riskLevel should be omitted when empty")
    }
}

func TestFindingPopulated(t *testing.T) {
    f := models.Finding{
        Severity:     "CRITICAL",
        Category:     "auth",
        Message:      "default creds",
        HumanMessage: "Factory default password accepted",
        RiskLevel:    "Fix now",
    }
    data, err := json.Marshal(f)
    if err != nil {
        t.Fatal(err)
    }
    if !bytes.Contains(data, []byte("humanMessage")) {
        t.Error("humanMessage should be present when populated")
    }
    if !bytes.Contains(data, []byte("riskLevel")) {
        t.Error("riskLevel should be present when populated")
    }
}
```

The plan only specifies `TestFindingOmitempty`. `TestFindingPopulated` is added here to verify the positive case as well.

## 7.3 Verification

```bash
cd src && go test ./internal/models/... -v && go test ./...
```

## 7.4 Commit

```bash
git add src/internal/models/types.go src/internal/models/models_test.go
git commit -m "feat: add humanized fields to Finding model with omitempty"
```

---

# 8. Inputs

| Input | Source | Used by |
|-------|--------|---------|
| `subnet string` | JS caller → `StartScan` | `scanner.Config.Subnet` |
| `allowDefaultCreds bool` | JS caller → `StartScan` | `scanner.Config.AllowDefaultCredsCheck` |
| `scanner.Result` | `elevate.RunElevatedScan` or `scanner.Run` | `report.GenerateJSON` |
| `models.Finding` | `scanner.Result.Devices[].Findings` | `humanizeFinding` |

---

# 9. Outputs

## 9.1 `DetectNetwork()` → `*NetworkInfo`

Returns a list of private-range network interfaces with one marked as preferred. The preferred selection heuristic (in priority order):
1. Interface named `en0`, `eth0`, `wlan0`, `wi-fi`, or `ethernet` (case-insensitive)
2. First interface with a `192.168.x.x` CIDR
3. First interface in the list

Returns `error` if no private interfaces are found.

## 9.2 `StartScan()` → `*ScanResult`

Returns `*ScanResult` always. On error, `ScanResult.Error` is populated and `ScanResult.Report` is nil. On success, `ScanResult.Report` is a `*report.ScanReport` with all devices and findings.

**Concurrent call behavior:** If a scan is already running, returns `&ScanResult{Error: "scan already running"}` immediately.

## 9.3 Wails Events

`StartScan` emits `scan:progress` events to the frontend. The frontend (Phase 3) registers for these via `window.runtime.EventsOn('scan:progress', callback)`.

---

# 10. Observability & Verification

## 10.1 Automated Checks

```bash
# After Task 3
go build ./cmd/iot-scout

# After Task 4
go test ./cmd/iot-scout/... -run TestIsPrivateIP -run TestPreferSubnet -run TestHumanizeFinding -v
go build ./cmd/iot-scout
go test ./...

# After Task 5
go test ./internal/models/... -v
go test ./...

# Cross-platform build check (all three tasks done)
GOOS=linux   GOARCH=amd64 go build ./cmd/iot-scout
GOOS=darwin  GOARCH=amd64 go build ./cmd/iot-scout
GOOS=windows GOARCH=amd64 go build ./cmd/iot-scout
```

## 10.2 Human Smoke Tests (Cannot Be Automated)

These require `wails dev` or `wails build` and actual execution. Phase 3 must not begin until these pass:

| Test | Steps | Pass Condition |
|------|-------|----------------|
| GUI window opens | `cd src/cmd/iot-scout && wails dev` | Native OS window appears with title "IoT Scout" |
| Network auto-detected | Window open | Network CIDR shown (e.g. `192.168.1.0/24 (en0)`) |
| CLI unchanged | `./iot-scout scan --subnet 192.168.1.0/24` | Identical output to pre-Phase-2 scan |
| `iot-scout gui` routes correctly | `./iot-scout gui` | Opens window, not scan output |
| `iot-scout scan` still routes correctly | `./iot-scout scan --subnet ...` | Runs CLI scan, no window |

---

# 11. Validation Scenarios

## 11.1 Visible Scenarios (Automated)

| ID | Name | Action | Expected |
|----|------|--------|----------|
| VS-001 | `isPrivateIP` — RFC1918 addresses | Call with `10.x`, `172.16.x`, `192.168.x` | Returns `true` |
| VS-002 | `isPrivateIP` — public addresses | Call with `8.8.8.8`, `1.1.1.1` | Returns `false` |
| VS-003 | `preferSubnet` — named interface | Subnets with `en0` and `utun0` | Returns `en0` subnet |
| VS-004 | `preferSubnet` — fallback | Subnets with no named interface | Returns first entry |
| VS-005 | `humanizeFinding` — CRITICAL | Severity `CRITICAL`, message `telnet...` | `RiskLevel = "Fix now"`, `HumanMessage` set |
| VS-006 | `humanizeFinding` — INFO | Severity `INFO`, generic message | `RiskLevel = "Looks fine"`, `HumanMessage = message` |
| VS-007 | `Finding.omitempty` — empty fields | Marshal `Finding` with no `HumanMessage`/`RiskLevel` | JSON contains neither key |
| VS-008 | `Finding.omitempty` — populated fields | Marshal fully populated `Finding` | JSON contains both keys |

## 11.2 Holdout Scenarios (Human-Controlled)

| ID | Name | What Human Verifies |
|----|------|---------------------|
| HS-001 | Scan result card count matches | Device cards rendered == `report.summary.total_devices` |
| HS-002 | Concurrent scan guard | Click Scan while scan is running → "scan already running" message appears, no crash |
| HS-003 | Elevated path shows indeterminate state | If not running as root, progress shows "Scanning (elevated)..." and doesn't advance past 5% during scan |
| HS-004 | CLI JSON output unchanged | `iot-scout scan --subnet x.x.x.x/24 --output-json` output has no `humanMessage` or `riskLevel` keys |

## 11.3 Satisfaction Metric

> All visible scenarios pass (100%). All holdout scenarios pass on the development platform. `wails dev` opens a working window before Phase 3 begins.

---

# 12. Failure Modes

| Failure Class | Description | Detection | Agent Response |
|---------------|-------------|-----------|----------------|
| Wails import fails to compile | `go.mod` Wails version incompatible with Go version | `go build` error | Surface exact error + Wails version to human |
| `//go:embed` fails | `frontend/` directory missing or empty | `go build` error | Ensure `.gitkeep` exists in `frontend/` |
| Bound method not accessible from JS | Lowercase method name accidentally used | Runtime JS error `undefined is not a function` | Ensure exported methods are capitalized |
| `startup` context nil | `app.ctx` used before Wails calls `OnStartup` | Nil pointer panic in `StartScan` | `startup` must be registered via `OnStartup`, not called manually |
| Scan progress emitted after context cancelled | User closes window mid-scan | Wails runtime panic | `runtime.EventsEmit` should check `a.ctx != nil` |
| `omitempty` omitted from model | Fields appear in CLI JSON output | `TestFindingOmitempty` fails | Add `omitempty` tags |
| `preferSubnet` returns nil | All interfaces filtered out | Nil pointer in JS | Ensure fallback always returns `&subnets[0]` when length > 0 |

---

# 13. Non-Goals

- The frontend UI itself — that is Phase 3
- Real-time progress during elevated scans — documented known limitation
- Code signing or packaging — Phase 4
- User-configurable scan ports or timeout in the GUI — hardcoded defaults in this phase
- `iot-scout` with no subcommand defaulting to `gui` — plan leaves this as a remaining question, not decided in this phase

---

# 14. Agent Planning Instructions

Before writing any file, the agent must produce a written plan covering:

1. **Read** `src/pkg/scanner/pipeline.go` and confirm the exact signatures of `Config`, `Result`, `Run`, `ProgressFunc`, and the stage constants — the `app.go` code depends on them
2. **Read** `src/pkg/elevate/elevate.go` and confirm the signatures of `NeedsElevation()` and `RunElevatedScan()`
3. **Read** `src/pkg/report/` to confirm the signature of `report.GenerateJSON()` and the shape of `report.ScanReport` — `ScanResult.Report` holds this type
4. **Read** `src/internal/models/types.go` to confirm the current `Finding` struct before modifying it
5. **State the implementation order**: Task 3 → verify → Task 4 → verify → Task 5 → verify
6. **List the bound methods and their exact Go signatures** before implementing `app.go`
7. **Confirm the embed path** — `gui.go` is at `src/cmd/iot-scout/gui.go`, the embed directive is `//go:embed all:frontend`, and the `frontend/` directory must be at `src/cmd/iot-scout/frontend/`
8. **State how the elevated scan progress limitation is handled** — confirm it is not "fixed" but handled per Section 6.6

> If reading any of the above reveals a signature mismatch or an unexpected type that would break the plan's code, halt and surface to human before proceeding.

---

# 15. Change Log

| Date | Version | Change | Author |
|------|---------|--------|--------|
| 2026-02-23 | 1.0 | Initial spec — Phase 2 Wails integration with three-task structure, build model disambiguation, binding contract table | |

---

*End of Specification — IoT Scout Phase 2*
