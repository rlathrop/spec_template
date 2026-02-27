# Agentic Development Specification
## IoT Scout — Phase 4: Build & Verify

> Version: 1.0  
> Created: 2026-02-23  
> Project: iot-scout  
> Phase: 4 of 4  
> Depends on: Phases 0, 1, 2, and 3 complete and merged  
> Source Plan: 2026-02-12-desktop-ui-transformation-v2.md

---

## ⚠️ Pre-Flight Gate

**This phase must not begin until all of the following are verified:**

- [ ] `go test ./...` passes with no failures
- [ ] `go build ./cmd/iot-scout` succeeds
- [ ] `src/cmd/iot-scout/frontend/index.html`, `styles.css`, and `app.js` all exist
- [ ] Phase 3 human smoke tests passed — `wails dev` opened a working window, scan produced results, IoT filter toggle worked
- [ ] `wails` CLI is installed: `wails version` returns without error

If any gate is not met, halt and surface to human.

---

## What This Phase Is

Phase 4 is integration and delivery, not implementation. The agent writes two lines to `.gitignore` and runs build commands. Almost everything in this spec is a verification checklist — the final acceptance test for the entire desktop transformation project.

The agent's primary job in this phase is to run `wails build`, confirm it produces a working binary, verify the CLI path is unbroken, and commit the `.gitignore` update. Nothing else.

**If `wails build` fails**, the agent must surface the exact error to the human rather than attempting to fix it autonomously. Build failures at this stage indicate a cross-phase issue — something in a prior phase's files is wrong — and those fixes belong in the appropriate phase's spec context, not silently patched here.

---

# 1. Objective

## 1.1 Business Outcome

Produce a signed-ready single binary (`build/bin/iot-scout`) that supports both `iot-scout scan` (CLI) and `iot-scout gui` (native desktop window), with the `build/` output excluded from version control.

## 1.2 Success Metric & Threshold

| Metric | Target | Verification |
|--------|--------|--------------|
| `wails build` exits 0 | Yes | Task 8 |
| `build/bin/iot-scout gui` opens native window | Yes | Human smoke test |
| `build/bin/iot-scout scan --subnet x.x.x.x/24` produces correct output | Yes | Human smoke test |
| `build/bin/` absent from git tracking | Yes | `git status` after `.gitignore` update |
| `go test ./...` still passes | Yes | Automated |

---

# 2. Agent Persona & Operating Discipline

```
You are a build and delivery agent. Your job in this phase is narrow:
update .gitignore, run wails build, verify outputs, commit.

You do not modify any source files. If wails build fails, you surface
the full error output to the human and stop. You do not attempt autonomous
fixes to prior phases' code.
```

---

# 3. Invariants

- [ ] No source files are modified in this phase — `.gitignore` is the only file changed
- [ ] `go test ./...` passes before and after this phase
- [ ] `build/bin/` is not committed to the repository
- [ ] The production binary supports both `scan` and `gui` subcommands

---

# 4. Explicit Constraints

- [ ] Do not modify any `.go`, `.html`, `.css`, `.js`, `wails.json`, or `go.mod` files
- [ ] Do not attempt to fix compilation or build errors autonomously — surface them to human
- [ ] Do not commit build artifacts — `build/bin/` and `dist/` must be in `.gitignore` before any build output exists

## 4.1 Declared Allowed File Operations

| Operation | Target | Justification |
|-----------|--------|---------------|
| Modify | `.gitignore` (repo root) | Exclude build output directories |

No other files may be created or modified.

---

# 5. Task 7: Development Mode Verification

**Gate to start:** Pre-flight gate passed.  
**Gate to end:** All six `wails dev` checks pass.

This task has no agent implementation work — it is a human verification step. The agent's role is to instruct the human to run `wails dev` and confirm the checklist below.

```bash
cd src/cmd/iot-scout && wails dev
```

