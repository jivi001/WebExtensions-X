# PRD: WebExtensions-X

Nov 15, 2026 · @Jivitesh

## 1. Overview

WebExtensions-X is a browser extension. It finds security problems in a web application while the user browses normally. It sends no attack traffic. It keeps page data on the user's device.

**Product name:** WebExtensions-X. **Release:** MVP (v0.1). **Platform:** Chrome and Chromium browsers (Manifest V3). Firefox comes later.

**Problem.** Pentesters and bug bounty hunters check cookies, libraries, versions, and JavaScript files by hand. This is slow. Existing tools show raw data but not risk. Each tool has its own output format, so notes get lost between tools.

**Value.** For each finding, the extension shows four things: what it is, why it matters, how to verify it, and how to fix it.

| Module | Pain point today | What the extension does |
| --- | --- | --- |
| Cookie analyzer | DevTools shows cookie flags but does not show which cookies matter | Checks flags on session-like cookies only. Gives the fix |
| Technology and version fingerprinting | Wappalyzer shows names, often with no version and no risk | Detects name and version. Shows a confidence level |
| CVE mapping | The user must search for CVEs by hand | Maps each detected version to known CVEs |
| JS secret scanner | Nobody reads 2 MB of minified JavaScript for keys | Scans scripts for keys and tokens. Masks each secret |

## 2. Goals, non-goals, and success metrics

**Goals**

1. Give a useful result on any page without setup.
2. Keep false positives low. A tool that shows noise loses trust.
3. Keep all analysis on the user's device.
4. Use one finding format and one export format for all modules.
5. Show a confidence level on every finding.

**Non-goals**

- Active exploitation, fuzzing, or any attack traffic.
- Phishing detection, ad blocking, or password management.
- A cloud backend or user accounts.
- AI-written explanations (after the core works).
- Mobile browsers.

**Success metrics.** These are targets set by this PRD. Change them if testing shows they are wrong.

| Metric | Target | How to measure |
| --- | --- | --- |
| Secret scanner precision (high-confidence rules) | 90% or more | 50-sample test file of real and fake secrets |
| Secret scanner noise on large public sites | 0 high-severity findings on 5 clean sites | Manual review of 5 sites |
| Version accuracy of fingerprinting | Correct version on 90% or more of pages where the version is public | 20 test pages with known versions |
| Cookie analyzer false flags on known tracking cookies | 0 | Test list of tracking cookies |
| Time to first result in the popup | 2 seconds or less | Timing test on Juice Shop |
| Main-thread block per scan step | 50 ms or less | Performance trace |
| Planted issues found on OWASP Juice Shop | All issues in the test plan | Test plan checklist |

## 3. Users and personas

The primary users are pentesters and bug bounty hunters. Design every choice for them first.

| Persona | Priority | Need | Main modules |
| --- | --- | --- | --- |
| Pentester | Primary | Passive recon before active tests. Evidence for the report | Fingerprinting, CVE mapping, secrets |
| Bug bounty hunter | Primary | Quick wins: leaked keys, old libraries, weak cookies | Secrets, CVE mapping, cookies |
| Developer | Secondary | Check own app in dev or staging before release | Cookies, secrets |
| DevSecOps engineer | Secondary | Baseline check of pre-production sites | All four |
| Security student | Secondary | Learn why a flag or finding matters | Cookies, fingerprinting |
| Security auditor | Secondary | Collect evidence and export it | Export |
| Everyday user | Later | A simple grade for a site | Not in MVP |

**Primary persona detail: the bug bounty hunter.** This person opens many targets each week. Time is the main cost. They know the issue types but do not want to check each by hand. They need results that they can trust and verify in seconds. A false claim in a report costs them reputation. For this reason, every finding must show its evidence and its confidence.

## 4. Use cases and scenarios

