# Agentic Development Specification
## IoT Scout — UI Enhancement: Dark Technical Theme

> Version: 1.0  
> Created: 2026-02-23  
> Project: iot-scout  
> Phase: Post-Phase-3 enhancement  
> Depends on: Phase 3 complete — `index.html`, `styles.css`, `app.js` all exist and working  
> Source: Design evaluation "Dark Technical" direction

---

## ⚠️ Pre-Flight Gate

- [ ] `wails dev` opens a working window with scan results
- [ ] Phase 3 smoke tests passed (all 15 items in Phase 3 spec Section 10.2)
- [ ] `go test ./...` passes — no regressions from prior phases

---

## What This Spec Covers

Seven visual enhancements across three files. No Go changes. No new packages. No build system changes. The `app.js` logic — Wails bindings, scan state machine, `escapeHtml`, event handling — is preserved exactly. Only rendering and styling are modified.

**Implementation surface by file:**

| File | Nature of change |
|------|-----------------|
| `styles.css` | Full replacement — new dark palette, typography, component styles |
| `index.html` | Surgical addition — hero section with pulse ring, font preconnect links |
| `app.js` | Surgical edits — 5 targeted changes to rendering functions only |

**The Phase 3 security invariants are non-negotiable and survive this enhancement unchanged.** Every `escapeHtml()` call that existed in Phase 3 must remain. The only change to `renderFindings` is replacing the wrapping div with a severity chip — the content is still escaped.

---

# 1. Objective

Transform the visual identity from generic SaaS blue-on-white to a dark technical aesthetic appropriate for a network security tool. Credibility, information hierarchy, and risk visibility are the primary design values.

## 1.1 Design Direction

**Reference aesthetic:** Tailscale admin UI meets terminal. Dark slate backgrounds, sharp Inter typography, JetBrains Mono for technical data fields, severity colors that read clearly against dark.

**What changes:** Visual presentation only — color, typography, layout of pre-existing elements, and the information hierarchy of security findings.

**What does not change:** Scan logic, Go bindings, state machine, escaping, event handling, element IDs, CSS class names used by `app.js` logic (`.non-iot`, `.has-issues`, `.show`, `.device-card`, `.scan-status`, `.status-hidden`).

## 1.2 Success Metrics

| Metric | Before | After | Verification |
|--------|--------|-------|--------------|
| Pre-scan state communicates identity | Floating button, no context | Full hero with app name, subtitle, network pill, pulse ring | Human visual check |
| Security Issues card prominence | Same weight as other two cards | Turns red background when issues > 0 | Human smoke test |
| Finding readability | Plain text in faint red box, no severity label | Severity chip (CRITICAL / HIGH / MEDIUM / LOW) + message | Human visual check |
| Technical field legibility | System font, same as prose | Monospace (JetBrains Mono) for IP, MAC, CIDR | Human visual check |
| Overall aesthetic | Generic light SaaS | Dark slate, matches security tooling conventions | Human visual check |
| Functional regression | All Phase 3 smoke tests pass | All Phase 3 smoke tests still pass | Human smoke test re-run |

---

# 2. Agent Persona & Operating Discipline

```
You are a UI enhancement agent making surgical changes to a working frontend.
Your operating discipline: preserve all logic, change only presentation.

Before editing app.js, you identify every existing escapeHtml() call and
confirm it survives in the modified version. You do not rewrite functions
that are not in scope — you edit only the lines that need to change.

You do not introduce new JavaScript dependencies, CDN links, or build steps.
JetBrains Mono and Inter are loaded via Google Fonts (one preconnect + one
stylesheet link in index.html head). This is the only external resource added.
```

---

# 3. Invariants — Must Survive This Enhancement

These existed in Phase 3 and must be true after this enhancement:

- [ ] `escapeHtml()` is called on every device-data field before `innerHTML` assignment — specifically: `name` (hostname/vendor/ip), `device.ip`, `device.mac`, `device.classification.DeviceType`, `data.message` (progress), and the finding message in `renderFindings`
- [ ] `.non-iot` CSS class hides non-IoT cards by default
- [ ] `.non-iot.show` CSS class reveals them when toggle is checked  
- [ ] `.has-issues` CSS class applies to device cards with CRITICAL or HIGH findings
- [ ] `.scan-status` makes the status area visible; `.status-hidden` hides it
- [ ] `scanBtn.disabled` is set true at scan start and false on both success and error paths
- [ ] All 10 element IDs referenced by `app.js` exist in `index.html`: `scan-btn`, `scan-status`, `network-info`, `summary`, `device-count`, `iot-count`, `issue-count`, `filter-bar`, `show-all-toggle`, `devices`
- [ ] `getRiskBadge` returns only from its hardcoded set — no device data interpolated into class names

---

# 4. Explicit Constraints

- [ ] Do not modify any Go files
- [ ] Do not remove or rename any element ID that `app.js` references
- [ ] Do not remove or rename any CSS class that `app.js` adds/removes/queries (`.non-iot`, `.has-issues`, `.show`, `.scan-status`, `.status-hidden`, `.device-card`)
- [ ] Do not introduce npm, webpack, or any build tooling
- [ ] Do not add any JavaScript beyond what is specified in the `app.js` edits below
- [ ] Do not use `eval()` or dynamic code execution
- [ ] Do not add CDN links for JavaScript libraries — Google Fonts for Inter and JetBrains Mono is the only external resource permitted

## 4.1 Declared Allowed File Operations

| Operation | Target | Scope |
|-----------|--------|-------|
| Replace | `src/cmd/iot-scout/frontend/styles.css` | Full file replacement |
| Edit | `src/cmd/iot-scout/frontend/index.html` | Add `<link>` tags in `<head>`, add `#hero` section in `<main>`, update `#network-info` element |
| Edit | `src/cmd/iot-scout/frontend/app.js` | 5 targeted function edits — specified line by line in Section 7 |

---

# 5. Change 1: Dark Theme Palette (styles.css — full replacement)

Replace the entire `styles.css` with the following. The new file is a complete replacement — do not attempt to merge or patch.

