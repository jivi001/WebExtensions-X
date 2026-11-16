# TRD: WebExtensions-X

Nov 15, 2026 · @Jivitesh

## 1. Overview

This document defines how to build the four MVP modules of WebExtensions-X. The PRD says what to build; SPEC.md is the authoritative implementation contract; this TRD supplies deeper technical detail.

**Technical goals**

- Use Manifest V3 on Chrome and Chromium browsers for the MVP. Firefox is deferred.
- Stay passive. The extension sends no attack traffic; cached script reads are local, while an explicitly opt-in re-fetch may create one request for a script URL already observed on the page and may appear in the target site's logs.
- Analyze locally. Analysis-service activity is limited to OSV.dev lookups containing the mapped package name/ecosystem and version and the CISA KEV feed download. An explicitly opt-in script re-fetch contacts the target site only; no page URL, cookie value, or script text is sent to an analysis service.
- Use the canonical `Finding` type defined in SPEC.md for all modules.
- Write analyzers as pure functions. Test them without a browser.
- Keep page work bounded. Run 256 KB secret-scan chunks in the service worker and add a Web Worker only if the measured 5 MB test cannot meet the budget.

| In scope (MVP) | Out of scope (MVP) |
| --- | --- |
| Cookie analyzer | Active path checks |
| Fingerprinting from globals, URLs, headers, and meta tags | JWT, CORS, and DOM sink modules |
| CVE mapping with OSV.dev and a CISA KEV check | Cloud backend and user accounts |
| JS secret scanner with masking | AI explanations |
| Popup UI and Markdown/JSON export | DevTools panel and Firefox build |

## 2. System architecture

&#91;embedded content: architecture · 3 areas, 1 outside service\]

The popup starts a scan and reads findings through the service worker. The service worker reads cookies, selected response headers, and script files, runs the analyzers, and writes every masked finding to the store. The CVE step sends only the mapped package name/ecosystem and version to OSV.dev. The secret scan runs in bounded chunks in the service worker and returns masked findings; the implementation must add a worker only if performance testing proves it necessary.

**Data flow for one scan**

1. The user clicks Scan in the popup. The popup sends `SCAN_REQUEST`.
2. The service worker reads cookies and starts the content script and the MAIN world helper after the permission/reload interlock completes.
3. The content script sends the script list and meta tags. The helper sends library globals.
4. The service worker receives selected headers and the content script's inline script text. The analyzers build the fingerprint and cookie findings.
5. Each detected product and version goes to the CVE mapper. It returns masked CVE findings.
6. The service worker scans script text in bounded 256 KB chunks and returns masked secret findings. A Web Worker is added only if the performance gate requires it.
7. The store merges all findings by ID and tells the popup to update.

## 3. Technology stack and project structure

The stack is resolved in SPEC.md. Any release blocker below is a validation task, not an unresolved technology choice.

| Area | Choice | Reason |
| --- | --- | --- |
| Extension platform | Manifest V3, service worker, content scripts, MAIN world helper | Required by Chrome. DevTools integration is deferred; script bodies come from inline content or an explicitly opt-in cached re-fetch |
| Language | TypeScript (strict mode) | Types protect the shared `Finding` format |
| Build | WXT | Handles extension scaffolding, manifest generation, hot reload, and typed browser APIs |
| UI | Svelte with plain CSS | The popup is small; a compiled UI keeps the extension bundle focused |
| JS library vulnerability data | License-audited Retire.js data, bundled at build time; OSV.dev fallback | Works offline first; sends only mapped package name/ecosystem and version when the fallback is used |
| Version compare | A small custom function for the range format of the data | OSV compares versions on its server. A library is not needed |
| Storage | `chrome.storage.local` for settings, cache, and findings | Findings are small. Move to IndexedDB when you add history across sites |
| Lint and format | ESLint, Prettier | Consistent code |
| Unit tests | Vitest | Fast, TypeScript native |
| End-to-end tests | Playwright with the extension loaded. Write 3 to 5 tests only | Extension tests in CI can be unstable |
| CI | GitHub Actions | Free for open source |