| ID | Module | Scenario | Expected result |
| --- | --- | --- | --- |
| UC-01 | Cookie | A pentester opens a login page. The session cookie has no `HttpOnly` flag | Finding at medium severity. The fix text explains `HttpOnly` |
| UC-02 | Cookie | A page sets `_ga` and `_fbp` without `HttpOnly` | No finding. These are known tracking cookies |
| UC-03 | Cookie | A cookie has `SameSite=None` and no `Secure` flag | Finding. Browsers reject this cookie |
| UC-04 | Cookie | A cookie value is a JWT | Info finding. The extension marks it for later JWT checks |
| UC-05 | Fingerprint | A page loads `jquery-1.12.4.min.js` | The extension lists jQuery 1.12.4 with high confidence |
| UC-06 | Fingerprint | A site hides its `Server` header but exposes `angular.version` | The extension reads the version from the page global |
| UC-07 | Fingerprint | Only a cookie name (`PHPSESSID`) shows the technology | The extension lists PHP with low confidence and no version |
| UC-08 | CVE | An old library version has known CVEs | The extension lists each CVE with severity. It marks a CVE that is in the CISA KEV list |
| UC-09 | CVE | OSV is offline or rate limited | The extension shows cached data or "lookup failed". It does not block the UI |
| UC-10 | Secret | A JavaScript bundle contains `AKIA` followed by 16 characters | High severity finding. The key is masked. Evidence shows file and line |
| UC-11 | Secret | A page contains a Stripe publishable key (`pk_live_`) | Info finding. This key is public by design |
| UC-12 | Secret | A scan of a large public site finds a test string in a library file | The extension skips known library files. No finding |
| UC-13 | Export | An auditor finishes a review and clicks Export | A Markdown report with all findings and evidence |
| UC-14 | Repeat visit | The user reloads the same page | Findings do not repeat. The store merges them by finding ID |

**Edge scenarios the design must handle**

- Single-page apps change scripts and cookies after load. The extension rescans on navigation and on new script loads.
- Pages in iframes from other origins. The MVP scans the top frame and same-origin frames only.
- Sites with a strict CSP. The extension must still read globals through the MAIN world.
- Pages with very large bundles (over 5 MB). The scanner works in chunks and shows progress.

## 5. Functional requirements

**Priority key.** Must = needed for the MVP. Should = in the MVP if time allows. Could = after the MVP.

### 5.1 Cookie analyzer

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-C01 | Read all cookies for the active site with `chrome.cookies`, including HttpOnly cookies | Must |
| FR-C02 | Flag a session-like cookie without `HttpOnly`. Session-like means the name matches `sess`, `sid`, `auth`, `token`, or `jwt` | Must |
| FR-C03 | Flag a cookie without `Secure` on an HTTPS site | Must |
| FR-C04 | Flag `SameSite=None` without `Secure` | Must |
| FR-C05 | Skip cookies on a known tracking list (`_ga`, `_fbp`, and similar) | Must |
| FR-C06 | Show a fix text on every cookie finding | Must |
| FR-C07 | Report a missing `SameSite` attribute as info | Should |
| FR-C08 | Flag an auth cookie with an expiry over one year | Should |
| FR-C09 | Flag a broad `Domain` attribute that shares the cookie with subdomains | Should |
| FR-C10 | Report a missing `__Host-` or `__Secure-` prefix on session cookies as info | Should |
| FR-C11 | Detect a JWT cookie value and mark it for later JWT checks | Could |

### 5.2 Technology and version fingerprinting

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-T01 | Read library globals in the page MAIN world (for example `jQuery.fn.jquery`) | Must |
| FR-T02 | Read versions from script URLs (for example `jquery-3.4.1.min.js`) | Must |
| FR-T03 | Read the first 500 bytes of a script for a version comment | Must |
| FR-T04 | Read the `Server` and `X-Powered-By` response headers | Must |
| FR-T05 | Ship 30 to 40 hand-written rules for libraries with known CVEs | Must |
| FR-T06 | Set confidence from the strongest signal. Show which signal gave the result | Must |
| FR-T07 | Show "version unknown" when no signal gives a version. Never guess | Must |
| FR-T08 | Read `<meta name="generator">` | Should |
| FR-T09 | Use cookie names as a low-confidence signal with no version | Should |
| FR-T10 | Keep rules in JSON files. A rule change needs no code change | Should |

### 5.3 CVE mapping

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-V01 | Map a detected product to an OSV package name with a lookup table | Must |
| FR-V02 | Query OSV.dev with the mapped package name, ecosystem, and version only | Must |
| FR-V03 | Never send the page URL or any other page data outside the device | Must |
| FR-V04 | Cache each result for 24 hours | Must |
| FR-V05 | Label results "Affected version (possible)". Do not write "Vulnerable" | Must |
| FR-V06 | Show cached data or "lookup failed" when OSV is offline or rate limited | Must |
| FR-V07 | Check each CVE against the CISA KEV list. Raise severity for a match | Should |
| FR-V08 | Use the Retire.js database as an offline source after the bundled-data license audit passes | Must |
| FR-V09 | Add NVD and CPE matching for server software | Could |