| # | Check | Pass Condition |
|---|-------|----------------|
| 1 | Native window opens | Window appears with title "IoT Scout", Wails dock icon |
| 2 | Network auto-detected | Subnet CIDR shown within ~1 second of window opening |
| 3 | Scan triggers OS dialog | Click "Scan My Network" → OS-native password/auth prompt appears (if not root) |
| 4 | Progress updates during scan | Status area shows stage messages and progress percentage |
| 5 | Results display after scan | Summary cards show counts; device cards render in grid |
| 6 | IoT filter works | Non-IoT devices hidden by default; "Show non-IoT devices" checkbox reveals them |

**Also verify the CLI path is unbroken:**

```bash
cd src && go build ./cmd/iot-scout
./iot-scout scan --subnet 192.168.1.0/24
./iot-scout gui   # must open desktop window, not print scan output
./iot-scout --help  # must not show internal-elevated-scan
```

If any of the above fail, this is a Phase 2 or Phase 3 issue. Surface to human with the specific failure before proceeding to Task 8.

---

# 6. Task 8: Production Build

**Gate to start:** Task 7 all checks passed.  
**Gate to end:** `build/bin/iot-scout` exists and passes the binary smoke test below.

## 6.1 Update .gitignore

Add to `.gitignore` at the repository root (not `src/.gitignore`):

```
build/bin/
dist/
```

If `.gitignore` does not exist, create it with these two lines. If it exists, append them. Do not remove or reorder existing entries.

## 6.2 Run Production Build

```bash
cd src/cmd/iot-scout && wails build
```

Expected output path: `src/cmd/iot-scout/build/bin/iot-scout` (or platform equivalent — `iot-scout.exe` on Windows).

**If `wails build` exits non-zero:** Copy the full terminal output and surface to human. Do not proceed. Do not attempt to fix it.

## 6.3 Verify Production Binary

```bash
# Confirm binary exists
ls -lh src/cmd/iot-scout/build/bin/iot-scout

# Confirm CLI subcommand works from production binary
src/cmd/iot-scout/build/bin/iot-scout scan --subnet 192.168.1.0/24

# Confirm GUI subcommand works
src/cmd/iot-scout/build/bin/iot-scout gui

# Confirm help does not expose internal command
src/cmd/iot-scout/build/bin/iot-scout --help | grep -c "internal-elevated-scan"
# Expected output: 0
```

## 6.4 Confirm Build Output Not Tracked

```bash
git status
```

`build/bin/` must not appear as untracked or staged. If it does, the `.gitignore` update did not take effect — verify the path is correct and retry.

## 6.5 Commit

```bash
git add .gitignore
git commit -m "chore: update gitignore for wails build output"
```

---

# 7. Final Project Acceptance Test

This is the complete end-to-end acceptance checklist for the entire desktop transformation. All items must pass before the project is considered complete.

## 7.1 Automated

```bash
cd src

# Full test suite
go test ./... -v

# Cross-platform compilation
GOOS=linux   GOARCH=amd64 go build ./cmd/iot-scout
GOOS=darwin  GOARCH=amd64 go build ./cmd/iot-scout
GOOS=windows GOARCH=amd64 go build ./cmd/iot-scout

# Hidden command not in help
./iot-scout --help | grep -c "internal-elevated-scan"
# Expected: 0

# CLI JSON output has no GUI-only fields
./iot-scout scan --subnet 192.168.1.0/24 --output-json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for device in data.get('devices', []):
    for finding in device.get('findings', []):
        assert 'humanMessage' not in finding or finding['humanMessage'] == '', \
            'humanMessage should be absent from CLI output'
        assert 'riskLevel' not in finding or finding['riskLevel'] == '', \
            'riskLevel should be absent from CLI output'
print('CLI JSON output clean')
"
```

## 7.2 Human: Per-Phase Regression Check

| Phase | What to verify | Pass Condition |
|-------|---------------|----------------|
| Phase 0 | `iot-scout scan` output | Identical to pre-transformation behavior |
| Phase 1 | Elevation path | OS dialog appears for unprivileged scan; already-root skips dialog |
| Phase 2 | `iot-scout gui` routes to window | No scan output on stdout; window opens |
| Phase 3 | UI renders scan results | Device cards, badges, filter toggle all functional |
| Phase 4 | Production binary | `build/bin/iot-scout` passes both CLI and GUI smoke tests |

## 7.3 Human: New File Inventory Check

Verify the complete set of new files exists. No extras, no missing:

| File | Exists? |
|------|:-------:|
| `src/pkg/scanner/pipeline.go` | ☐ |
| `src/pkg/scanner/pipeline_test.go` | ☐ |
| `src/pkg/elevate/elevate.go` | ☐ |
| `src/pkg/elevate/elevate_darwin.go` | ☐ |
| `src/pkg/elevate/elevate_windows.go` | ☐ |
| `src/pkg/elevate/elevate_linux.go` | ☐ |
| `src/pkg/elevate/elevate_test.go` | ☐ |
| `src/cmd/iot-scout/elevated.go` | ☐ |
| `src/cmd/iot-scout/gui.go` | ☐ |
| `src/cmd/iot-scout/app.go` | ☐ |
| `src/cmd/iot-scout/app_test.go` | ☐ |
| `src/cmd/iot-scout/wails.json` | ☐ |
| `src/cmd/iot-scout/frontend/index.html` | ☐ |
| `src/cmd/iot-scout/frontend/styles.css` | ☐ |
| `src/cmd/iot-scout/frontend/app.js` | ☐ |

## 7.4 Human: Modified File Audit

Verify only the expected files were modified, nothing else:

| File | Expected Change | Matches? |
|------|----------------|:--------:|
| `src/cmd/iot-scout/main.go` | Inline scan replaced with `scanner.Run()` | ☐ |
| `src/internal/models/types.go` | `HumanMessage`, `RiskLevel` added to `Finding` with `omitempty` | ☐ |
| `src/go.mod` | Wails v2 and `golang.org/x/sys` added | ☐ |
| `.gitignore` | `build/bin/` and `dist/` added | ☐ |

## 7.5 Open Questions From the Plan

The plan listed two unresolved questions. These are deferred to future work, not resolved in this project:

| Question | Status | Next Step |
|----------|--------|-----------|
| Should the elevated subprocess send progress via pipe/socket instead of temp file? | Deferred | Future release — would enable real-time progress in elevated mode |
| Should `iot-scout` with no subcommand default to `gui` or show help? | Deferred | Decision for next release — currently shows cobra help |

---

# 8. Failure Modes

| Failure | Symptom | Action |
|---------|---------|--------|
| `wails build` fails — missing system deps | Error about WebKit, GTK, or WebView2 | Surface to human; install platform deps per Wails docs |
| `wails build` fails — embed path wrong | `no matching files found` for embed directive | Surface to human; verify `frontend/` path relative to `gui.go` |
| `wails build` fails — Go compilation error | Any Go compiler error | Surface full error to human; do not fix autonomously |
| Build output committed accidentally | `git status` shows `build/bin/` tracked | `git rm -r --cached build/` then verify `.gitignore` path |
| Production binary scan fails | Error running `scan` from `build/bin/` | CLI regression — surface to human |

---

# 9. Expected Commit History

At the end of Phase 4, `git log --oneline` should show roughly this sequence:

```
refactor: extract scan pipeline from main.go into pkg/scanner         ← Phase 0
feat: add OS-native privilege escalation for scanning                 ← Phase 1
feat: add hidden elevated scan subprocess command                     ← Phase 1
feat: add Wails v2 dependency and build config                        ← Phase 2
feat: add gui cobra command and Wails-bound DesktopApp backend        ← Phase 2
feat: add humanized fields to Finding model with omitempty            ← Phase 2
feat: add desktop frontend with IoT device filtering                  ← Phase 3
chore: update gitignore for wails build output                        ← Phase 4
```

If the commit history differs significantly, verify no phases were skipped or collapsed before considering the project complete.

---

# 10. Non-Goals

- Code signing or notarization — deferred per plan
- CI/CD pipeline — not in scope
- Cross-platform binary packaging or distribution — not in scope
- `dist/` directory contents — Wails may create this; it is gitignored but otherwise not managed here

---

# 11. Change Log

| Date | Version | Change | Author |
|------|---------|--------|--------|
| 2026-02-23 | 1.0 | Initial spec — Phase 4 build, final project acceptance matrix, open questions disposition | |

---

*End of Specification — IoT Scout Phase 4*
*End of IoT Scout Desktop Transformation Specification Suite*