**Removed from the first draft**

| Item | Reason |
| --- | --- |
| Tailwind CSS | A small popup does not need it |
| Acorn | No MVP module parses an AST. Minified bundles make it slow |
| `csp-evaluator` | The CSP module is out of the MVP |
| `semver` | See version compare above |
| Offscreen document and Web Worker | The MVP uses bounded service-worker chunks. Add a worker only if the 5 MB performance test proves the service worker cannot meet the budget |
| IndexedDB | See storage above |

**Folder structure**

```text
src/
  background/        service worker, message router, OSV client
  content/           content script (isolated world)
  injected/          MAIN world helper (fixed path walker)
  devtools/          Reserved directory; full DevTools panel deferred to v1.2
  core/
    finding.ts       Finding type, id hashing, dedupe
    store.ts         findings store (chrome.storage.local)
  analyzers/
    cookies/         rules + analyzer
    fingerprint/     rules (JSON) + matcher
    cve/             Retire.js lookup, OSV fallback, KEV check
    secrets/         rules, entropy, masking
  ui/                popup and options page
  export/            Markdown and JSON writers
data/
  retire/            bundled vulnerability data (build step)
tests/
  fixtures/          50 secret samples, cookie sets, test pages
```

## 4. Manifest V3 configuration and permissions

Ask for the fewest permissions that work. Chrome Web Store reviewers and users check this. Ask for host access for one site at a time, when the user clicks Scan.

| Permission | Used for | Note |
| --- | --- | --- |
| `cookies` | Read all cookies, including HttpOnly, with `chrome.cookies.getAll` | Requires the `cookies` permission and matching host access; validate on a real Chrome build |
| `scripting` | Run the helper in the page MAIN world to read library globals | Content scripts cannot read page globals |
| `storage` | Settings, cache, and findings |  |
| `activeTab` | Temporary access to the current tab after a user click | It does not replace host access for cookies or response-header capture; denied scans report reduced coverage |
| `webRequest` (observe only) | Read selected response headers (`Server`, `X-Powered-By`) | Needs host access. Register one non-blocking listener at service-worker startup with `responseHeaders`; scan state filters the current origin. Reload only after access and active state are persisted |
| Optional host permissions `https://*/*`, `http://*/*` | Lets the extension ask for one origin at a time with `chrome.permissions.request` | Not granted at install; request only the current origin pattern |
| Host permission `https://api.osv.dev/*` | CVE fallback lookup | Fixed, narrow |
| Host permission `https://www.cisa.gov/*` | Download of the CISA KEV list | Confirm the exact list URL before release |
| `devtools_page` | Not included in the MVP | Reconsider only when the deferred DevTools panel is promoted into scope |

The extension manifest must not include a `devtools_page` entry in the MVP.

**Permission flow for one scan**

1. The user clicks Scan in the popup.
2. The popup explains the requested access and calls `chrome.permissions.request` for the current origin; this call needs the user click.
3. If access is granted, write `scanState:<tabId>` with `{ phase: "permission-granted", headerListener: "pending" }`.
4. Register the `webRequest.onHeadersReceived` listener for the origin, then persist `{ headerListener: "active" }` and wait for that write to complete.
5. Only after the listener is active and persisted, reload the tab once with the normal cache policy. If registration cannot be confirmed, do not reload and report headers as unavailable. Header capture remains best effort because browser memory-cache responses may not emit webRequest events.
6. Read cookies and run the content script and MAIN world helper. Persist scan phase before each asynchronous boundary and resume incomplete work after service-worker restart.
7. If access is denied, use reduced coverage and show the affected modules; do not claim that `activeTab` provides cookie or header access it does not provide.

The `offscreen` permission is not used in the MVP.