```css
/* ── Google Fonts are loaded via <link> in index.html ── */

:root {
    /* Palette */
    --bg:           #0d1117;
    --surface:      #161b22;
    --surface-2:    #21262d;
    --border:       #30363d;
    --primary:      #58a6ff;
    --success:      #3fb950;
    --warning:      #d29922;
    --danger:       #f85149;
    --danger-bg:    rgba(248, 81, 73, 0.12);
    --text:         #e6edf3;
    --text-muted:   #8b949e;
    --text-subtle:  #484f58;

    /* Typography */
    --font-sans:  'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
    --font-mono:  'JetBrains Mono', 'Fira Code', 'Cascadia Code', monospace;
}

* { margin: 0; padding: 0; box-sizing: border-box; }

body {
    font-family: var(--font-sans);
    background: var(--bg);
    color: var(--text);
    line-height: 1.6;
    -webkit-font-smoothing: antialiased;
}

/* ── Layout ── */
.container { max-width: 1200px; margin: 0 auto; padding: 2rem; }

/* ── Header ── */
header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding-bottom: 1.5rem;
    border-bottom: 1px solid var(--border);
    margin-bottom: 2rem;
}
header h1 {
    font-size: 1.25rem;
    font-weight: 600;
    letter-spacing: -0.01em;
    color: var(--text);
}
header h1 span.accent { color: var(--primary); }
.header-right { display: flex; align-items: center; gap: 0.75rem; }

/* Network pill — shown in header after detection */
.network-pill {
    display: inline-flex;
    align-items: center;
    gap: 0.4rem;
    padding: 0.3rem 0.75rem;
    background: var(--surface-2);
    border: 1px solid var(--border);
    border-radius: 9999px;
    font-family: var(--font-mono);
    font-size: 0.75rem;
    color: var(--text-muted);
}
.network-pill .dot {
    width: 6px;
    height: 6px;
    border-radius: 50%;
    background: var(--success);
}
.network-pill.error .dot { background: var(--danger); }

/* ── Hero (pre-scan state) ── */
#hero {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 5rem 2rem;
    text-align: center;
}
.hero-ring-wrap {
    position: relative;
    width: 120px;
    height: 120px;
    margin-bottom: 2rem;
}
.hero-ring {
    width: 120px;
    height: 120px;
    border-radius: 50%;
    border: 2px solid var(--primary);
    position: absolute;
    opacity: 0.3;
    animation: pulse-ring 2s ease-out infinite;
}
.hero-ring:nth-child(2) { animation-delay: 0.6s; }
.hero-ring:nth-child(3) { animation-delay: 1.2s; }
@keyframes pulse-ring {
    0%   { transform: scale(0.5); opacity: 0.4; }
    100% { transform: scale(1.4); opacity: 0; }
}
.hero-icon {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
}
/* SVG radar icon — inline in HTML (see Section 6) */
.hero-icon svg {
    width: 40px;
    height: 40px;
    stroke: var(--primary);
    fill: none;
    stroke-width: 1.5;
}
.hero-title {
    font-size: 2rem;
    font-weight: 700;
    letter-spacing: -0.03em;
    margin-bottom: 0.5rem;
}
.hero-title span { color: var(--primary); }
.hero-subtitle {
    color: var(--text-muted);
    font-size: 1rem;
    margin-bottom: 2.5rem;
    max-width: 360px;
}

/* ── Scan section ── */
.scan-section { text-align: center; margin-bottom: 2rem; }

/* #network-info used only as fallback when header pill is unavailable */
#network-info {
    font-size: 0.85rem;
    color: var(--text-muted);
    margin-bottom: 1rem;
    font-family: var(--font-mono);
}

.btn-primary {
    background: var(--primary);
    color: #0d1117;
    border: none;
    padding: 0.875rem 2rem;
    font-size: 0.9375rem;
    font-weight: 600;
    border-radius: 0.5rem;
    cursor: pointer;
    transition: background 0.15s, transform 0.15s, box-shadow 0.15s;
    font-family: var(--font-sans);
    letter-spacing: 0.01em;
}
.btn-primary:hover {
    background: #79b8ff;
    transform: translateY(-1px);
    box-shadow: 0 4px 16px rgba(88, 166, 255, 0.25);
}
.btn-primary:disabled {
    opacity: 0.4;
    cursor: not-allowed;
    transform: none;
    box-shadow: none;
}

/* ── Scan status ── */
.status-hidden { display: none; }
.scan-status {
    margin-top: 1rem;
    padding: 0.875rem 1rem;
    background: var(--surface);
    border: 1px solid var(--border);
    border-radius: 0.5rem;
    font-size: 0.875rem;
    color: var(--text-muted);
    max-width: 480px;
    margin-left: auto;
    margin-right: auto;
}

.progress-bar {
    width: 100%;
    height: 3px;
    background: var(--surface-2);
    border-radius: 2px;
    margin-top: 0.625rem;
    overflow: hidden;
}
.progress-fill {
    height: 100%;
    background: var(--primary);
    border-radius: 2px;
    transition: width 0.3s ease;
}

/* ── Summary cards ── */
.summary-section {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
    gap: 1rem;
    margin-bottom: 2rem;
}

.card {
    background: var(--surface);
    border: 1px solid var(--border);
    padding: 1.25rem 1.5rem;
    border-radius: 0.5rem;
}
.card h3 {
    font-size: 0.75rem;
    font-weight: 500;
    text-transform: uppercase;
    letter-spacing: 0.06em;
    color: var(--text-muted);
    margin-bottom: 0.5rem;
}
.stat {
    font-size: 2rem;
    font-weight: 700;
    letter-spacing: -0.02em;
    font-family: var(--font-mono);
}

/* Security Issues card — turns red when issues > 0 */
.card.alert { border-color: var(--border); }
.card.alert.has-issues {
    background: var(--danger-bg);
    border-color: var(--danger);
}
.card.alert.has-issues h3 { color: var(--danger); }
.card.alert.has-issues .stat { color: var(--danger); }

/* ── Filter bar ── */
#filter-bar {
    margin-bottom: 1rem;
    text-align: right;
}
#filter-bar label {
    font-size: 0.8125rem;
    color: var(--text-muted);
    cursor: pointer;
    display: inline-flex;
    align-items: center;
    gap: 0.4rem;
    user-select: none;
}

/* ── Device cards ── */
.devices-section {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    gap: 1rem;
}

.device-card {
    background: var(--surface);
    border: 1px solid var(--border);
    padding: 1.25rem 1.5rem;
    border-radius: 0.5rem;
    transition: border-color 0.15s;
}
.device-card:hover { border-color: #444c56; }
.device-card.has-issues {
    border-color: var(--danger);
    background: linear-gradient(135deg, var(--surface) 0%, rgba(248,81,73,0.06) 100%);
}
.device-card.non-iot { display: none; }
.device-card.non-iot.show { display: block; }

.device-card h4 {
    font-size: 0.9375rem;
    font-weight: 600;
    margin-bottom: 0.4rem;
    color: var(--text);
    display: flex;
    align-items: center;
    gap: 0.5rem;
}
/* Device type icon — SVG inline via JS, not emoji */
.device-icon {
    width: 16px;
    height: 16px;
    stroke: var(--text-muted);
    fill: none;
    stroke-width: 1.5;
    flex-shrink: 0;
}
.device-info {
    font-size: 0.8125rem;
    color: var(--text-muted);
    margin-bottom: 0.2rem;
}
.device-info .mono {
    font-family: var(--font-mono);
    font-size: 0.75rem;
    color: var(--text-subtle);
    letter-spacing: 0.02em;
}

/* ── Risk badge ── */
.badge {
    display: inline-flex;
    align-items: center;
    padding: 0.2rem 0.6rem;
    border-radius: 0.25rem;
    font-size: 0.6875rem;
    font-weight: 600;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    margin-top: 0.5rem;
}
.badge-success { background: rgba(63, 185, 80, 0.15); color: #3fb950; }
.badge-danger  { background: rgba(248, 81, 73, 0.15);  color: #f85149; }
.badge-warning { background: rgba(210, 153, 34, 0.15); color: #e3b341; }
.badge-muted   { background: var(--surface-2);          color: var(--text-subtle); }

/* ── Findings ── */
.findings {
    margin-top: 1rem;
    padding-top: 1rem;
    border-top: 1px solid var(--border);
    display: flex;
    flex-direction: column;
    gap: 0.4rem;
}
.finding {
    display: flex;
    align-items: flex-start;
    gap: 0.5rem;
    font-size: 0.8125rem;
    color: var(--text-muted);
    line-height: 1.4;
}
/* Severity chip replaces old colored-background div */
.severity-chip {
    display: inline-block;
    flex-shrink: 0;
    padding: 0.1rem 0.4rem;
    border-radius: 0.2rem;
    font-family: var(--font-mono);
    font-size: 0.625rem;
    font-weight: 700;
    letter-spacing: 0.05em;
    text-transform: uppercase;
    line-height: 1.6;
}
.chip-critical { background: rgba(248,81,73,0.2);  color: #f85149; }
.chip-high     { background: rgba(248,81,73,0.12); color: #ff7b72; }
.chip-medium   { background: rgba(210,153,34,0.15);color: #e3b341; }
.chip-low      { background: rgba(88,166,255,0.1); color: #58a6ff; }
```

