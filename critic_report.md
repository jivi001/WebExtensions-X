# Goldfish Review Report — Stage 7 Audit

**Target:** SPEC.md, PRD.md, TRD.md, DESIGN.md, agents.md (full document set)

**Audit Date:** 2026-11-16

**Audit Scope:** Fresh Stage 7 review after latest nit fixes

---

## Reviewer Status

| Reviewer | Status |
| --- | --- |
| `egm-style` | completed |
| `egm-codebase-details` | completed |
| `egm-structural` | completed |
| `egm-comprehension` | completed |
| `egm-implementation-readiness` | completed |
| `egm-design` | completed |

---

## Executive Summary

**All blockers:** 0  
**All major issues:** 0  
**All nits:** 4

All five project documents (SPEC.md, PRD.md, TRD.md, DESIGN.md, agents.md) are **production-ready**. The latest nit fixes successfully resolved all blocking and major issues from previous review stages. Four nit-level polish opportunities remain:

1. Minor redundancy in cross-reference explanations (style)
2. One verbose permission flow sentence (style)
3. Incomplete illustrative code example interface (technical)
4. Forward-looking OSV rate limit scalability note (design)

**None of these nits block MVP implementation or release.** All implementation contracts are complete, internally consistent, and actionable.

---

## Prioritized Findings (Nits Only)

### Redundant cross-reference explanation pattern — Severity: Nit
- **Reviewer Lens:** `egm-style`
- **Affected Scope:** SPEC.md, lines 6–11
- **Discrepancy Description:** The "Relationship to other documents" section repeats "referenced for" in every bullet, creating pattern redundancy. The relationship is already clear from the section heading.
- **Rationale & Recommendation:** Simplify to direct statements. Change:
  - "**PRD.md:** Product requirements and user stories (referenced for persona details, use cases)"
  
  To:
  - "**PRD.md:** Product requirements, personas, use cases"
  
  Apply the same pattern to TRD.md and DESIGN.md bullets.

---

### Verbose permission flow sentence — Severity: Nit
- **Reviewer Lens:** `egm-style`
- **Affected Scope:** SPEC.md, lines 137–140
- **Discrepancy Description:** Step 3 of the permission flow uses a 60-word sentence with nested subordinate clauses describing the listener registration, persistence, and reload sequence. While technically precise, it exceeds conversational sentence length and forces re-reading.
- **Rationale & Recommendation:** Break into two sentences:
  
  **Current:**
  > "If access is granted, the service worker writes `{ phase: "permission-granted", headerListener: "pending" }` to `scanState:<tabId>`, confirms the global `webRequest.onHeadersReceived` listener is registered, writes `{ headerListener: "active" }`, waits for the storage write to complete, and only then reloads the tab once to capture response headers."
  
  **Suggested:**
  > "If access is granted, the service worker writes `{ phase: "permission-granted", headerListener: "pending" }` to `scanState:<tabId>`. It confirms the global `webRequest.onHeadersReceived` listener is registered, writes `{ headerListener: "active" }`, waits for the storage write to complete, then reloads the tab once to capture response headers."

---

### Incomplete illustrative AnalyzerContext interface — Severity: Nit
- **Reviewer Lens:** `egm-codebase-details`
- **Affected Scope:** SPEC.md, lines 331–333
- **Discrepancy Description:** The illustrative `AnalyzerContext` interface shows only `origin` and `timestamp`, but TRD.md section 6.1 describes the cookie analyzer as receiving a `ctx` that "holds the page protocol and the site origin." The protocol field is missing from the example interface.
- **Rationale & Recommendation:** While the comment above the code states "The following is an illustrative contract excerpt," implementers may copy this interface. Add the missing `protocol` field or add an explicit note that the interface is incomplete. 

**Option A (complete):**
```typescript
interface AnalyzerContext {
  origin: string;
  protocol: 'http' | 'https';  // Required for Secure flag validation
  timestamp: number;
}
```

**Option B (explicit disclaimer):**
```typescript
// Simplified for illustration; production context includes protocol, tab metadata
interface AnalyzerContext {
  origin: string;
  timestamp: number;
}
```

---

### OSV.dev rate limit enforcement is reactive, not preventive — Severity: Nit
- **Reviewer Lens:** `egm-design`
- **Affected Scope:** SPEC.md lines 181–186, TRD.md lines 225–232
- **Discrepancy Description:** The documented retry strategy (2s, 4s, 8s backoff) responds to rate limit errors after they occur. At scale (10K users, 5 scans/day, 3 libraries/site = 150K queries/day), the cache-hit rate becomes critical, but there is no circuit breaker or request queuing to prevent a thundering herd after cache expiry.
- **Rationale & Recommendation:** While this is unlikely to affect the MVP (small user base, 24h cache), consider adding a note in TRD.md section 10 (Technical risks) or SPEC.md Infrastructure Economics that OSV.dev rate limit enforcement should be revisited post-MVP when active user count crosses 1K. 