```json
{
  "manifest_version": 3,
  "name": "WebExtensions-X",
  "version": "0.1.0",
  "background": { "service_worker": "background.js", "type": "module" },
  "action": { "default_popup": "popup.html" },
  "permissions": ["cookies", "scripting", "storage", "activeTab", "webRequest"],
  "host_permissions": ["https://api.osv.dev/*", "https://www.cisa.gov/*"],
  "optional_host_permissions": ["https://*/*", "http://*/*"],
  "content_security_policy": {
    "extension_pages": "script-src 'self'; object-src 'self'"
  }
}
```

The manifest is a starting point. Test each permission on a real build before release. WXT generates the final manifest; the generated MVP manifest must preserve this permission boundary and must not add `devtools_page`.

## 5. Data model

**Finding.** All modules return this type. The ID is a hash of module, rule ID, and location. The same issue on reload gets the same ID, so the store can merge it.

The canonical `Finding` contract is defined in SPEC.md. TRD implementations must use that type without adding or removing fields. In particular, `verify` is required and `evidenceHash` is internal metadata that is never exported.

**Fingerprint rule (JSON).** Rules are data, not code. This lets contributors add rules without TypeScript. A page global is a path array. A fixed helper walks window by property name. The extension never evaluates a rule string.

```json
{
  "product": "jQuery",
  "vulnData": {
    "retire": "jquery",
    "osv": { "ecosystem": "npm", "name": "jquery" }
  },
  "signals": [
    { "type": "global", "path": ["jQuery", "fn", "jquery"], "confidence": "high" },
    { "type": "url", "regex": "jquery[-.]([0-9]+\\.[0-9]+\\.[0-9]+)(\\.min)?\\.js", "confidence": "high" },
    { "type": "header-comment", "regex": "jQuery v([0-9.]+)", "confidence": "high" }
  ]
}
```

**Secret rule (JSON).**

```json
{
  "ruleId": "secret.aws.access-key",
  "regex": "AKIA[0-9A-Z]{16}",
  "severity": "high",
  "confidence": "high",
  "publicByDesign": false,
  "keywordNear": null
}
```

**Storage schema**

| Store | Where | Content | Lifetime |
| --- | --- | --- | --- |
| Settings | `chrome.storage.local` | Scan mode, scope list, enabled modules | Until the user changes it |
| OSV cache | `chrome.storage.local` | Key `packageName@ecosystem@version`. Value: CVE list and fetch time | 24 hours |
| KEV list | `chrome.storage.local` | CVE ID set and fetch time | 24 hours |
| Findings | `chrome.storage.local`, key `findings:<origin>` | Up to 500 masked `Finding` records per origin and 5,000 total. `origin` is a validated serialized origin; remove the oldest records first when a cap is reached | Until the user clears them |
| Secret evidence metadata | `chrome.storage.local` | Non-reversible hash and masked value only. Never the full secret | Same as findings |
| Scan state | `chrome.storage.local`, key `scanState:<tabId>` | Phase, permission state, listener state, completed analyzers, pending CVE keys, last update | Until completion or 5 minutes idle |
| Retire.js data | Bundled file in the extension | License-audited library vulnerability ranges | Updated with each release; audit provenance before release |

## 6. Module designs

### 6.1 Cookie analyzer

**Input.** `chrome.cookies.getAll({ url })` for the active tab. **Output.** `Finding[]`.

1. The service worker reads all cookies for the site.
2. The analyzer removes cookies on the tracking list.
3. It tests each remaining cookie against the rules. A rule is a pure function: `(cookie, ctx) => Finding | null`. `ctx` holds the page protocol and the site origin.
4. It returns findings with fix text.

The session-name test uses a regular expression on the cookie name: `/(sess|sid|auth|token|jwt)/i`. Keep the tracking list in a JSON file so users can add names.

### 6.2 Technology and version fingerprinting

**Signals and where each one runs**