---

# 6. Change 2: index.html Edits (surgical)

Three additions to `index.html`. Do not alter anything else.

## 6.1 Add Font Links to `<head>`

Add immediately before `</head>`:

```html
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&family=JetBrains+Mono:wght@400;700&display=swap" rel="stylesheet">
```

## 6.2 Update `<header>` Structure

Replace the existing `<header>` block:

```html
<!-- REMOVE this: -->
<header>
    <h1>IoT Scout</h1>
    <p class="subtitle">Network Device Scanner</p>
</header>
```

With:

```html
<header>
    <h1>IoT<span class="accent">Scout</span></h1>
    <div class="header-right">
        <div id="network-info" class="network-pill" style="display:none;"></div>
    </div>
</header>
```

Note: `#network-info` moves from `<main>` into the header as a pill element. The `display:none` initial state is intentional — `detectNetwork()` in `app.js` will populate and show it.

## 6.3 Add Hero Section

Add the following as the first child inside `<main>`, before `<section class="scan-section">`:

```html
<div id="hero">
    <div class="hero-ring-wrap">
        <div class="hero-ring"></div>
        <div class="hero-ring"></div>
        <div class="hero-ring"></div>
        <div class="hero-icon">
            <svg viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
                <circle cx="12" cy="12" r="10"/>
                <circle cx="12" cy="12" r="6"/>
                <circle cx="12" cy="12" r="2"/>
                <line x1="12" y1="2" x2="12" y2="0"/>
                <line x1="22" y1="12" x2="24" y2="12"/>
            </svg>
        </div>
    </div>
    <p class="hero-title">IoT<span>Scout</span></p>
    <p class="hero-subtitle">Scan your local network for IoT devices and security vulnerabilities.</p>
</div>
```

