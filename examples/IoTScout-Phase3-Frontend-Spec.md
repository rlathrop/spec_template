# Agentic Development Specification
## IoT Scout — Phase 3: Frontend

> Version: 1.0  
> Created: 2026-02-23  
> Project: iot-scout  
> Phase: 3 of 4  
> Depends on: Phase 0, 1, and 2 complete and merged  
> Source Plan: 2026-02-12-desktop-ui-transformation-v2.md

---

## ⚠️ Pre-Flight Gate

**This phase must not begin until all of the following are verified:**

- [ ] `src/cmd/iot-scout/frontend/` directory exists (created in Phase 2, Task 3)
- [ ] `src/cmd/iot-scout/gui.go` exists with `//go:embed all:frontend`
- [ ] `src/cmd/iot-scout/app.go` exists with `DetectNetwork()` and `StartScan()` bound methods
- [ ] `go build ./cmd/iot-scout` succeeds
- [ ] Phase 2 human smoke tests passed — `wails dev` opens a window and `DetectNetwork()` returns a subnet
- [ ] The agent has read `src/pkg/report/` to determine the exact JSON shape of `ScanReport` (see Section 6)

If any gate is not met, halt and surface to human.

---

## Why Phase 3 Has a Different Verification Model

Every prior phase had a compiler and `go test` as the primary correctness signal. Phase 3 has neither. The frontend files are HTML, CSS, and vanilla JavaScript — no build step, no type system, no test runner. The only runtime that can execute these files correctly is Wails itself, because `window.go` and `window.runtime` are injected by the Wails webview, not available in a browser.

This changes the spec's job in three ways:

**1. The JSON contract must be made explicit.** `app.js` accesses fields like `report.summary.total_devices`, `device.classification.IsIoT`, and `device.classification.DeviceType` by name. These names come from the Go structs in `pkg/report` and `internal/models`, serialized as JSON. If the Go struct uses `TotalDevices int \`json:"totalDevices"\`` but the JS accesses `total_devices`, it silently produces `undefined`. The agent must read the actual Go types before writing any JS field access.

**2. Wails runtime timing is platform-dependent.** The plan calls `window.runtime.EventsOn(...)` inside `DOMContentLoaded`. On some platforms (particularly Windows WebView2), `window.runtime` is not available at `DOMContentLoaded` — it's injected asynchronously by Wails after the DOM is ready. The correct pattern is to use the `wails:ready` event or `window.addEventListener('load', ...)` for runtime-dependent initialization. This spec resolves the correct approach before implementation.

**3. Verification is human-only.** There is no automated test for this phase. The human smoke test matrix in Section 10 is the only gate before Phase 4.

---

# 1. Objective

## 1.1 Business Outcome

Implement the desktop UI that displays scan results to end users. The UI must be usable by a non-technical homeowner: one-click scanning, plain-language security summaries, IoT devices shown by default with non-IoT devices hidden behind a toggle.

## 1.2 Success Metric & Threshold

| Metric | Baseline | Target | Verification |
|--------|----------|--------|--------------|
| Scan initiates on button click | No UI | Click triggers `StartScan()`, OS dialog appears if needed | Human smoke test |
| Network auto-detected on open | No UI | CIDR shown on launch, no user input required | Human smoke test |
| Device cards render after scan | No UI | One card per device in scan report | Human smoke test |
| IoT devices shown by default | No UI | Non-IoT cards hidden, IoT cards visible | Human smoke test |
| Non-IoT toggle works | No UI | Checkbox reveals/hides non-IoT cards | Human smoke test |
| No device data appears unescaped in DOM | No UI | All network-sourced strings pass through `escapeHtml` before `innerHTML` | Code review invariant |
| Elevated scan shows indeterminate state | No UI | Progress shows "Scanning (elevated)..." without advancing | Human smoke test |

## 1.3 Current State Baseline

- `frontend/` contains only `.gitkeep`
- `wails dev` opens a blank white window (Wails shell works, no content)
- All Go backend methods are implemented and working

---

# 2. Agent Persona & Operating Discipline