| Signal | Runs in | Method | Confidence |
| --- | --- | --- | --- |
| JS global | MAIN world | `chrome.scripting.executeScript({ world: "MAIN" })` runs a fixed helper. The helper walks `window` along a path array, for example `["jQuery","fn","jquery"]`. No string is evaluated | High |
| Script URL | Content script | Parse `<script src>` and match the URL regex | High |
| Script header comment | Content script for inline text, or service worker for an opt-in re-fetch | Read the first 500 bytes of the script body | High |
| `Server`, `X-Powered-By` | Service worker | `webRequest.onHeadersReceived`. Needs host access and a page reload | Medium |
| `<meta name="generator">` | Content script | DOM query | Medium |
| Cookie name | Service worker | Match against the cookie list | Low |

A content script runs in an isolated world and cannot read page globals. The MAIN world step is required. The helper reads `window` by property access only. It rejects any path segment that does not match `/^[A-Za-z_$][\w$]*$/`. `executeScript` returns the result to the service worker, so no message is needed. If the page blocks the helper, use URL and header signals and lower the confidence.

**Merge step.** The matcher groups signals by product. It picks the version from the strongest signal. If two strong signals disagree, it keeps both and sets confidence to medium.

### 6.3 CVE mapper

The license and data provenance of every bundled Retire.js artifact must be audited before release. If the audit fails, do not bundle the artifact; use OSV.dev as the fallback source with a visible cache-age status and retain the same privacy boundary.

1. Receive `{ product, version, packageName, ecosystem }` from the fingerprint step's reviewed mapping.
2. Look up the product in the bundled Retire.js data. Compare the version with each vulnerability range.
3. For each match, return one `Finding` with the CVE IDs and the severity (if the data gives one). If there are more than 10 matches, return one grouped finding.
4. If the product is not in the Retire.js data and has an OSV mapping, check the cache for `packageName@ecosystem@version`. If the entry is under 24 hours old, use it.
5. If there is no cache entry, send `POST https://api.osv.dev/v1/query` with `{ package: { name: packageName, ecosystem }, version }`. Read the CVE IDs from the `aliases` field of each result. Save the result in the cache.
6. Check the CVE IDs against the KEV set. Raise severity by one level for a KEV match, capped at `critical`.

If the OSV call fails, retry after 2 seconds, then 4 seconds, then 8 seconds, for a maximum of 3 retries. If all retries fail, use the last valid cache or bundled result; if none exists, return one info finding "CVE lookup failed" and suppress further retries for 1 hour. Never send the page URL, cookie data, script text, or any other page data to OSV.

### 6.4 JS secret scanner

**Pipeline**

1. The content script collects inline scripts and the list of external script URLs.
2. The service worker scans inline text directly. For external scripts, use an opt-in re-fetch with `cache: "force-cache"`; it is off by default, may reach the target, and may appear in its server log. Show this warning before enabling the option.
3. The service worker skips known-library URLs and files over the 5 MB size cap.
4. The service worker splits the text into 256 KB chunks with a small overlap, so a secret on a chunk edge is not lost. It yields between chunks and saves progress.
5. For each chunk, it runs the regex rules. Then it runs the entropy check near keywords.
6. For each match, it computes the line number, masks the value, and hashes it. Then it drops the script text.
7. It returns `Finding[]` with masked evidence only.

The MVP uses bounded chunks in the service worker. Keep each step short and yield between chunks. Add a Web Worker only if the 5 MB performance test proves the service worker cannot meet the documented budget.

**Regex safety.** Write rules without nested quantifiers. Time each rule on a 5 MB input before you ship it.

**Entropy check.** Compute Shannon entropy on the candidate string. Use it only when a keyword (`key`, `secret`, `token`, `password`) is within 40 characters before the value. Use the 4.5 bits/character threshold validated against the 50-sample file. Give entropy-only findings low confidence.

**Masking.** Show the first 4 and last 4 characters, with `****` between them. A secret shorter than 12 characters shows `****` only.

## 7. Messaging, caching, and performance