The hero is hidden by `app.js` as soon as results arrive (see Section 7.4). It is visible only in the pre-scan state.

## 6.4 Remove Standalone `#network-info` from `<main>`

The existing `<div id="network-info" class="network-info"></div>` inside `<section class="scan-section">` must be removed — `#network-info` now lives in the header (added in 6.2). There must be exactly one element with `id="network-info"` in the document.

---

# 7. Change 3–7: app.js Edits (surgical — 5 targeted changes)

The agent must make exactly these changes and no others. Each change is described with the old code and the replacement. All other code is untouched.

## 7.1 Change 3 — `detectNetwork()`: Populate Header Pill

The network info now renders into the header pill element. Replace the existing `detectNetwork` function body:

**Old:**
```javascript
function detectNetwork() {
    window.go.main.DesktopApp.DetectNetwork().then(function(info) {
        if (info && info.preferred) {
            detectedSubnet = info.preferred.cidr;
            networkInfoEl.textContent = 'Detected network: ' + info.preferred.cidr +
                ' (' + info.preferred.interfaceName + ')';
        }
    }).catch(function(err) {
        networkInfoEl.textContent = 'Could not detect network: ' + err;
    });
}
```

**New:**
```javascript
function detectNetwork() {
    window.go.main.DesktopApp.DetectNetwork().then(function(info) {
        if (info && info.preferred) {
            detectedSubnet = info.preferred.cidr;
            networkInfoEl.innerHTML =
                '<span class="dot"></span>' +
                '<span>' + escapeHtml(info.preferred.cidr) + '</span>' +
                '<span style="color:var(--text-subtle)">·</span>' +
                '<span>' + escapeHtml(info.preferred.interfaceName) + '</span>';
            networkInfoEl.style.display = 'inline-flex';
            networkInfoEl.className = 'network-pill';
        }
    }).catch(function(err) {
        networkInfoEl.innerHTML = '<span class="dot"></span><span>' + escapeHtml(String(err)) + '</span>';
        networkInfoEl.style.display = 'inline-flex';
        networkInfoEl.className = 'network-pill error';
    });
}
```

Note: `escapeHtml` is applied to `cidr`, `interfaceName`, and the error string. The `.dot` span and separator span contain no user data.

## 7.2 Change 4 — `displayResults()`: Hide Hero, Toggle Alert Card State

Add hero hide and issue-card class toggle. Replace the existing `displayResults` function:

**Old:**
```javascript
function displayResults(report) {
    summarySection.style.display = 'grid';
    filterBar.style.display = 'block';

    document.getElementById('device-count').textContent = (report.summary && report.summary.total_devices) || 0;
    document.getElementById('iot-count').textContent = (report.summary && report.summary.iot_devices) || 0;

    var issueCount = 0;
    (report.devices || []).forEach(function(d) {
        (d.findings || []).forEach(function(f) {
            if (f.severity !== 'INFO') issueCount++;
        });
    });
    document.getElementById('issue-count').textContent = issueCount;

    devicesSection.innerHTML = '';
    (report.devices || []).forEach(function(device) {
        devicesSection.appendChild(createDeviceCard(device));
    });
}
```