```
You are a frontend implementation agent working inside a Wails v2 desktop application.
Your operating discipline: read the Go types first, implement the JSON contract
before writing any field access, escape all network-sourced data before DOM insertion.

You understand that `window.go` and `window.runtime` are injected by the Wails
webview — they do not exist in a browser. You do not test by opening index.html
in a browser. You implement the timing fix for window.runtime (see Section 5.3)
and do not revert to the plan's DOMContentLoaded pattern.

You treat all device data (hostname, vendor, IP, MAC, finding messages) as
untrusted network input that must be escaped before any innerHTML assignment.
Badge color values are exempt because they come only from hardcoded lookup
tables, not from device data.
```

---

# 3. Invariants (Must Always Be True)

- [ ] All three files (`index.html`, `styles.css`, `app.js`) live at `src/cmd/iot-scout/frontend/` — no other location
- [ ] The `.gitkeep` file is deleted once real frontend files exist (or kept — either is fine, it does not affect embedding)
- [ ] Every string derived from device data (hostname, vendor, IP, MAC, finding message, device type) is passed through `escapeHtml()` before being assigned to `innerHTML`
- [ ] `window.runtime.EventsOn` is not called at `DOMContentLoaded` — see Section 5.3 for the correct timing pattern
- [ ] `window.go.main.DesktopApp.DetectNetwork()` is called once on load — not on every scan
- [ ] `scanBtn` is disabled for the full duration of a scan and re-enabled on both success and error paths
- [ ] `devicesSection.innerHTML = ''` clears previous results before rendering new ones — no duplicate cards
- [ ] The `escapeHtml` function uses `document.createElement('div')` + `textContent` pattern — not a regex or manual replacement
- [ ] JSON field names in `app.js` match exactly what `report.ScanReport` and `models.Device` serialize to — verified in Section 6

---

# 4. Explicit Constraints (What Is Forbidden)

- [ ] Do not modify any Go files — this phase is frontend only
- [ ] Do not add any JavaScript frameworks, libraries, or npm packages
- [ ] Do not add `<script src="...">` tags referencing external CDNs — offline-capable only
- [ ] Do not use `eval()`, `Function()`, or any dynamic code execution
- [ ] Do not access `window.runtime` before it is injected — see Section 5.3
- [ ] Do not interpolate `risk.color` values from device data — `getRiskBadge` must only return from its hardcoded set: `muted`, `danger`, `warning`, `success`
- [ ] Do not remove the `escapeHtml` wrapper from any device-data field assignment
- [ ] Do not add a Node.js build step, webpack config, or any build tooling
- [ ] Do not use CSS frameworks — the plan's CSS is self-contained and complete

## 4.1 Declared Allowed File Operations

| Operation | Target | Justification |
|-----------|--------|---------------|
| Create | `src/cmd/iot-scout/frontend/index.html` | UI structure |
| Create | `src/cmd/iot-scout/frontend/styles.css` | Styling |
| Create | `src/cmd/iot-scout/frontend/app.js` | UI logic and Wails calls |
| Delete | `src/cmd/iot-scout/frontend/.gitkeep` | No longer needed once real files exist |

No other files may be created or modified.

---

# 5. Implementation

## 5.1 Prerequisite: Read the JSON Contract

Before writing `app.js`, the agent must read the following Go source files and record the exact JSON field names they serialize to:

- `src/pkg/report/` — find `ScanReport`, `DeviceSummary` (or equivalent summary struct), and how `Devices` is serialized
- `src/internal/models/types.go` — find `Device`, `Classification`, `Finding`, `Service`

The agent must produce a written mapping before writing any JS field access:

```
report.summary → JSON key: ?
report.summary.total_devices → JSON key: ?
report.summary.iot_devices → JSON key: ?
report.devices → JSON key: ?
device.ip → JSON key: ?
device.mac → JSON key: ?
device.hostname → JSON key: ?
device.vendor → JSON key: ?
device.classification → JSON key: ?
device.classification.IsIoT → JSON key: ?
device.classification.DeviceType → JSON key: ?
device.findings → JSON key: ?
finding.severity → JSON key: ?
finding.message → JSON key: ?
finding.humanMessage → JSON key: ?
```