**Service worker rule.** Chrome stops an idle service worker. Do not keep state in memory. Write state to `chrome.storage.local` before each async step ends. Keep each step short and yield between chunks of work.

**Messages.** All messages use one typed envelope: `{ type, tabId, payload }`.

| Message type | From | To | Payload |
| --- | --- | --- | --- |
| `SCAN_REQUEST` | Popup | Service worker | `tabId`, permission result, requested origin |
| `PAGE_SNAPSHOT` | Content script | Service worker | Script URLs, inline scripts, meta tags |
| `SCRIPT_BODY` | Content script or service worker | Service worker | Inline script text or external script text after the configured opt-in re-fetch path; discarded after analysis and never persisted |
| `FINDINGS_UPDATED` | Service worker | Popup | Count and finding IDs |

The MAIN world helper needs no message. `chrome.scripting.executeScript` returns its result to the service worker. The permission request runs in the popup, because it needs the user click. The popup sends the result to the service worker; the worker owns the persisted scan state and reload interlock.

**Rescan triggers.** Rescan on a top-frame navigation. Also rescan when a content script sees a new `<script>` element (use a `MutationObserver`). Wait 500 ms after the last change, so a burst of changes causes one scan. After the user grants host access, reload the tab once to capture the headers.

**Caching**

| Data | Key | Lifetime | Note |
| --- | --- | --- | --- |
| Retire.js data | None | Bundled with the release | No network call |
| OSV result | `packageName@ecosystem@version` | 24 hours | Cache an empty result too |
| KEV list | One entry | 24 hours | Refresh in the background |
| Script text | None | Scan only | Do not store script text. Store masked findings only |

**Performance budget.** These are targets. Verify them with a performance trace on large sites.

| Item | Budget |
| --- | --- |
| Content script main-thread block per scan step | 50 ms or less |
| Time from click to first result in the popup (cookies and headers) | 2 seconds or less |
| Secret scan chunk size | 256 KB |
| Time to scan a 5 MB bundle in the service worker | 30 seconds or less, with 256 KB chunks and progress |
| Parallel script re-fetches (opt-in only) | 4 at a time |
| Memory use | Free the script text after each scan |

**Failure behavior.** A failed module must not stop the other modules. Each module returns its own result or its own error state to the popup.

## 8. Security and privacy design

A security tool must be safe itself. Users trust it with data from sensitive sites.

| Risk | Control |
| --- | --- |
| Page data leaves the device | No analysis network call except OSV (mapped package name/ecosystem and version) and the KEV list download. An opt-in script re-fetch sends the observed script URL to its target site, not to an analysis service, and may appear in the target site's logs. Enforce the analysis boundary with a strict extension CSP and fixed analysis-host permissions |
| The page attacks the extension | Treat all page data as untrusted. Render findings with `textContent`, never `innerHTML`. Escape evidence in exports |
| MAIN world code runs page-controlled input | Run a fixed list of read-only expressions. Never build code from page strings |
| Secrets leak through the extension | Mask in the UI. Store a hash and a masked value only. Remove script text from memory after the scan |
| Secrets leak through export | Export masked values only. Show a warning before the user adds evidence to a public report |
| Cache poisoning of OSV or KEV data | Fetch over HTTPS only. Check the response shape. Ignore fields that the extension does not use |
| Third-party library supply chain | Pin dependency versions. Run `npm audit` in CI. Keep the dependency list short |
| Extension update compromise | Use two-factor sign-in on the Chrome Web Store account. Sign release tags in GitHub |
| Legal risk of scanning | Default to manual scan. Keep active checks out of the MVP. Show a scope warning in the options page |

**Privacy policy points.** State what is read (cookies, headers, scripts), what is stored (findings, masked values), what leaves the device (mapped package name/ecosystem and version to OSV, plus the KEV feed request), the opt-in target-site re-fetch behavior, and how to delete data.

**Incognito mode.** The extension is off in incognito by default. Do not change this default.

## 9. Testing, build, CI, and release