**New:**
```javascript
function displayResults(report) {
    // Hide hero on first results
    var hero = document.getElementById('hero');
    if (hero) hero.style.display = 'none';

    summarySection.style.display = 'grid';
    filterBar.style.display = 'block';

    document.getElementById('device-count').textContent = (report.summary && report.summary.total_devices) || 0;
    document.getElementById('iot-count').textContent = (report.summary && report.summary.iot_devices) || 0;

    var issueCount = 0;
    (report.devices || []).forEach(function(d) {
        (d.findings || []).forEach(function(f) {
            if (f.severity !== 'INFO') issueCount++;
        });
    });
    document.getElementById('issue-count').textContent = issueCount;

    // Toggle alert card red state
    var alertCard = document.querySelector('.card.alert');
    if (alertCard) {
        if (issueCount > 0) {
            alertCard.classList.add('has-issues');
        } else {
            alertCard.classList.remove('has-issues');
        }
    }

    devicesSection.innerHTML = '';
    (report.devices || []).forEach(function(device) {
        devicesSection.appendChild(createDeviceCard(device));
    });
}
```

## 7.3 Change 5 — `createDeviceCard()`: Monospace for IP/MAC, Remove Emoji

Replace `getIcon(deviceType) + ' '` with a device-type SVG icon call, and wrap IP and MAC in `<span class="mono">`.

**Old:**
```javascript
    card.innerHTML =
        '<h4>' + getIcon(deviceType) + ' ' + escapeHtml(name) + '</h4>' +
        '<div class="device-info">' + escapeHtml(device.ip) + (device.mac ? ' &mdash; ' + escapeHtml(device.mac) : '') + '</div>' +
        '<div class="device-info">' + escapeHtml(deviceType) + '</div>' +
        '<span class="badge badge-' + risk.color + '">' + risk.text + '</span>' +
        renderFindings(device.findings);
```

**New:**
```javascript
    card.innerHTML =
        '<h4>' + getDeviceIcon(deviceType) + escapeHtml(name) + '</h4>' +
        '<div class="device-info"><span class="mono">' + escapeHtml(device.ip) + (device.mac ? ' &mdash; ' + escapeHtml(device.mac) : '') + '</span></div>' +
        '<div class="device-info">' + escapeHtml(deviceType) + '</div>' +
        '<span class="badge badge-' + risk.color + '">' + risk.text + '</span>' +
        renderFindings(device.findings);
```

Note: `risk.color` and `risk.text` come only from `getRiskBadge`'s hardcoded return values — this is unchanged from Phase 3.

## 7.4 Change 6 — `renderFindings()`: Severity Chips

Replace the old colored-background div with a severity chip + message layout.

**Old:**
```javascript
function renderFindings(findings) {
    var important = (findings || []).filter(function(f) { return f.severity !== 'INFO'; });
    if (!important.length) return '';
    return '<div class="findings">' + important.map(function(f) {
        var cls = (f.severity === 'LOW') ? 'finding low' : 'finding';
        var msg = f.humanMessage || f.message;
        return '<div class="' + cls + '">' + escapeHtml(msg) + '</div>';
    }).join('') + '</div>';
}
```

**New:**
```javascript
function renderFindings(findings) {
    var important = (findings || []).filter(function(f) { return f.severity !== 'INFO'; });
    if (!important.length) return '';
    return '<div class="findings">' + important.map(function(f) {
        var chipClass = 'chip-' + (f.severity || 'low').toLowerCase();
        var msg = f.humanMessage || f.message;
        return '<div class="finding">' +
            '<span class="severity-chip ' + chipClass + '">' + escapeHtml(f.severity || 'LOW') + '</span>' +
            '<span>' + escapeHtml(msg) + '</span>' +
            '</div>';
    }).join('') + '</div>';
}
```

Note: `chipClass` is constructed from `f.severity.toLowerCase()` which produces one of `critical`, `high`, `medium`, `low` — the CSS defines `.chip-critical`, `.chip-high`, `.chip-medium`, `.chip-low`. `f.severity` is a Go-defined enum value from the scanner, not user-controlled network data, but `escapeHtml` is still applied to it in the visible label as a defensive measure.

## 7.5 Change 7 — Replace `getIcon()` with `getDeviceIcon()`

Remove `getIcon()` entirely and add `getDeviceIcon()` which returns inline SVG strings. The old emoji map is deleted.

**Remove:**
```javascript
function getIcon(deviceType) {
    var icons = {
        'IP Camera': '📷', 'Router/Gateway': '🌐', 'Smart Speaker': '🔊',
        'Smart TV': '📺', 'Network Storage': '💾', 'Network Printer': '🖨️',
        'UPnP Device': '📡', 'MQTT IoT Device': '🔗', 'Unknown': '💻'
    };
    return icons[deviceType] || '📱';
}
```