If any Go field uses a JSON tag that differs from the plan's JS field access (e.g., `json:"totalDevices"` vs. `total_devices`), the agent must use the correct JSON key in `app.js`, not the plan's version. The plan's JS was written assuming these field names — if the Go types disagree, the Go types win.

**This is the single most common silent failure mode in this phase.** An incorrect field name produces `undefined` in JavaScript with no error.

## 5.2 index.html

Implement exactly as specified in the plan. No deviations. Key structural elements the agent must verify are present:

| Element ID | Purpose | Required |
|------------|---------|:--------:|
| `scan-btn` | Triggers scan | Yes |
| `scan-status` | Progress and status messages | Yes |
| `network-info` | Detected subnet display | Yes |
| `summary` | Summary card section (hidden initially) | Yes |
| `device-count` | Total device count stat | Yes |
| `iot-count` | IoT device count stat | Yes |
| `issue-count` | Security issue count stat | Yes |
| `filter-bar` | Filter toggle container (hidden initially) | Yes |
| `show-all-toggle` | Checkbox for non-IoT visibility | Yes |
| `devices` | Device card container | Yes |

The `<script src="/app.js"></script>` tag must be at the end of `<body>`, not in `<head>`. The Wails runtime is injected before scripts at the end of body, making `window.go` and `window.runtime` available when `app.js` executes.

No `<meta http-equiv="Content-Security-Policy">` is needed — Wails manages its own CSP for the webview.

## 5.3 Wails Runtime Timing Fix

**The plan's code has a timing issue that this spec corrects.**

The plan calls `window.runtime.EventsOn(...)` inside `DOMContentLoaded`:

```javascript
// Plan's approach — may fail on Windows WebView2
document.addEventListener('DOMContentLoaded', function() {
    // ...
    window.runtime.EventsOn('scan:progress', function(data) { ... });
});
```

On some Wails platforms (particularly Windows WebView2), `window.runtime` is not yet defined when `DOMContentLoaded` fires. The correct approach is to use the `window.addEventListener('load', ...)` event for any code that depends on `window.runtime`, or to check for runtime availability with a small polling loop.

**The agent must implement the following pattern instead:**

```javascript
// Phase 3 implementation — split DOM init from runtime init
document.addEventListener('DOMContentLoaded', function() {
    // Safe: only DOM element references and non-Wails event listeners
    scanBtn = document.getElementById('scan-btn');
    scanStatus = document.getElementById('scan-status');
    summarySection = document.getElementById('summary');
    devicesSection = document.getElementById('devices');
    networkInfoEl = document.getElementById('network-info');
    filterBar = document.getElementById('filter-bar');
    showAllToggle = document.getElementById('show-all-toggle');

    scanBtn.addEventListener('click', startScan);
    showAllToggle.addEventListener('change', toggleNonIoT);
});

window.addEventListener('load', function() {
    // Safe: Wails runtime is guaranteed available after window load
    window.runtime.EventsOn('scan:progress', function(data) {
        var pct = Math.round(data.progress * 100);
        scanStatus.className = 'scan-status';
        scanStatus.innerHTML =
            escapeHtml(data.message) + ' (' + pct + '%)' +
            '<div class="progress-bar"><div class="progress-fill" style="width:' + pct + '%"></div></div>';
    });

    detectNetwork();
});
```

`detectNetwork()` is also moved to `load` because it calls `window.go.main.DesktopApp.DetectNetwork()`, which also depends on the Wails runtime being available.

**Note:** The `scanStatus.className = 'scan-status'` line in the progress handler ensures the element is visible when a progress event fires, even if it was in `status-hidden` state. Add this to the progress handler.

## 5.4 styles.css

Implement exactly as specified in the plan. The CSS is complete and correct — no additions or modifications needed. The agent must implement it verbatim.

Key CSS behaviors the human smoke tests will verify:

