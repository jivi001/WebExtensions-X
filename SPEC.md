# Spec: WebExtensions-X — MVP v0.1.0
## Overview

This is the authoritative implementation contract for WebExtensions-X MVP v0.1.0. It consolidates requirements from PRD.md (product vision) and TRD.md (technical architecture) into a single reference for implementers.

**Relationship to other documents:**
- **PRD.md:** Product requirements, personas, use cases
- **TRD.md:** Technical reference with deep architectural decisions: permissions, data flow, failure modes
- **DESIGN.md:** Apple-inspired design system: UI/UX requirements
- **agents.md:** Repository workflow and quality rules; see [Compliance with agents.md](#compliance-with-agentsmd) below

**Core Concept:** A **Finding** is a detected security issue with severity, confidence, evidence, location, verification guidance, and remediation guidance. See [Finding Structure Interface](#finding-structure-interface) for the data model.

## Objective
The extension sends no attack traffic. All analysis runs locally on the user's device. Build a Manifest V3 Chrome extension that performs passive web security analysis while a user browses normally. Primary users: pentesters and bug bounty hunters. For each finding, show what it is, why it matters, how to verify, and how to fix. The MVP targets Chrome and Chromium browsers; Firefox is deferred.

### Acceptance Criteria (MVP v0.1 Gates)

**Performance (Measured on local OWASP Juice Shop):**
- ✓ First result (cookies + page signals) appears ≤2s after Scan click
- ✓ Content script main-thread block ≤50ms per scan step
- ✓ Secret scanner processes 5MB bundle in ≤30s (256KB chunks with progress)

**Functional Completeness (All Must Pass):**
- ✓ Cookie analyzer: flags HttpOnly, Secure, SameSite violations (0 false positives on tracking cookies)
- ✓ Fingerprinting: detects library name + version from globals, URLs, headers (≥90% accuracy on 20-page test set)
- ✓ CVE mapper: queries OSV.dev with mapped package data only, caches 24h, handles rate limits gracefully
- ✓ Secret scanner: detects AWS keys, GitHub tokens, Stripe secrets (≥90% precision on 50-sample fixture)

**Architecture (Non-Negotiable):**
- ✓ All analysis on-device (page URLs, cookie values, and script text are not sent to analysis services; an opt-in script re-fetch may send the observed script URL to its target site)
- ✓ Only mapped package name/ecosystem and version sent to OSV.dev
- ✓ Secrets masked (first 4 + last 4 chars) in UI, storage, exports
- ✓ Findings deduplicate on reload (stable ID via hash)
- ✓ Export to Markdown + JSON with masked secrets

**Gate:** All checkboxes must pass before v0.1 release to Chrome Web Store.

## Key Terminology

Project-specific terms used throughout this spec:

- **Finding:** One detected security issue with severity, confidence, evidence, location, why, verification, and fix (see [Finding Structure Interface](#finding-structure-interface))
- **Manifest V3:** Chrome's extension manifest model used by this MVP; it uses an ephemeral service worker instead of a persistent background page.
- **Passive analysis:** Watching traffic/code without sending attack requests or modifying page behavior
- **Content script (isolated world):** Extension script in page context but separate JavaScript scope; can read DOM, cannot access page's JS variables
- **MAIN world helper:** Extension script injected into page's own JS context; can access `window` globals like `jQuery` or `angular.version`
- **Entropy threshold (4.5 bits/char):** Measure of randomness; high entropy suggests secret (API key, token) vs. English text (~4.0 bits/char)
- **OSV.dev:** Google's open-source vulnerability database API; receives only the mapped package name/ecosystem and version, never page URLs or other page data
- **Retire.js:** Bundled JSON database of vulnerable JavaScript libraries; enables offline CVE lookups

- **CVE (Common Vulnerabilities and Exposures):** Standardized ID for a known security flaw (for example, CVE-2024-1234)
- **CSP (Content Security Policy):** A page policy that restricts scripts and other resources; it can limit page-context helpers.
- **Severity and confidence:** Severity describes potential impact if the issue is real; confidence describes how strong the evidence is. They must remain separate.

For broader security terminology, see the [OWASP Glossary](https://owasp.org/www-community/vulnerabilities/).

**Tools & Frameworks:**
- **WXT**: Build framework for browser extensions (handles manifest generation, hot reload, TypeScript)
- **Svelte**: UI framework that compiles to vanilla JS (smaller output than React/Vue)
- **TypeScript strict**: TypeScript with strictest type-checking enabled
- **Vitest**: Fast unit test runner (Vite-based, drop-in Jest replacement)
- **Playwright**: Browser automation for end-to-end testing (launches real Chrome with extension loaded)

**Compliance with agents.md:**
`agents.md` is a repository-level process guide, not the implementation contract. Svelte remains the selected UI framework. Applicable process rules remain in force:
- Apply the Apple-inspired principles in DESIGN.md.
- Use a spec → plan → incremental implementation → test → review → release workflow with human approval before release.
- Use unit, integration, browser, accessibility, performance, and security testing before release.
- Keep secrets out of source, storage, logs, exports, and test artifacts.
- Treat this SPEC.md as authoritative for WebExtensions-X implementation choices; update PRD.md and TRD.md when they drift.

## Architecture Overview

**Three Component Model (referenced from TRD section 2):**
1. **Service Worker** - Background orchestrator: reads cookies, manages caches, coordinates analyzers, stores findings
2. **Content Script (isolated world)** - Injected into page: reads DOM (meta tags, script URLs), cannot access page JavaScript
3. **MAIN World Helper** - Injected into page context: reads library globals (jQuery.version, angular.version) via `chrome.scripting.executeScript`

**Data Flow (7-step scan sequence):**
1. At service-worker startup, register one non-blocking `webRequest.onHeadersReceived` listener at global scope. The handler records only selected headers for origins with an active scan state.
2. User clicks Scan → popup explains the requested access and calls `chrome.permissions.request` for the current origin.
3. The popup sends `SCAN_REQUEST` to the service worker with the permission result. If access is granted, the service worker writes `scanState:<tabId>` with `phase: "permission-granted"`, confirms the global listener is registered, writes `headerListener: "active"`, waits for the storage write to complete, then reloads the tab once with the normal cache policy. Header capture is best effort because browser memory-cache responses may not emit webRequest events; report headers as unavailable when no matching event arrives. If denied, it records `phase: "limited-permission"` and continues with reduced coverage.
4. Service worker reads cookies via `chrome.cookies.getAll`.
5. Content script sends a DOM snapshot (meta tags, script URLs, and inline scripts).
6. MAIN world helper reads library globals and returns validated signals to the service worker.
7. Four analyzers run in parallel, findings are stored in `chrome.storage.local`, and scan state is cleared after completion. If the tab navigates during the scan, cancel stale work for the old document and retain only findings already committed.

**Failure Modes & Degradation (referenced from TRD sections 7, 10):**
- **Service worker stops** (Chrome may kill idle workers): Mark the scan incomplete, retain findings already committed, and let the next user action start a fresh scan. Do not persist in-progress scan state in the MVP.

- **OSV.dev rate limit or outage:** Use bundled data or the most recent cache, show its age, and label the lookup as incomplete.
- **MAIN world helper blocked:** Use URL, header, and metadata signals with lower confidence.
- **Large bundle timeout (>30 seconds):** Return partial results and show "Scan incomplete" with the skipped size.
- **Permission denied:** Continue with reduced coverage and show the affected modules.
- **Malformed external data:** Validate the OSV and KEV response shape; discard invalid data and keep the previous valid cache.

### Message Protocol

All extension messages use `{ type, tabId, payload }`. The service worker validates `type`, `tabId`, and payload shape before handling a message.

| Message | Sender → receiver | Payload |
|---|---|---|
| `SCAN_REQUEST` | Popup → service worker | `tabId`, permission result, requested origin |
| `PAGE_SNAPSHOT` | Content script → service worker | Script URLs, inline scripts, metadata |
| `MAIN_WORLD_SIGNALS` | MAIN helper → service worker | Validated library signals only |
| `FINDINGS_UPDATED` | Service worker → popup | Finding count and IDs |

### Storage Schema

| Key | Contents | Retention |
|---|---|---|
| `settings` | Enabled modules and scan preferences | Until changed |
| `osv:<packageName>@<ecosystem>@<version>` | Validated CVE result and fetch time | 24 hours |
| `kev` | Validated CVE ID set and fetch time | 24 hours |
| `findings:<origin>` | Up to 500 masked Finding records per origin and 5,000 total; `origin` is a validated serialized origin, not a full URL | Until cleared; oldest records removed first when a cap is reached |

The extension never stores script text, raw cookie values, or raw secret values.

## Permission Model

**Required permissions (referenced from TRD section 4):**
- `cookies` - Read cookie names and security attributes, including HttpOnly cookies; host access is required
- `scripting` - Inject the fixed MAIN world helper via `chrome.scripting.executeScript`
- `storage` - Store settings, cache, and capped findings in `chrome.storage.local`
- `activeTab` - Temporary access after a user click for the current tab
- Optional host permission for the current origin - Requested only after the user clicks Scan
- Fixed host access for `https://api.osv.dev/*` and the official CISA feed URL - Used only for vulnerability data
- Optional host access: `https://*/*` and `http://*/*`; request only the current origin pattern at scan time

**Permission flow:**
1. The popup explains the data each permission enables before requesting access.
2. The user clicks Scan and approves access for the current origin.
3. If access is granted, the service worker writes `{ phase: "permission-granted", headerListener: "pending" }` to `scanState:<tabId>`. It confirms the global `webRequest.onHeadersReceived` listener is registered, writes `{ headerListener: "active" }`, waits for the storage write to complete, then reloads the tab once to capture response headers.
4. If the listener cannot be registered or confirmed, the service worker does not reload and reports headers as unavailable.
5. If access is denied, the scan continues with reduced coverage and reports the affected modules.
6. The extension is disabled in incognito by default.

**Privacy boundaries:**

| Data Type | Read Locally | Transmitted | Stored |
|-----------|--------------|-------------|--------|
| Cookie names/flags | ✓ | ✗ Never | ✗ Findings only (no values) |
| Response headers | ✓ | ✗ Never | ✗ Findings only |
| Inline script bodies | ✓ content script reads page DOM | ✗ Never | ✗ Dropped after scan |
| External script bodies (opt-in) | ✓ service worker fetch with `cache: "force-cache"` | ✗ Script text never sent; a cache miss or revalidation may appear in target server log | ✗ Dropped after scan |
| Mapped package name/ecosystem + version | — | ✓ OSV.dev only | ✓ Cached 24h |
| Page URL, cookie values, script text | ✓ (local analysis) | ✗ Never | ✗ Never |

*External script acquisition uses `cache: "force-cache"`; it is opt-in because a cache miss or revalidation may create a target-site request and appear in its server log. The script body remains in local extension memory and is never sent to OSV.dev or any other analysis service.*

**Only the mapped package name/ecosystem and version are sent to OSV.dev; full navigation URLs, cookie values, and script text are never sent outside the extension for analysis.**

## UI/UX Requirements

The popup and options page follow DESIGN.md:

- Explain requested access before `chrome.permissions.request` runs.
- Show a visible reload notice after access is granted and the tab reloads for header capture.
- Show per-module reduced-coverage states when access is denied or an analyzer fails:
  - **Cookie analyzer:** cookie findings are unavailable when the required host access is denied.
  - **Fingerprinting:** DOM script URLs and page globals may remain available through the active tab; response-header signals are unavailable without host access. Cross-origin external script text is unavailable without host access and opt-in re-fetch.
  - **CVE mapper:** no direct page permission loss; coverage depends on detected products and versions, and OSV fallback requires fixed OSV host access.
  - **Secret scanner:** inline scripts may remain available through the active tab; external script bodies require host access and explicit re-fetch opt-in.
- Let users enable or disable each analyzer in the options page.
- Show detection source, confidence, data source, cache age, and lookup status in findings.
- Keep severity and confidence as separate text labels; never communicate either through color alone.
- Keep findings and secret evidence masked in the UI, exports, logs, and accessibility labels.
- Support keyboard navigation, visible focus, dark mode, forced colors, browser zoom, and reduced motion.
- Keep the extension disabled in incognito by default.

## Infrastructure Economics

| Resource | Free Tier Limit | MVP Mitigation | Scale Plan |
|----------|----------------|----------------|------------|
| OSV.dev API | Provider limit must be verified before release | 24h cache, fallback to bundled Retire.js | Backoff at 2s, 4s, 8s for a maximum of 3 retries; then use the last cache and wait 1 hour before retrying. Show cache age. |
| CISA KEV list | ~200KB/day | Cache 24h, fail gracefully if unavailable | Weekly refresh post-MVP |
| GitHub Actions | 2000 min/month | Run E2E only on main, not PRs | Self-hosted runner at 1K+ monthly commits |
| Chrome Web Store | Current fee must be verified before release | Publish only after account, fee, and policy review | N/A |

**Scale scenario:** At 10K users, 5 scans/day, and 3 libraries/site, an uncached design could issue 150K OSV queries/day. The provider limit and actual cache-hit rate must be measured before relying on this estimate; the MVP mitigates load with 24-hour caching, bounded retries, and stale-data messaging.

**Retire.js license audit:** Complete the bundled-data provenance and license audit before v1 release; the upstream repository license alone does not establish the license for every embedded data source. Re-audit quarterly and whenever Retire.js publishes a new release. If any data becomes incompatible with MIT redistribution, remove it from the bundle and promote OSV.dev to primary within one release cycle. Record the audit result and source versions in the release checklist.

## Finding Structure Interface

This is the canonical contract; every analyzer returns this shape.

```typescript
type Severity = 'info' | 'low' | 'medium' | 'high' | 'critical';
type Confidence = 'low' | 'medium' | 'high';

interface Finding {
  id: string;
  module: 'cookie' | 'tech' | 'cve' | 'secret';
  ruleId: string;
  title: string;
  severity: Severity;
  confidence: Confidence;
  evidence: string;          // masked for secrets
  location: string;           // normalized origin/path or script path + line; no query/fragment
  why: string;
  verify: string;
  fix: string;
  origin: string;
  firstSeen: number;
  lastSeen: number;
  evidenceHash?: string;      // non-reversible deduplication metadata
  refs?: string[];
  cwe?: string;
}
```

Secrets are masked before a Finding is stored, rendered, logged, or exported. `location` strips query strings and fragments before storage. `evidenceHash`, when present, is a one-way digest used for local deduplication, is not a secret-protection boundary, and is never exported.

## Tech Stack

| Component | Choice | Rationale |
|---|---|---|
| Platform | Manifest V3, service worker, content scripts, MAIN world helper | Required Chrome extension model; separates page access from extension logic |
| Language | TypeScript strict | Protects the shared `Finding` contract and catches type errors at build time |
| Build | WXT | Generates extension scaffolding and manifests, supports hot reload, and provides typed browser APIs |
| UI | Svelte | Compiles to a small browser bundle; the popup does not need a large component runtime |
| Storage | `chrome.storage.local` | Sufficient for settings, short-lived cache, and capped findings in the MVP |
| Unit tests | Vitest | Fast TypeScript test runner for pure analyzer functions |
| E2E tests | Playwright | Loads the built extension in a real Chromium browser |
| CVE data | Retire.js bundled data plus OSV.dev fallback | Supports offline lookup first and a network lookup when no bundled mapping exists |

### Configuration Constants

| Constant | MVP value | Review trigger |
|---|---:|---|
| Secret entropy threshold | 4.5 bits/character | Re-tune if the fixture precision target fails |
| Maximum script size | 5 MB | Revisit if large-site scans exceed the 30-second target |
| OSV and KEV cache lifetime | 24 hours | Revisit if freshness or rate limits become a problem |
| Secret scan chunk size | 256 KB | Keep unless performance tracing shows a better boundary |

## Commands

Standard npm scripts (requires Node.js 18+ and dependencies installed via `npm ci`):

```bash
# Build
npm run build

# Test
npm test -- --coverage
npm run test:unit
npm run test:e2e

# Lint
npm run lint
npm run lint -- --fix

# Development
npm run dev

# Local safeguards
npm run setup:hooks
npm run check:secrets
```

## Project Structure

```text
src/
  background/        # Service worker, message router, OSV client
  content/           # Content script (isolated world)
  injected/          # MAIN world helper (reads page globals)
  devtools/          # Reserved directory; full DevTools panel deferred to v1.2
  core/
    finding.ts       # Finding type, ID hashing, deduplication
    store.ts         # Findings store (chrome.storage.local)
  analyzers/
    cookies/         # Rules + analyzer
    fingerprint/     # Rules JSON + matcher
    cve/             # Retire.js lookup, OSV fallback, KEV check
    secrets/         # Rules, entropy, masking
  ui/                # Popup and options
  export/            # Markdown and JSON writers
data/
  retire/            # Bundled vulnerability data
tests/
  unit/              # Vitest unit tests (*.test.ts)
  integration/       # Mocked chrome.* API tests
  e2e/               # Playwright tests (*.spec.ts)
  fixtures/          # Test data
    secrets/         # 50-sample-set.json
    versions/        # 20-known-sites.json
    cookies/         # tracking-list.json
    clean-sites.json # False-positive check targets
```

## Code Style
- TypeScript strict, pure analyzer functions
- Finding type is shared across modules
- Rules are JSON data, not code
- One-line comment for why, not how
- Mask secrets in UI and exports: first 4 + last 4 chars

### Analyzer Function Signature (all analyzers use this pattern)

The following is an illustrative contract excerpt. `Cookie` and the helper functions are defined in the cookie analyzer module; the shared `Finding`, `Severity`, and `Confidence` types appear in the canonical interface below.

```typescript
type Cookie = {
  name: string;
  httpOnly: boolean;
  secure: boolean;
  sameSite: string;
  domain: string;
  path: string;
};

type Analyzer<TInput> = (input: TInput, ctx: AnalyzerContext) => Finding[];

function isSessionLike(cookie: Cookie): boolean {
  return /(sess|sid|auth|token|jwt)/i.test(cookie.name);
}

function hashFindingId(module: string, ruleId: string, location: string): string {
  return `${module}:${ruleId}:${location}`; // Synchronous placeholder; production uses a stable SHA-256 digest.
}

interface AnalyzerContext {
  origin: string;
  protocol: 'http' | 'https';  // Required for Secure flag validation
  timestamp: number;
}

// Example: Cookie analyzer
const cookieAnalyzer: Analyzer<Cookie[]> = (cookies, ctx) => {
  return cookies
    .filter(isSessionLike)
    .filter(cookie => !cookie.httpOnly)
    .map(cookie => ({
      id: hashFindingId('cookie', 'no-httponly', `${ctx.origin}:${cookie.name}`),
      module: 'cookie',
      ruleId: 'cookie.session.no-httponly',
      title: 'Session cookie is readable by scripts',
      severity: 'medium',
      confidence: 'high',
      evidence: cookie.name,
      location: ctx.origin,
      why: 'A session cookie without HttpOnly can be read by injected scripts.',
      verify: 'Inspect the Set-Cookie response and confirm that HttpOnly is absent.',
      fix: 'Add the HttpOnly attribute to the session cookie.',
      origin: ctx.origin,
      firstSeen: ctx.timestamp,
      lastSeen: ctx.timestamp,
    }));
};
```

### Secret Masking Function

```typescript
function maskSecret(secret: string): string {
  if (secret.length <= 12) return '****';
  return secret.slice(0, 4) + '****' + secret.slice(-4);
}
```

## Testing Strategy

### Test Targets

| Target | Purpose | Coverage |
|--------|---------|----------|
| OWASP Juice Shop (local) | Known vulnerabilities (planted secrets, old jQuery) | All 4 modules |
| Local fixture pages | Exercise fingerprinting and secret scanning without third-party traffic | Fingerprinting, secrets |
| React app with CSP (local) | CSP blocks `unsafe-inline`, test MAIN world helper | Fingerprinting globals |
| GitHub.com (passive only) | Real-world CSP, CDN headers, false-positive check | Cookies, fingerprinting |
| Extension options page | Self-test, ensure no false positives on own code | All 4 modules |

### Clean Site List (False-Positive Check)
Defined in `tests/fixtures/clean-sites.json`:
1. github.com (modern SPA, strict CSP)
2. stackoverflow.com (jQuery, CDN scripts)
3. mozilla.org (security-focused site)
4. cloudflare.com (CDN provider, representative headers)
5. Extension's own options page (self-test)

Public-site checks are passive only and require no opt-in script re-fetch. Run them only when permitted by the site's terms and the test plan.

**Expectation:** 0 high-severity findings, ≤3 low/info findings (acceptable: public API keys, version detection)

### CSP Compatibility Test
- Test page with CSP `script-src 'self'` (blocks inline scripts, eval)
- Verify MAIN world helper still reads library globals
- Verify content script can inject helper via `chrome.scripting.executeScript`
- **Expected result:** Fingerprinting still detects libraries; if blocked, fall back to URL/header signals with "low confidence" label

## Boundaries (Enforced)

| Rule | Enforcement Mechanism | Verification |
|------|----------------------|--------------|
| Run tests before commits | Pre-commit hook: `npm test` | CI blocks if tests fail |
| Validate inputs, mask secrets | `Finding` constructor auto-masks, ESLint validates | `npm run check:secrets` scans for leaked patterns |
| Never commit full secrets | Pre-commit and CI scan source, generated artifacts, ordinary project files, and `tests/fixtures/**` for credential patterns; fixture files additionally use approved fake values and a local format/negative-validation test | CI fails on credential matches or invalid fixture metadata; no live provider calls are made from CI |
| Ask first: add dependencies | Manual PR review + `npm audit` in CI | Branch protection rule |
| Never send page URL off-device | Integration test: mock fetch, assert no URL in payloads | `tests/integration/privacy.test.ts` |
| Never run page strings as code | ESLint rule: no `eval()`, `Function()`, or string-based timers | CI lint check |

**Setup:** Run `npm run setup:hooks` after clone to install pre-commit hooks.

## Success Criteria (Measured via Test Fixtures)

Metrics verified using fixtures in `tests/fixtures/`:

| Metric | Target | Measurement Method | Fixture Location |
|--------|--------|-------------------|------------------|
| Secret scanner precision (high-confidence rules) | ≥90% | TP / (TP + FP) on 50-sample set | `tests/fixtures/secrets/50-sample-set.json` |
| Secret scanner noise (clean sites) | 0 high-severity findings | Manual scan of the five targets in [Clean Site List](#clean-site-list-false-positive-check) | `tests/fixtures/clean-sites.json` |
| CSP compatibility | MAIN world helper reads globals or falls back to URL/header signals with low confidence | Manual test with `script-src 'self'`; verify no extension CSP violation is emitted | `tests/e2e/csp.spec.ts` |
| Version accuracy | ≥90% correct | 18+ matches on 20-page test set with known versions | `tests/fixtures/versions/20-known-sites.json` |
| Cookie false positives | 0 on tracking cookies | Test against known list | `tests/fixtures/cookies/tracking-list.json` |
| Time to first result | ≤2s | Playwright timing on Juice Shop | `tests/e2e/performance.spec.ts` |
| Main-thread block | ≤50ms per step | Chrome DevTools Performance trace | `tests/e2e/performance.spec.ts` |

**Fixture maintenance:** Fixtures versioned in repo. Update quarterly or when Chrome/libraries change behavior.

## Out of Scope (MVP v0.1)

Deferred features with tentative timelines:
- Active exploitation / fuzzing → v2.0+ (requires legal review)
- JWT, CORS, DOM sink modules → v1.1 (post-MVP)
- DevTools panel → v1.2 (after options page stabilizes)
- Firefox build → v1.1 (after Chrome release, 6-week offset)
- Cloud backend, user accounts → no current plan
- Mobile browsers → no current plan (Manifest V3 unsupported)

## Rollback Strategy

### Version Pinning
- All dependencies pinned in `package-lock.json`
- Major updates: test in separate branch, manual approval before merge
- Chrome API changes: monitor [Chromium blog](https://developer.chrome.com/blog/), test in Canary channel

### Extension Release Strategy
- Blue/green releases: publish new version, keep v0.9 available for 2 weeks (Chrome Web Store supports multiple versions)
- Feature flags: risky analyzers (secret scanner, CVE mapper) have kill switches in options page
- Degradation behavior:
  - Secret scan timeout (>30s): return partial results + warning "Large file, scan incomplete"
  - OSV.dev down: show cached data + age "CVE data from cache (4h old)"
  - Permission denied: show "Limited scan: cookies unavailable" + link to re-request

### Chrome API Monitoring
- Subscribe to the Chromium Extensions developer group for API change announcements.
- Test the extension in Chrome Canary monthly.
- Review Chrome Platform Status quarterly for changes affecting Manifest V3, cookies, scripting, and webRequest.
- When a breaking change is announced, test in Canary and plan mitigation before Stable release.

### Decision Review Cadence
"Release Decisions" reviewed:
- Quarterly or after 1000 active users
- When Chrome ships a new manifest generation or major API changes
- When a dependency has a critical CVE or becomes unmaintained

Example triggers:
- Svelte v6 breaks build → evaluate migration cost vs. staying on v5
- 5MB script limit causes timeouts on 10% of sites → increase to 10MB or add skip warning
- Entropy 4.5 generates false positives → re-tune on expanded fixture set

## Release Decisions
- Product name: **WebExtensions-X**
- License: MIT, subject to dependency and bundled-data license review before release
- Browser: Chrome and Chromium browsers for MVP; Firefox remains deferred
- Build: WXT
- UI: Svelte
- Entropy threshold: 4.5 bits/character, validated against the 50-sample fixture
- Script scan limit: 5 MB per script; larger files are skipped with a visible warning
- KEV source: Official CISA feed, cached for 24 hours
- Fingerprint rules: Original rules plus reviewed community patterns; each reused rule must pass provenance and license review