### 5.4 JS secret scanner

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-S01 | Scan inline scripts | Must |
| FR-S02 | Scan external scripts from browser-cached responses; if text is unavailable, offer an explicitly opt-in service-worker re-fetch using the browser cache | Must |
| FR-S03 | Keep script scanning off the page main thread by using bounded service-worker chunks; add a Web Worker only if the performance gate requires it | Must |
| FR-S04 | Ship starter rules: AWS key, GitHub token, Slack token, Stripe secret key, private key block, JWT, Google API key | Must |
| FR-S05 | Use the entropy check only near a keyword such as `key`, `secret`, or `token` | Must |
| FR-S06 | Mark keys that are public by design as info (Stripe `pk_live_`, Google Maps key, Firebase config) | Must |
| FR-S07 | Skip known library files | Must |
| FR-S08 | Mask each secret in the UI: first 4 and last 4 characters only | Must |
| FR-S09 | Store a hash and the masked value. Never store the full secret | Must |
| FR-S10 | Show script URL and line number as evidence | Must |
| FR-S11 | Read large files in chunks. Show progress | Should |
| FR-S12 | Detect public source map (`.map`) files | Could |

## 6. Common requirements

These requirements apply to all four modules.

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-G01 | Every module returns findings in one format with these fields: title, severity, confidence, evidence, location, why, fix | Must |
| FR-G02 | Give each finding a stable ID. The same issue on reload does not create a second finding | Must |
| FR-G03 | Show a confidence level (high, medium, low) on every finding | Must |
| FR-G04 | The popup shows counts by severity and a list grouped by module | Must |
| FR-G05 | A finding detail view shows four items: what it is, why it matters, how to verify, how to fix | Must |
| FR-G06 | Export all findings with evidence as Markdown and as JSON | Must |
| FR-G07 | Scan the active tab only after a user click (default mode) | Must |
| FR-G08 | Let the user turn each module on or off | Must |
| FR-G09 | Let the user clear all stored findings | Must |
| FR-G10 | Add a scope list. In continuous mode, scan only domains on the list | Should |
| FR-G11 | Filter the finding list by severity, module, and confidence | Should |
| FR-G12 | Tag findings with a CWE ID and an OWASP Top 10 category where one applies | Should |
| FR-G13 | Continuous scan mode for all sites the user allows | Could |
| FR-G14 | DevTools panel with the same finding list | Could |

**Confidence definitions**

| Level | Meaning | Example |
| --- | --- | --- |
| High | The evidence is direct. A person can verify it in one step | A library global returns `3.4.1` |
| Medium | The evidence is indirect, or two signals disagree | The `Server` header shows a version, but a proxy can change it |
| Low | The evidence is a weak hint | A cookie name suggests PHP |

**Severity rules.** Severity describes the impact if the finding is true. Confidence describes how sure the tool is that it is true. Show both. Do not merge them into one score.

**Export format.** The Markdown report has one section per finding with these parts: title, severity and confidence, location, evidence (masked), why it matters, how to verify, how to fix, and references. The JSON export uses the canonical `Finding` type from SPEC.md.

## 7. Non-functional requirements

| ID | Area | Requirement |
| --- | --- | --- |
| NFR-01 | Privacy | Analyze all page data on the device. The extension may query OSV.dev with a mapped package name/ecosystem and version only and may download the CISA KEV list without page data. Optional script re-fetches can create a request in the target site's server logs, so they require an explicit warning and opt-in. No page URL, cookie value, or script text is sent to an analysis service |
| NFR-02 | Privacy | Send no analytics or telemetry. State this in the privacy policy |
| NFR-03 | Privacy | Never store a full secret. Store a hash and a masked value |
| NFR-04 | Passive behavior | Send no attack traffic. Cached script reads do not create a request; an explicitly opt-in re-fetch may create a target-site request and must be disclosed |
| NFR-05 | Permissions | Use the fewest permissions that work. Request broad host access at run time, not at install |
| NFR-06 | Performance | A scan must not make the page visibly slower. See the performance budget in the TRD |
| NFR-07 | Reliability | A failure in one module must not stop the other modules |
| NFR-08 | Accuracy | Prefer a missed low-value finding over a false high-severity finding |
| NFR-09 | Compatibility | Chrome and Chromium browsers (Manifest V3) in the MVP. Firefox in a later release |
| NFR-10 | Usability | A user can run a scan and read the first result in two clicks |
| NFR-11 | Accessibility | The popup works with a keyboard. Severity uses text labels, not color only |
| NFR-12 | Maintainability | Rules are data files. A contributor can add a rule without changing code |
| NFR-13 | Security of the extension | The extension must not add a new risk to the pages it scans. It never writes to the page DOM and never runs strings from the page as code |
| NFR-14 | Licensing | Use the MIT license for the project, subject to dependency and bundled-data review. Check the license of each third-party rule set before use |