| Selector | Behavior |
|----------|----------|
| `.device-card.non-iot` | `display: none` — hides non-IoT cards by default |
| `.device-card.non-iot.show` | `display: block` — reveals them when toggle is checked |
| `.device-card.has-issues` | `border-left-color: var(--danger)` — red left border |
| `.btn-primary:disabled` | `opacity: 0.5; cursor: not-allowed` — visual disabled state |
| `.progress-fill` | `transition: width 0.3s` — animated progress bar |

## 5.5 app.js — Security Requirements

The `escapeHtml` function and its correct application are the most important security property of this file. IoT Scout scans the local network and renders device-supplied data (hostnames, vendor strings, finding messages) directly in the UI. A malicious device could serve an mDNS hostname containing HTML or script content.

**Escaping rules:**

| Data source | Escaping required | Method |
|-------------|:-----------------:|--------|
| `device.hostname` | Yes | `escapeHtml()` |
| `device.vendor` | Yes | `escapeHtml()` |
| `device.ip` | Yes | `escapeHtml()` (technically safe as IPv4, but must be consistent) |
| `device.mac` | Yes | `escapeHtml()` |
| `device.classification.DeviceType` | Yes | `escapeHtml()` |
| `finding.humanMessage` / `finding.message` | Yes | `escapeHtml()` |
| `data.message` (progress event) | Yes | `escapeHtml()` |
| `risk.color` (badge class suffix) | **No** | Comes only from `getRiskBadge` hardcoded return values |
| `risk.text` (badge label) | **No** | Comes only from `getRiskBadge` hardcoded return values |
| `getIcon(deviceType)` return | **No** | Returns only emoji literals from a hardcoded map |

**The `escapeHtml` implementation:**

```javascript
function escapeHtml(str) {
    var div = document.createElement('div');
    div.textContent = str || '';
    return div.innerHTML;
}
```

This is the correct implementation. The agent must not substitute a regex-based alternative.

## 5.6 app.js — Wails Binding Call Pattern

All calls to Go methods follow this pattern:

```javascript
window.go.main.DesktopApp.MethodName(arg1, arg2)
    .then(function(result) { /* success */ })
    .catch(function(err) { /* failure */ });
```

The `window.go.main.DesktopApp` prefix is fixed — `main` is the Go package name, `DesktopApp` is the struct name. Any deviation from this path means the binding won't resolve.

`StartScan` returns a `ScanResult` object (never rejects), so errors appear in `result.error`, not in `.catch()`. The `.catch()` handler covers Wails binding failures (e.g., Go panicked), not application-level errors. Both paths must re-enable `scanBtn`.

## 5.7 app.js — Elevated Scan Progress Behavior

When the scan is elevated (`NeedsElevation()` was true on the Go side), the frontend receives exactly two `scan:progress` events:
1. `{ progress: 0.0, message: "Requesting permissions..." }`
2. `{ progress: 0.05, message: "Scanning (elevated)..." }`

After event 2, no further progress events arrive until `StartScan()` resolves. The progress bar will sit at 5% for the duration of the scan. This is correct and expected — the agent must not attempt to animate or fake progress beyond what the events provide.

## 5.8 app.js — Complete Function Reference

The agent must implement all of the following functions. None may be omitted or renamed.

| Function | Called from | Purpose |
|----------|------------|---------|
| `detectNetwork()` | `load` event | Calls `DetectNetwork()`, populates `detectedSubnet` and `networkInfoEl` |
| `startScan()` | `scan-btn` click | Guards against missing subnet, disables button, calls `StartScan()` |
| `displayResults(report)` | `startScan` success | Populates summary cards and device card grid |
| `createDeviceCard(device)` | `displayResults` | Creates and returns a single device card DOM element |
| `toggleNonIoT()` | `show-all-toggle` change | Adds/removes `.show` class on `.non-iot` cards |
| `getRiskBadge(device, isIoT)` | `createDeviceCard` | Returns `{ text, color }` based on findings severity |
| `renderFindings(findings)` | `createDeviceCard` | Returns HTML string for findings section, empty string if no findings |
| `getIcon(deviceType)` | `createDeviceCard` | Returns emoji for device type from hardcoded map |
| `escapeHtml(str)` | Multiple | Escapes HTML special characters in network-sourced strings |

---

# 6. JSON Contract (Must Be Verified Before Implementation)

