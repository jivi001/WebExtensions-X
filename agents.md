# AGENTS.md — WebExtensions-X Working Rules

## Document authority

- `SPEC.md` is the authoritative implementation contract for WebExtensions-X.
- `PRD.md` defines product goals, users, and MVP requirements.
- `TRD.md` provides deeper technical detail and must not contradict `SPEC.md`.
- `DESIGN.md` defines the popup and options-page interaction and accessibility contract.
- This file defines the agent workflow and quality gates; it does not select product frameworks.

When documents drift, update the non-authoritative document or record the unresolved conflict. Do not silently choose a conflicting implementation.

## Product guardrails

- Keep the MVP passive: send no attack traffic and perform analysis on-device.
- Send only the documented mapped package name/ecosystem and version to OSV.dev. The KEV feed carries no page data.
- Treat opt-in script re-fetch as a request to the target site: warn before enabling it and state that it may appear in the target site's server logs.
- Never upload page URLs, cookie values, script text, secrets, or findings to managed tools.
- Never store, log, export, or commit a full secret. Test fixtures use fake values only; CI validates fixture metadata and scans fixture files for credential patterns without calling live providers.
- Test active security behavior only against local, owned, or explicitly authorized targets. Public-site checks remain passive false-positive checks.

## Workflow

1. Read the applicable contract and identify every affected caller, test, fixture, configuration file, export, and document.
2. Plan small, independently verifiable slices before implementation.
3. Implement the smallest complete change, preserving existing conventions.
4. Add or update tests before declaring behavior complete; use red-green-refactor for new logic.
5. Run unit, integration, browser, accessibility, performance, privacy, and applicable security checks.
6. Review the diff and all affected documents for drift, stale claims, and accidental data exposure.
7. Record release blockers. A human approves implementation and release gates.

## Required quality gates

- TypeScript strict checking and lint pass.
- Unit tests cover pure analyzers, masking, entropy, deduplication, and cache behavior.
- Integration tests cover message validation, permission denial, service-worker restart/resume, OSV privacy, and malformed external data.
- Chromium browser tests cover permission flow, listener/reload ordering, CSP fallback, degraded modules, export masking, and the documented performance budgets.
- Accessibility checks cover keyboard operation, focus visibility, semantic labels, contrast, forced colors, dark mode, zoom, and reduced motion.
- Security review covers untrusted page data, extension CSP, message validation, secret handling, dependency risk, and relevant OWASP categories without active testing of unauthorized targets.
- Release checks include the Retire.js provenance/license audit, dependency audit, privacy-policy review, and Chrome Web Store permission review.

## UI requirements

Use Svelte and `DESIGN.md`. The UI must explain permissions before requesting them, show scan and degraded states, keep severity and confidence separate, expose masked evidence, and remain keyboard-operable and usable with zoom, forced colors, dark mode, and reduced motion.

## Tool availability

Use repository-approved browser, test, and security tools when available. If an external tool or browser runtime is unavailable, record that limitation and use the safest local verification available; an unavailable tool does not justify uploading protected page data or inventing a passing result.

## Documentation maintenance

Keep commands, paths, permissions, interfaces, privacy boundaries, and release decisions synchronized across the project documents. Prefer one authoritative definition and cross-reference it elsewhere. Remove stale provisional language when a decision is resolved. Do not mark a validation task complete until it has actually run.