## 8. Milestones and release plan

Build in this order. Do not start a step until the exit check of the step before it passes. Effort values are rough estimates for one developer who works part time. Change them to match your real schedule.

1. **Project setup** (about 1 weekend). MV3, TypeScript, WXT, Svelte, canonical `Finding` type, test setup. *Exit check:* the popup opens and shows one test finding.
2. **Cookie analyzer** (about 1 weekend). *Exit check:* correct results on OWASP Juice Shop. No flags on tracking cookies.
3. **Fingerprinting** (about 2 weekends). Globals, URLs, headers, and 30 to 40 rules. *Exit check:* it detects jQuery and Angular versions on test pages.
4. **CVE mapper with cache** (about 1 weekend). *Exit check:* an old jQuery version shows its CVEs. A second scan uses the cache.
5. **JS secret scanner with masking** (about 2 weekends). *Exit check:* it finds all planted test keys. It shows no high-severity finding on 5 clean public sites.
6. **Confidence, dedupe, and export** (about 1 weekend). *Exit check:* a reload creates no duplicate. The Markdown report is complete.
7. **Hardening and release** (about 1 to 2 weekends). Privacy policy, store listing, permission review. *Exit check:* all success metrics in section 2 pass.

**First usable product.** Steps 3 and 4 together give a tool that is already useful to a pentester. Add no new feature until these two steps work well.

**Release plan**

- Alpha: GitHub release with an unpacked extension build. Share it with 3 to 5 testers.
- Beta: Chrome Web Store, unlisted. Collect false-positive reports.
- v1.0: Chrome Web Store, public. Open-source repository with the MIT license, subject to dependency and bundled-data license review.
- After v1.0: scope manager, JWT and CORS modules, DevTools panel, Firefox build.

## 9. Risks, legal points, and open questions

| Risk | Impact | Mitigation |
| --- | --- | --- |
| False positives from the secret scanner | Users lose trust | Use high-precision rules first. Show confidence. Skip library files. Test on 5 clean sites before each release |
| False claims from CVE mapping | Users report a wrong finding | Label "Affected version (possible)". Show the detection source. Explain that distributions can backport fixes |
| Version not visible on many sites | Fewer CVE results | Show "version unknown". Add more signals over time |
| Manifest V3 limits | Some old techniques do not work | Test early. Use the MAIN world helper, service worker, and explicitly opt-in cached script re-fetch only when needed |
| Broad host permission blocks store approval | Release delay | Use optional permissions. Give a clear reason in the listing |
| Wide scope growth | MVP does not ship | Build only the four MVP modules. Defer all others |
| Legal risk of scanning sites without permission | Harm to users | Default to manual scan. No active checks in the MVP. Show a scope warning |

**Legal points.** The extension is passive, but the re-read of a script can still appear in a site's server log. Tell users to use the tool only on sites they own or have permission to test, or sites in a bug bounty scope that allows this kind of testing. Add this notice to the options page and the store listing. This PRD is not legal advice. Ask a qualified person before public release.

**Release Decisions**

- Product name: **WebExtensions-X** (matches repo, recognizable)
- Chrome and Chromium browsers for MVP (Firefox deferred post-V1)
- Open source: Day one
- License: MIT, subject to dependency and bundled-data license review before release
- UI framework: Svelte (smaller bundle, better for extensions)
- Build tool: WXT (extension scaffolding, hot reload, type-safe chrome APIs)
- Entropy threshold: 4.5 bits/char, validated against the 50-sample fixture
- Script scan size limit: 5MB per script; larger files are skipped with a visible warning
- KEV list: https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json (official CISA feed, check daily, cache 24h)
- Fingerprinting rules: Original rules plus reviewed community patterns after provenance and license checks