The following table is a template. The agent must fill in the actual JSON keys by reading the Go source, then implement `app.js` using those keys.

| Go Path | Go JSON Tag (to verify) | JS Access in app.js |
|---------|------------------------|---------------------|
| `ScanReport.Summary` | `json:"summary"` (verify) | `report.summary` |
| `Summary.TotalDevices` | `json:"total_devices"` (verify) | `report.summary.total_devices` |
| `Summary.IoTDevices` | `json:"iot_devices"` (verify) | `report.summary.iot_devices` |
| `ScanReport.Devices` | `json:"devices"` (verify) | `report.devices` |
| `Device.IP` | `json:"ip"` (verify) | `device.ip` |
| `Device.MAC` | `json:"mac"` (verify) | `device.mac` |
| `Device.Hostname` | `json:"hostname"` (verify) | `device.hostname` |
| `Device.Vendor` | `json:"vendor"` (verify) | `device.vendor` |
| `Device.Classification` | `json:"classification"` (verify) | `device.classification` |
| `Classification.IsIoT` | `json:"IsIoT"` or `json:"isIoT"` (verify) | `device.classification.IsIoT` |
| `Classification.DeviceType` | `json:"DeviceType"` or `json:"deviceType"` (verify) | `device.classification.DeviceType` |
| `Device.Findings` | `json:"findings"` (verify) | `device.findings` |
| `Finding.Severity` | `json:"severity"` (verify) | `finding.severity` |
| `Finding.Message` | `json:"message"` (verify) | `finding.message` |
| `Finding.HumanMessage` | `json:"humanMessage,omitempty"` (verify) | `finding.humanMessage` |

> **If any actual JSON tag differs from the plan's JS access, the agent must use the actual tag.** Surface any mismatch in the planning output before writing files.

---

# 7. UI State Machine

The UI moves through these states. The agent must ensure all transitions are handled:

```
INITIAL
  │ load event fires
  ▼
DETECTING_NETWORK
  │ DetectNetwork() resolves
  ├─ success → READY (subnet shown, button enabled)
  └─ failure → READY_NO_NETWORK (error shown, button enabled but will guard in startScan)

READY
  │ user clicks "Scan My Network"
  ▼
SCANNING
  │ scan:progress events update status area
  │ scanBtn disabled
  │ StartScan() resolves
  ├─ result.error present → ERROR (status shows error, button re-enabled)
  └─ result.report present → RESULTS (summary + cards shown, button re-enabled)

ERROR → READY (button re-enabled, user can retry)
RESULTS → SCANNING (user can scan again, previous results cleared on new scan)
```

The critical state invariants:
- `scanBtn` is ONLY disabled during SCANNING state
- `devicesSection` is cleared at the start of `displayResults`, not at the start of `startScan`
- `summarySection` and `filterBar` remain visible between scans — they show the previous result until a new one arrives

---

# 8. Outputs

Phase 3 produces three files:

| File | Approx. lines | Content |
|------|:-------------:|---------|
| `frontend/index.html` | ~45 | Static HTML shell, 10 element IDs, script tag at end of body |
| `frontend/styles.css` | ~130 | CSS custom properties, component styles, `.non-iot` visibility logic |
| `frontend/app.js` | ~150 | Wails binding calls, DOM manipulation, escaping, event handlers |

No binary output. No build artifacts. `wails dev` and `wails build` embed these files via the `//go:embed` directive in `gui.go`.

---

# 9. Failure Modes