**Add:**
```javascript
function getDeviceIcon(deviceType) {
    // All paths are static SVG strings — no user data interpolated
    var icons = {
        'IP Camera':       '<svg class="device-icon" viewBox="0 0 24 24"><path d="M14.5 12a2.5 2.5 0 1 1-5 0 2.5 2.5 0 0 1 5 0Z"/><path d="M22 8.5V6a2 2 0 0 0-2-2H4a2 2 0 0 0-2 2v12a2 2 0 0 0 2 2h16a2 2 0 0 0 2-2v-2.5M22 8.5l-4 3.5 4 3.5"/></svg>',
        'Router/Gateway':  '<svg class="device-icon" viewBox="0 0 24 24"><rect x="2" y="14" width="20" height="6" rx="1"/><path d="M6 14V9m4 5V7m4 7V9m4 5V7M6 7a6 6 0 0 1 12 0"/></svg>',
        'Smart Speaker':   '<svg class="device-icon" viewBox="0 0 24 24"><rect x="7" y="2" width="10" height="20" rx="2"/><circle cx="12" cy="14" r="2"/><line x1="12" y1="6" x2="12" y2="6"/></svg>',
        'Smart TV':        '<svg class="device-icon" viewBox="0 0 24 24"><rect x="2" y="4" width="20" height="14" rx="2"/><path d="M8 20h8M12 18v2"/></svg>',
        'Network Storage': '<svg class="device-icon" viewBox="0 0 24 24"><ellipse cx="12" cy="6" rx="8" ry="3"/><path d="M4 6v6c0 1.66 3.58 3 8 3s8-1.34 8-3V6"/><path d="M4 12v6c0 1.66 3.58 3 8 3s8-1.34 8-3v-6"/></svg>',
        'Network Printer': '<svg class="device-icon" viewBox="0 0 24 24"><path d="M6 9V2h12v7"/><rect x="6" y="14" width="12" height="8"/><path d="M6 18H4a2 2 0 0 1-2-2v-5a2 2 0 0 1 2-2h16a2 2 0 0 1 2 2v5a2 2 0 0 1-2 2h-2"/><line x1="18" y1="12" x2="18" y2="12"/></svg>',
        'UPnP Device':     '<svg class="device-icon" viewBox="0 0 24 24"><path d="M5 12.55a11 11 0 0 1 14.08 0"/><path d="M1.42 9a16 16 0 0 1 21.16 0"/><path d="M8.53 16.11a6 6 0 0 1 6.95 0"/><circle cx="12" cy="20" r="1"/></svg>',
        'MQTT IoT Device': '<svg class="device-icon" viewBox="0 0 24 24"><path d="m8 6 4-4 4 4"/><path d="M12 2v10.3"/><path d="M4.93 10.93A10 10 0 1 0 19.07 10.93"/></svg>',
        'Unknown':         '<svg class="device-icon" viewBox="0 0 24 24"><rect x="2" y="3" width="20" height="14" rx="2"/><path d="M8 21h8M12 17v4"/></svg>'
    };
    return (icons[deviceType] || icons['Unknown']) + ' ';
}
```

All SVG paths are static string literals. No user data is interpolated into the SVG content. The `class="device-icon"` attribute is hardcoded in the template strings, not derived from device data.

---

# 8. Verification

## 8.1 Code Inspection (Before wails dev)

Before running anything, the agent must verify:

```
Invariant check — run through each item in Section 3 and confirm:
- escapeHtml() present on: info.preferred.cidr, info.preferred.interfaceName,
  String(err) in detectNetwork; device.ip, device.mac, name, deviceType in
  createDeviceCard; f.severity, msg in renderFindings; data.message in
  progress handler
- .non-iot class: present in createDeviceCard className assignment
- .has-issues class: present in createDeviceCard classList.add
- .scan-status / .status-hidden: present in startScan and progress handler
- All 10 element IDs present in index.html
- getIcon() removed, getDeviceIcon() added — no references to getIcon remain
- Exactly one element with id="network-info" in index.html
```