Suggested addition to TRD.md risks table:

| Risk | Effect | Mitigation |
|------|--------|-----------|
| OSV thundering herd after 24h cache expiry | Rate limit errors for many users simultaneously | Stagger cache expiry (±1h jitter) or add a request queue post-1K users |

This is a nit because the documented cache and retry strategy is sufficient for MVP scale.

---

## Verification Summary

### Style (egm-style)
- ✓ No fluff, marketing language, or filler phrases
- ✓ Technical voice consistent across all documents
- ✓ Objective framing maintained
- ✓ Word choice precise and appropriate

### Technical Accuracy (egm-codebase-details)
- ✓ All data types and interfaces complete (except noted illustrative example)
- ✓ No contradictions between documents
- ✓ All cross-document references validated
- ✓ Finding structure, message protocol, storage schema all canonical

### Structure (egm-structural)
- ✓ Heading hierarchy logical and consistent
- ✓ Markdown formatting clean and valid
- ✓ Tables well-formed
- ✓ Cross-references complete
- ✓ No buried critical information

### Comprehension (egm-comprehension)
- ✓ All acronyms defined
- ✓ All technical terms explained
- ✓ No ambiguous instructions
- ✓ All assumptions stated
- ✓ All decision points documented

### Implementation Readiness (egm-implementation-readiness)
- ✓ All build/test/deployment steps complete
- ✓ All error handling specified
- ✓ All edge cases documented
- ✓ All performance budgets measurable
- ✓ All security boundaries enforceable

### Design & Architecture (egm-design)
- ✓ Architecture scales to documented targets
- ✓ Infrastructure costs documented
- ✓ All technical risks identified with mitigations
- ✓ Privacy boundaries enforceable
- ✓ Deployment readiness complete

---

## Separation: Document Defects vs. Runtime/Release Validation Tasks

### Current Document Findings (Above)
All findings above are documentation-level polish opportunities. None block implementation.

### Runtime/Release Validation Tasks (Not Document Defects)
The following are **not document issues**; they are validation tasks from the release checklist that must be completed before v0.1 release:

| Validation Task | Document Reference | Status |
|----------------|-------------------|--------|
| Complete Retire.js bundled-data provenance and license audit | TRD.md line 373, SPEC.md line 188 | **Not run** — Release blocker |
| Validate permission flow and header-capture reload on real Chrome build | TRD.md line 374 | **Not run** — Release blocker |
| Run documented performance gates (2s first result, 50ms main-thread, 30s 5MB scan) | TRD.md line 375, SPEC.md lines 19–22 | **Not run** — Release blocker |
| Run accessibility gates (keyboard, contrast, zoom, forced colors, dark mode, reduced motion) | agents.md line 38, DESIGN.md section 15 | **Not run** — Release blocker |
| Run security review (untrusted data, CSP, message validation, secret handling) | agents.md line 39 | **Not run** — Release blocker |
| Verify OSV.dev and Chrome Web Store provider limits and fees | SPEC.md lines 181–184 | **Not run** — Release blocker |
| Run false-positive check on 5 clean public sites | PRD.md line 45, SPEC.md lines 380–388 | **Not run** — Functional gate |
| Verify 50-sample secret scanner precision ≥90% | PRD.md line 44, SPEC.md line 417 | **Not run** — Functional gate |
| Verify version accuracy ≥90% on 20-page test set | PRD.md line 46, SPEC.md line 420 | **Not run** — Functional gate |
| Run all planted issues found on OWASP Juice Shop | PRD.md line 50 | **Not run** — Functional gate |

These validation tasks are correctly documented as release blockers in TRD.md section 10 and the release checklist. They are separate from document review and require a running implementation and test environment.

---

## Conclusion

**Document Quality:** Production-ready. All five documents are internally consistent, complete, and actionable for implementers.

**Nits:** 4 polish opportunities that do not block implementation or release.

**Next Steps:** 
1. (Optional) Address the 4 nits if additional polish is desired.
2. Begin implementation using SPEC.md as the authoritative contract.
3. Before v0.1 release, complete all runtime/release validation tasks listed above.

**Audit Status:** ✅ **COMPLETE** — All blockers: 0, All majors: 0, All nits: 4