**Test levels**

| Level | Tool | What it covers |
| --- | --- | --- |
| Unit | Vitest | Each analyzer as a pure function: cookie rules, fingerprint matcher, secret rules, masking, entropy, finding ID hash |
| Rule fixtures | Vitest | 50 secret samples (real-format and fake), cookie sets, tracking-cookie list, version strings |
| Integration | Vitest with mocked `chrome.*` APIs | Message flow, cache behavior, OSV client with a mock server |
| End to end | Playwright with the extension loaded | A full scan on local test pages and on OWASP Juice Shop |
| False-positive check | Manual, passive only | 5 popular public sites. Expect no high-severity finding |

**Test targets.** Use OWASP Juice Shop and DVWA that you run yourself. Use public sites for passive false-positive checks only. Do not test with active requests on a site you do not own or have permission to test.

**Write these tests first.** The secret scanner makes the most false positives. The CVE mapper makes the most false claims. Write the 50-sample secret test file and a version-to-CVE test table before you write those two modules.

**CI pipeline (GitHub Actions)**

1. Install with `npm ci`.
2. Run ESLint and the TypeScript compiler (`tsc --noEmit`).
3. Run Vitest.
4. Build the extension with WXT.
5. Run Playwright end-to-end tests against the build.
6. Run `npm audit` and fail on high-severity issues.
7. On a version tag, create a ZIP of the build as a release artifact.

**Release checklist**

- [ ] Permission list reviewed. Each permission has a one-line reason.
- [ ] Privacy policy published.
- [ ] Success metrics in the PRD pass.
- [ ] No full secret appears in storage, logs, or exports. Check with a test.
- [ ] Store listing, icons, and screenshots ready.
- [ ] Version tag signed. ZIP built from CI, not from a laptop.

## 10. Technical risks and open items

| Risk | Effect | Mitigation |
| --- | --- | --- |
| The service worker stops during a scan | Lost state, partial results | Write state to storage before each async step. Make each step safe to repeat |
| `webRequest` cannot read response bodies in MV3 | The extension cannot see script text from network traffic | Re-fetch scripts from the service worker with `force-cache` |
| A cache miss makes a new request to the target | The request can appear in the site's log. A script behind a login can return different content | Use `force-cache`. Document this in the options page. Skip a script if the fetch fails |
| Re-fetch of cross-origin scripts needs host access | Scan fails if the user did not grant the permission | Ask for the optional host permission when the user starts a scan. Show a clear message if it is missing |
| A version match does not prove a vulnerability | Wrong claim in a report | Use the "possible" label. Show the signal source |
| Missing OSV package mapping for a product | No CVE result | Keep a small, tested product map. Show "no mapping", not "no CVEs" |
| A large minified bundle slows the scan | Slow UI | Use a worker, chunks, and a size limit with a visible warning |
| Rule set license | Legal problem | Write original rules for the MVP; audit any reused community patterns before bundling |
| MAIN world access is blocked on some pages | Fewer high-confidence signals | Fall back to URL and header signals. Lower confidence |
| OSV thundering herd after 24h cache expiry | Rate limit errors for many users simultaneously | Stagger cache expiry (±1h jitter) or add a request queue post-1K users |

**Resolved decisions**

- [x] Use Svelte for the UI.
- [x] Use a 4.5 bits/character entropy threshold, validated against the 50-sample fixture.
- [x] Use license-audited Retire.js data first and OSV.dev as the fallback for JavaScript libraries.
- [x] Keep the tracking-cookie list in versioned project data and test it with cookie fixtures.
- [x] Limit each script scan to 5 MB and show a skipped-file warning.
- [x] Use the official CISA KEV feed URL and cache it for 24 hours.

**Release blockers**

- [ ] Complete the Retire.js bundled-data provenance and license audit.
- [ ] Validate the permission flow and header-capture reload on a real Chrome build.
- [ ] Run the documented performance, accessibility, privacy, and security gates.