## 8.2 Automated

```bash
# No Go changes — but confirm no accidental Go file touched
git diff --name-only | grep '\.go$'
# Expected: empty output

# Confirm only frontend files changed
git diff --name-only
# Expected: only src/cmd/iot-scout/frontend/* and src/cmd/iot-scout/frontend/index.html
```

## 8.3 Human Smoke Tests

Run `wails dev` and verify all of the following:

| ID | Test | Pass Condition |
|----|------|----------------|
| ST-U01 | Dark background | Window background is dark slate (`#0d1117`), not white |
| ST-U02 | Header pill | After load, subnet pill appears in header (e.g. `● 192.168.1.0/24 · en0`) |
| ST-U03 | Hero visible pre-scan | Pulse ring animation and "IoTScout" hero text visible before first scan |
| ST-U04 | Hero hides post-scan | After scan completes, hero disappears and results take full space |
| ST-U05 | Security Issues card neutral | When scan has 0 non-INFO findings, alert card has default dark border |
| ST-U06 | Security Issues card red | When scan has any CRITICAL/HIGH/MEDIUM/LOW findings, alert card turns red |
| ST-U07 | Severity chips render | Finding items show labeled chip (CRITICAL, HIGH, MEDIUM, LOW) next to message text |
| ST-U08 | IP/MAC in monospace | Device IP and MAC addresses render in JetBrains Mono, visually distinct from prose |
| ST-U09 | No emoji icons | Device cards show SVG icons, not 📷 🌐 📡 emoji |
| ST-U10 | IoT filter still works | Non-IoT cards hidden; toggle reveals them — behavior unchanged |
| ST-U11 | Scan flow unchanged | Button disables, progress shows, results render — all Phase 3 smoke tests pass |
| ST-U12 | XSS still blocked | `escapeHtml` test from Phase 3 ST-012 — malicious hostname renders as literal text |

## 8.4 Phase 3 Regression

Re-run Phase 3 smoke tests ST-001 through ST-015. All must still pass. Any failure is a regression introduced by this enhancement and must be fixed before the enhancement is considered complete.

---

# 9. Failure Modes

| Failure | Symptom | Cause | Fix |
|---------|---------|-------|-----|
| Fonts don't load | System font rendered, no Inter/JetBrains Mono | `wails dev` running offline or preconnect links wrong | Verify font link URLs; fonts degrade gracefully to system fonts |
| Hero doesn't hide after scan | Hero overlaps results | `document.getElementById('hero')` returns null | Verify `id="hero"` on the div added in Section 6.3 |
| Alert card never turns red | Always default border | `document.querySelector('.card.alert')` returns null | Verify `class="card alert"` on the Security Issues card in index.html |
| `getIcon is not defined` error | JS console error, cards broken | Old `getIcon` call left in `createDeviceCard` | Replace with `getDeviceIcon` per Section 7.3 |
| Duplicate `#network-info` | Two pills appear, or JS targets wrong one | Old element not removed from `<main>` | Remove per Section 6.4 |
| `chipClass` produces invalid class | Finding has no chip color | `f.severity` is an unexpected value | Default `(f.severity \|\| 'low').toLowerCase()` handles null |
| SVG icons invisible | `device-icon` class has no styles | CSS not fully replaced | Confirm full `styles.css` replacement, not patch |

---

# 10. Non-Goals

- Dark/light mode toggle — dark only
- Animated card entrance transitions
- Sorting or filtering by severity
- Any change to scan logic, Go bindings, or state machine
- Accessibility improvements (ARIA) — future work
- Persisting the hero hide state between scans (hero re-shows on window reopen — acceptable)

---

# 11. Commit

```bash
git add src/cmd/iot-scout/frontend/
git commit -m "feat: dark technical theme, severity chips, monospace fields, hero pre-scan state"
```

---

# 12. Change Log

| Date | Version | Change | Author |
|------|---------|--------|--------|
| 2026-02-23 | 1.0 | Initial spec — 7 UI enhancements across CSS/HTML/JS, surgical edit model, Phase 3 regression gate | |

---

*End of Specification — IoT Scout UI Enhancement*