| Failure | Symptom | Root Cause | Fix |
|---------|---------|-----------|-----|
| `window.runtime` undefined | `TypeError: Cannot read properties of undefined` in console at load | `EventsOn` called at `DOMContentLoaded` before Wails injects runtime | Use `load` event per Section 5.3 |
| `window.go` undefined | `StartScan` silently does nothing | `window.go` called before Wails runtime available | Same fix — move to `load` event |
| Wrong JSON field name | Summary shows 0, cards don't render | Mismatch between Go JSON tag and JS field access | Read Go types first per Section 5.1 and 6 |
| `device.classification.IsIoT` undefined | All devices show as non-IoT | JSON tag casing mismatch (`IsIoT` vs `isIoT`) | Verify actual tag before writing |
| XSS via hostname | `<script>` tag in device hostname executes | Missing `escapeHtml` on a field | Apply escaping per Section 5.5 table |
| `scanBtn` stays disabled after error | User cannot retry after a failed scan | `scanBtn.disabled = false` missing from `.catch()` path | Both `.then()` and `.catch()` must re-enable button |
| Double cards on second scan | Previous scan's cards remain visible | `devicesSection.innerHTML = ''` missing | Clear in `displayResults` before appending |
| Progress bar never appears | Status area remains hidden during scan | `scanStatus.className` not updated to `'scan-status'` from `'status-hidden'` | Set class in `startScan` and progress handler |

---

# 10. Validation: Human Smoke Test Matrix

There is no automated testing for this phase. All validation is human-executed via `wails dev`.

## 10.1 Setup

```bash
cd src/cmd/iot-scout && wails dev
```

The Wails dev server opens a native window with hot-reload. The browser console is accessible via right-click → Inspect (on most platforms).

**First check:** Open the browser console (DevTools) before testing anything. Any errors visible at startup are blocking — fix before proceeding.

## 10.2 Smoke Tests

| ID | Test | Steps | Pass Condition | Fail Indicators |
|----|------|-------|----------------|-----------------|
| ST-001 | Window opens | `wails dev` | Native window appears, title "IoT Scout", white background | Blank window, error dialog, crash |
| ST-002 | Network detected | Open window | Subnet displayed (e.g. `192.168.1.0/24 (en0)`) within 1 second | "Could not detect network" or no text |
| ST-003 | Scan initiates | Click "Scan My Network" | Button disables, status area appears with progress text | Button stays enabled, nothing happens |
| ST-004 | OS dialog (unprivileged) | Click scan as non-root | Native OS password/auth dialog appears | No dialog, immediate error |
| ST-005 | Elevated progress | Approve OS dialog | Status shows "Scanning (elevated)..." at ~5%, stays there during scan | Progress advancing past 5%, or missing |
| ST-006 | Privileged progress | Run `sudo wails dev`, click scan | Progress bar advances through stages with percentages | No progress, or stuck at 0% |
| ST-007 | Results render | Scan completes | Summary cards show counts, device cards appear | Empty UI, "0" counts when devices present |
| ST-008 | IoT default filter | Results visible | Only IoT devices visible, non-IoT hidden | Non-IoT devices visible without toggle |
| ST-009 | Toggle reveals non-IoT | Check "Show non-IoT devices" | Non-IoT cards appear | No change |
| ST-010 | Toggle hides non-IoT | Uncheck toggle | Non-IoT cards hide again | Cards remain visible |
| ST-011 | Risk badges correct | Results with known devices | IoT devices with CRITICAL/HIGH findings show "Fix Now" badge | Wrong badge, missing badge |
| ST-012 | No XSS via hostname | See Section 10.3 | Browser renders escaped text | Alert dialog, script executes |
| ST-013 | Button re-enables on error | Cancel OS dialog | Button re-enables, error message shown | Button stays disabled |
| ST-014 | Second scan clears first | Scan twice | Second results replace first, no duplicate cards | Cards accumulate |
| ST-015 | CLI still works | In separate terminal: `./iot-scout scan --subnet x.x.x.x/24` | CLI output identical to pre-Phase-3 | Missing output, error, or extra fields |

## 10.3 XSS Verification (ST-012)

Because this is a security tool, XSS resistance must be verified explicitly. Use one of:

**Option A (if you control a device on the network):** Configure a device hostname via mDNS or router DHCP to contain `<img src=x onerror=alert(1)>`. Run scan. Verify the card displays the literal string, no alert fires.

**Option B (manual DOM injection test):** In the browser DevTools console, after a scan has completed:

```javascript
// Simulate a device with malicious hostname
createDeviceCard({
    ip: '192.168.1.99',
    mac: '',
    hostname: '<img src=x onerror=alert(1)>',
    vendor: '',
    classification: { IsIoT: true, DeviceType: 'Unknown' },
    findings: []
});
```

Append the result to `devicesSection` and verify the card shows the literal text `<img src=x onerror=alert(1)>` without triggering an alert.

---

# 11. Validation Scenarios (Visible)

These can be verified by code inspection before `wails dev` is run.

| ID | Name | What to verify in code |
|----|------|------------------------|
| VS-001 | `escapeHtml` used on all device fields | `device.hostname`, `device.ip`, `device.mac`, `device.vendor`, `device.classification.DeviceType`, `data.message`, `finding.humanMessage \|\| finding.message` all wrapped in `escapeHtml()` |
| VS-002 | `window.runtime.EventsOn` in `load` not `DOMContentLoaded` | Search `app.js` for `EventsOn` — it must appear inside a `load` event listener |
| VS-003 | `detectNetwork` in `load` not `DOMContentLoaded` | `detectNetwork()` call appears inside `load` event listener |
| VS-004 | `scanBtn` re-enabled in both paths | `scanBtn.disabled = false` appears in both `.then()` and `.catch()` of `StartScan` call |
| VS-005 | `devicesSection.innerHTML = ''` clears before render | Appears at start of `displayResults` before the `.forEach` loop |
| VS-006 | `getRiskBadge` only returns hardcoded colors | Function only returns from `{ text, color }` literals — no device data interpolated into `color` |
| VS-007 | All 9 required functions present | `detectNetwork`, `startScan`, `displayResults`, `createDeviceCard`, `toggleNonIoT`, `getRiskBadge`, `renderFindings`, `getIcon`, `escapeHtml` all defined |

---

# 12. Holdout Scenarios (Human-Controlled)

| ID | Name | What Human Verifies |
|----|------|---------------------|
| HS-001 | Second scan on different subnet | Change network (e.g. VPN), rescan — new subnet auto-detected on window open, not cached from first detection |
| HS-002 | Zero IoT devices | Scan a network with only non-IoT devices (or no devices) — UI shows 0 IoT, all devices hidden unless toggled |
| HS-003 | Device with no findings | Renders card with no findings section, "Looks Fine" badge |
| HS-004 | Device with CRITICAL finding | Renders card with red left border, "Fix Now" badge, finding message shown |
| HS-005 | Issue count accuracy | Manual count of non-INFO findings across all devices matches "Security Issues" summary card |

---

# 13. Non-Goals

- Sorting or searching devices in the UI
- Exporting scan results from the GUI (report generation is Go-side only)
- Dark mode
- Displaying open ports or services per device
- Subnet input field — subnet is auto-detected, not user-editable in Phase 3
- `allowDefaultCreds` toggle in UI — hardcoded to `false` in `StartScan` call
- Accessibility compliance (ARIA labels, keyboard nav) — future work

---

# 14. Agent Planning Instructions

Before writing any file, the agent must produce a written plan covering:

1. **Read** `src/pkg/report/` and list the exact Go type and JSON tag for each field accessed in `app.js` — fill in the JSON contract table from Section 6
2. **Read** `src/internal/models/types.go` and confirm the `Classification` struct field names and their JSON tags — particularly `IsIoT` and `DeviceType` casing
3. **State any JSON tag mismatches** between the plan's JS field access and the actual Go tags — these must be corrected in `app.js`
4. **Confirm the timing fix** — state that `EventsOn` and `detectNetwork` will be in `load`, not `DOMContentLoaded`
5. **Confirm the escaping table** — list every field that will receive `escapeHtml` treatment
6. **State implementation order**: `index.html` → `styles.css` → `app.js`
7. **Only then** write files

> If any Go type is not JSON-serializable (unexported fields, interface types, etc.) in a way that would prevent `ScanReport` from round-tripping through Wails to the frontend, halt and surface to human.

---

# 15. Change Log

| Date | Version | Change | Author |
|------|---------|--------|--------|
| 2026-02-23 | 1.0 | Initial spec — Phase 3 frontend with runtime timing fix, JSON contract verification requirement, XSS invariants, complete smoke test matrix | |

---

*End of Specification — IoT Scout Phase 3*
