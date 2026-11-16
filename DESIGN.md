# Design System: WebExtensions-X

**Design reference:** Apple Human Interface Guidelines (HIG), adapted for a Manifest V3 browser extension popup, options page, and planned DevTools panel.

**Source:** [Apple Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines)

**Captured:** 2026-11-15 via Playwright MCP.

> This is a project design contract derived from Apple's guidance. Apple platform-specific APIs, point sizes, and materials are translated into web equivalents where necessary. Apple HIG guidance remains authoritative for native Apple surfaces; this file defines how to apply the principles to this browser extension.
>
> **Scope note:** The MVP ships the popup and options page. DevTools panel guidance is retained for the deferred surface described in the PRD and TRD; it does not expand MVP scope.

---

## 1. Product identity and atmosphere

WebExtensions-X is a focused security instrument for pentesters, bug bounty hunters, developers, and auditors. It should feel:

- **Calm and trustworthy:** no alarmist decoration, fake certainty, or unexplained scores.
- **Dense but scannable:** evidence-rich information with strong hierarchy, predictable alignment, and progressive disclosure.
- **Quietly technical:** security terminology is precise, but explanations remain plain and direct.
- **Native to the browser:** the extension should feel like a familiar browser tool, not a marketing dashboard.
- **Respectful of attention:** get the user to the first useful finding quickly; keep controls unobtrusive until needed.

The visual language is restrained, semantic, and evidence-first. Color, motion, and ornament support comprehension; they never compete with findings or imply severity without textual explanation.

### Primary design promise

Every finding must make four things immediately understandable:

1. **What it is**
2. **Why it matters**
3. **How to verify it**
4. **How to fix it**

Severity and confidence are separate concepts and must never be merged into one visual score.

---

## 2. Apple-inspired project principles

The principles below synthesize Apple HIG themes for this project; they are not presented as a verbatim Apple taxonomy.

### Purpose — make something meaningful

- Put the active scan and the most important result at the center of the experience.
- Prioritize findings that help the person make a decision or verify a report.
- Do not add a feature, chart, badge, or animation unless it helps the user understand or act.
- Keep the main experience focused on passive evidence: cookies, technologies, CVEs, and secrets.
- Keep JWT, CORS, DOM-sink, active-exploitation, fuzzing, cloud-backend, account, and AI-written explanation features out of the MVP; see `SPEC.md` and `PRD.md` for deferred scope.
- Do not recreate existing browser security tools without a clear improvement in evidence, confidence, or remediation.

### Agency — let people do things their own way

- Scan only after a clear user action in the default mode.
- If continuous mode is implemented, make navigation and new-script rescans available only after the person explicitly enables it and restrict them to the configured scope; continuous mode is deferred from the MVP unless the scope is promoted.
- Let users enable or disable each analyzer.
- Let users filter by severity, module, and confidence.
- Let users clear stored findings.
- Let users export Markdown and JSON.
- Let users move between summary and detail without being trapped in a wizard.
- Provide an escape or back path for every secondary view.
- Make destructive or irreversible actions explicit and recoverable where possible.
- Never hide the evidence behind an opaque score.

### Responsibility — act in people’s best interest

- Explain what the extension reads and why before requesting access.
- Keep page URLs, cookie values, and script contents on-device.
- Use bundled Retire.js data first for JavaScript-library lookups; use OSV.dev only as the documented fallback and send only the mapped package name/ecosystem and version.
- The CISA KEV list download carries no page data. An opt-in external-script re-fetch is a request to the target site and may appear in its logs; it is not an analysis upload. Cache OSV results and the KEV list for 24 hours, and distinguish bundled data, cached data, no mapping, and lookup failure in the UI.
- Never display or export a full secret; show only masked evidence.
- Use clear confidence language such as **High**, **Medium**, and **Low**.
- Label CVE matches as **Affected version (possible)** rather than claiming vulnerability as fact.
- Tell the user when a lookup fails, data is cached, or evidence is incomplete.
- Never use fear-based copy, fabricated urgency, or a red color to imply certainty.

### Familiarity — build on what people know

- Use standard browser conventions for buttons, tabs, checkboxes, links, focus, scrolling, and keyboard navigation.
- Keep the same control appearance and behavior wherever a control type repeats.
- Use familiar terms: **Scan**, **Findings**, **Details**, **Export**, **Settings**, **Clear**.
- Use standard severity and status patterns, but always pair color with text and/or iconography.
- Prefer native semantic HTML controls over custom controls.

### Flexibility — adapt to contexts and needs

- Support the MVP’s narrow popup and options page; treat the DevTools layout as post-MVP guidance until that surface is promoted in the product scope.
- Design for keyboard, screen readers, zoom, forced colors, dark mode, and reduced motion in the MVP. Treat touch, RTL, broad localization, and other platform adaptations as quality targets unless they receive explicit acceptance criteria.
- Preserve the user’s place when the layout changes or a finding list updates.
- Keep primary content and controls in predictable positions across viewport sizes.
- Support long URLs, long technology names, and large text without clipping; treat translated text and RTL behavior as quality targets until supported locales are defined.
- Use responsive layout decisions based on available space, not on assumed device names.

### Simplicity — be clear and direct

- Every visible element must earn its place.
- Prefer one clear primary action per view.
- Use progressive disclosure for evidence, references, and implementation details.
- Establish hierarchy through size, weight, spacing, grouping, and semantic color—not through decoration.
- Use concise labels and active verbs.

### Craft — care about every detail

- Prototype early and test on real pages, not only fixtures.
- Check loading, empty, partial, error, permission-denied, offline, large-bundle, and long-text states.
- Release with no known critical or serious accessibility issues, backed by automated checks plus manual keyboard, screen-reader, zoom, forced-colors, and reduced-motion testing.
- Refine wording, alignment, focus states, and motion as deliberately as primary functionality.
- Treat every scan result as a trust-building moment.

### Delight — make it human without adding noise

- Use small, purposeful transitions for scan progress, completion, filtering, and navigation.
- Make successful completion feel clear, not celebratory or distracting.
- Give errors actionable next steps.
- Never use animation, confetti, sound, or visual drama to decorate a security finding.

---

## 3. Layout system

### 3.1 Information hierarchy

Use this order in the popup and primary views:

1. **Context:** current site/origin and scan state.
2. **Primary action:** Scan or Rescan.
3. **Summary:** counts by severity and modules scanned.
4. **Findings:** grouped, filterable list.
5. **Detail:** evidence, confidence, why, verification, fix, references.
6. **Secondary actions:** Export, settings, clear data.

Place the most important content near the top and leading edge in the reading direction. Use logical DOM order so the visual and keyboard order agree.

### 3.2 Grouping and hierarchy methods

Use the following methods intentionally:

- **Alignment:** align titles, values, controls, and finding metadata to shared edges.
- **Indentation:** use indentation to show subordinate evidence or nested metadata.
- **Negative space:** use spacing to separate groups before adding borders.
- **Containers:** use a container only when it clarifies a functional or semantic group.
- **Separators:** use low-contrast separators for repeated rows, not as the primary hierarchy mechanism.
- **Progressive disclosure:** collapse evidence, references, and technical details until requested.
- **Scrollable sections:** use scrolling for long finding lists and evidence; do not make the entire popup an undifferentiated scroll region if a fixed action area is required.
- **Consistent row structure:** keep finding rows structurally similar so users can scan quickly.

### 3.3 Responsive behavior

The extension is web-based, so use CSS layout primitives rather than native Apple layout APIs:

- Use CSS Grid for summary cards and two-dimensional metadata.
- Use Flexbox for toolbars, rows, and inline control groups.
- Use `minmax()`, wrapping, and intrinsic sizing instead of fixed device widths.
- Use `min-width: 0` on flexible children so long evidence cannot force overflow.
- Use `overflow-wrap: anywhere` for URLs, hashes, tokens, and script locations.
- Allow rows to grow vertically when text size or localization requires it.
- Stack horizontally adjacent content when the available width or text size makes it crowded.
- Never make the main action disappear solely because the viewport is narrow; move it into a predictable overflow only when necessary.
- Preserve the same function when layout changes; change presentation and visible density, not meaning.

### 3.4 Safe areas and boundaries

For browser surfaces, translate safe-area guidance into content safety boundaries:

- Keep content clear of browser-provided or extension-provided headers, toolbars, and sticky action areas.
- Do not place essential controls at the bottom edge where they can be obscured by a clipped popup or viewport.
- Keep fixed or sticky controls visually distinct from scrollable content.
- Do not allow evidence, buttons, or focus rings to be clipped by the popup boundary.
- Keep critical information away from corners when browser overlays or platform chrome may cover them.

### 3.5 Density and spacing

Use a compact, consistent spacing scale rather than arbitrary gaps:

- **4 px:** icon/text optical adjustment or very tight metadata.
- **8 px:** related label/value pairs and compact control gaps.
- **12 px:** standard row and control-group separation.
- **16 px:** section padding and primary group separation.
- **24 px:** major section separation and unboxed glyph hit-region padding.
- **32 px:** rare, reserved for major view transitions or spacious empty states.

These are web translation tokens, not Apple platform measurements. The interaction-size boundaries below are normative.

### 3.6 Layout anti-patterns

- Do not center every element by default; center only when it supports the content hierarchy.
- Do not create a dense wall of cards.
- Do not use a dashboard grid when a simple list better supports evidence review.
- Do not place important controls at the bottom of a potentially clipped surface.
- Do not hide essential actions behind hover-only behavior.
- Do not rely on fixed heights for finding rows or text containers.
- Do not change functionality when the layout crosses a responsive breakpoint.

---

## 4. Responsive layout methods

### Popup layout

Use a narrow single-column layout:

```text
[Origin / scan state]
[Primary Scan action]
[Severity summary]
[Filters]
[Finding list]
[Export / Settings]
```

Rules:

- Keep the scan action visible near the top.
- Keep severity counts textual and scannable.
- Use one finding per row with a clear title, module, severity, and confidence.
- Open details in a full-width secondary popup state or the options page when the popup becomes cramped. The DevTools two-pane layout below is post-MVP and must not be required for the MVP flow.
- Do not place four large metric cards above the first useful finding.

### Options page layout

Use a wider reading layout with a persistent navigation rail or top navigation only when multiple settings categories exist:

```text
[Page title + context]
[Settings navigation] [Current settings section]
                         [Controls]
                         [Explanations]
                         [Save / reset actions]
```

Rules:

- Group settings by task: scan scope, enabled modules, privacy, storage, export.
- Explain permission or privacy impact next to the relevant control.
- Avoid deeply nested settings.
- Keep reset and clear actions visually separated from ordinary configuration.

### DevTools panel layout — post-MVP

The DevTools panel is deferred in the MVP. If it is later promoted, use a two-pane layout at sufficient width:

```text
[Toolbar: origin | filters | export]
[Finding list] | [Selected finding detail]
```

At narrow widths, collapse to list → detail navigation. Preserve selection when filters or scan updates occur unless the selected finding no longer exists.

### Finding detail layout

```text
[Finding title] [Severity] [Confidence]
[Short explanation]
[Evidence block]
[How to verify]
[How to fix]
[References / metadata]
```

Evidence should be visible without requiring the user to interpret a chart. Technical detail can be disclosed progressively but must remain available.

---

## 5. Typography

### Typeface

- Prefer the platform/system sans-serif stack for a browser extension:
  `-apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif`.
- Use a single primary typeface family. Do not mix typefaces unless a clear semantic reason exists.
- Use a monospaced system stack only for code, hashes, URLs, cookie names, and evidence where character alignment helps verification.
- Use font weight to establish hierarchy: Regular for body, Medium/Semibold for labels and headings, Bold only for strong emphasis.
- Avoid Ultralight, Thin, and Light weights for UI text.

### Size and line-height translation

Apple’s native platform defaults are point-based. For this web UI, preserve the intent rather than copying platform point sizes directly:

| Role | Web target | Guidance |
| --- | ---: | --- |
| Page/view title | 20–24 px | Strong hierarchy; wrap if needed. |
| Section heading | 15–17 px | Semibold; concise. |
| Body/default | 14–16 px | Comfortable reading; avoid text below 12 px unless a tested, non-critical use case requires it. |
| Metadata/caption | 12–13 px | Use only for secondary information; maintain contrast. |
| Code/evidence | 12–14 px | Monospace; allow wrapping or horizontal scroll. |

### Typography rules

- Maintain hierarchy when text is enlarged.
- Support browser zoom and user text-size preferences without clipping or overlap. Test popup and options-page reflow in the MVP; include the DevTools surface only when it is implemented.
- Prefer `rem`/`em` sizing and flexible line heights.
- Use line height that supports reading; never use tight leading for three or more lines of text.
- Keep text truncation to a minimum.
- Do not truncate evidence, URLs, or remediation text without a separate way to view the full content.
- Let labels and finding titles wrap to multiple lines when needed.
- Stack metadata below titles when horizontal space becomes constrained.
- Keep primary content near the top of a view even at large text sizes.
- Use sentence case for most UI labels unless an established product convention requires otherwise.
- Use active verbs: **Scan**, **Export report**, **Clear findings**, **Open settings**.

### Security-specific writing

- Say **Affected version (possible)**, not **Vulnerable**, when the evidence is version mapping rather than exploit confirmation.
- Say **Version unknown** rather than guessing.
- Explain confidence independently from severity.
- Use length- and secret-type-aware redaction. Default to the secret type or a short fingerprint; only show first/last characters when the rule confirms that this cannot expose or reconstruct sensitive data.
- Explain what the user can verify and what they can fix.
- Do not use unexplained abbreviations in the first appearance of a concept.

---

## 6. Accessibility contract

Accessibility is a launch requirement, not a polish task. The interface must be intuitive, perceivable, and adaptable.

### Text and contrast

- Support at least 200% text enlargement where the platform/browser allows it, and test reflow at approximately 320 CSS px and 400% zoom where applicable. Test popup, options, and DevTools surfaces independently.
- Minimum WCAG contrast targets:
  - Normal text: **4.5:1**.
  - Large text: **3:1** (at least 24 px regular or 18.67 px bold in CSS terms).
  - Meaningful non-text UI components, graphics, and focus indicators: **3:1** against adjacent colors.
- Bold text is not automatically large text; apply the large-text threshold only when its size also qualifies.
- Test light, dark, increased-contrast, forced-colors, and reduced-transparency contexts. Where supported, honor `prefers-reduced-transparency`; always provide an opaque fallback.
- Prefer semantic/system colors and CSS custom properties with light/dark variants.
- Never communicate severity, state, or interactivity through color alone.
- Pair severity colors with visible labels and, when useful, distinct icons/shapes.

### Controls and hit regions

Apple’s platform guidance lists these native-platform default/minimum control sizes for reference only; Apple points are not CSS pixels and do not define web accessibility requirements:

| Platform | Default | Minimum |
| --- | ---: | ---: |
| iOS/iPadOS | 44×44 pt | 28×28 pt |
| macOS | 28×28 pt | 20×20 pt |
| tvOS | 66×66 pt | 56×56 pt |
| visionOS | 60×60 pt | 28×28 pt |
| watchOS | 44×44 pt | 28×28 pt |

For this browser extension, the following are project choices:

- Prefer **44×44 CSS px** targets for primary touch controls.
- Keep essential targets at least **24×24 CSS px** where WCAG 2.2 target-size guidance applies, with adequate spacing and applicable exceptions; use **28×28 CSS px** as this project’s chosen minimum for essential controls.
- Add padding around controls as needed to reach the target size; approximately **12 px** is a starting point for controls with a visible bezel.
- For unboxed glyphs or icons, provide a sufficiently large hit region; approximately **24 px** of surrounding space is a starting point, not a fixed visual requirement.
- Increase the hit region before increasing the visible icon size.
- Keep adjacent controls sufficiently separated to prevent accidental activation.
- Make the complete custom bar-button hit region contiguous; do not leave gaps inside one visual button.

### Keyboard and assistive technology

- All interactive controls must be reachable and operable with a keyboard.
- Focus order must follow the visual and reading order.
- Focus indicators must be visible in light, dark, high-contrast, and forced-colors modes.
- Never remove the browser’s focus outline without providing a stronger equivalent.
- Use semantic elements (`button`, `a`, `input`, `select`, headings, lists) before ARIA.
- Give every icon-only control an accessible name.
- Announce scan progress, completion, lookup failure, and result-count changes using an appropriate live region.
- Ensure finding severity, confidence, and module are available to screen readers as text.
- Support keyboard-only navigation, screen readers, browser zoom, and high-contrast modes.
- Provide non-gesture alternatives for every core action.

### Motion and sensory alternatives

- Never use animation as the only way to communicate status or change.
- Pair important visual states with text and/or iconography.
- Pair audio-like feedback with visual feedback; do not require sound.
- Respect `prefers-reduced-motion: reduce` by removing nonessential transitions and continuous motion.
- Keep transitions short and precise; make gesture-driven or reversible transitions interruptible.
- Avoid flashing, oscillating, or peripheral motion.

---

## 7. Color system

No fixed Apple palette is copied into the project. Use semantic roles so colors can adapt across appearances and accessibility settings.

### Semantic roles

| Role | Meaning | Requirements |
| --- | --- | --- |
| `--color-bg` | Primary surface | Adapts in light/dark mode. |
| `--color-surface` | Elevated/grouped surface | Separates related content without overpowering it. |
| `--color-text-primary` | Main content | Highest text contrast. |
| `--color-text-secondary` | Supporting content | Must remain readable; never use for critical content alone. |
| `--color-border` | Separation | Low contrast but visible in high-contrast modes. |
| `--color-accent` | Interactive/brand accent | Use consistently for links and primary actions. |
| `--color-info` | Informational state | Always paired with text/icon. |
| `--color-warning` | Caution | Always paired with text/icon. |
| `--color-danger` | High-risk state | Never implies exploit confirmation by itself. |
| `--color-success` | Completed/verified | Always paired with text/icon. |

### Color rules

- Do not use the same color to mean different things.
- Use semantic colors by purpose, not by hard-coded appearance.
- Provide light, dark, and increased-contrast variants for custom colors.
- Test colors over content, backgrounds, transparency, and image-like evidence.
- Avoid similar control and background colors when evidence is visually rich.
- Use color sparingly on controls; reserve strong accent treatment for the primary action or a truly important state.
- Do not rely on red/green alone for severity or pass/fail.
- Consider cultural meanings when adding new color semantics or localization.
- Do not hard-code system color values as though they are stable across releases.

### Dark mode

- Support light, dark, and automatic appearance where the browser supports it.
- Do not offer a competing app-specific appearance setting unless a real product need exists.
- Test with increased contrast and reduced transparency.
- Use `prefers-color-scheme`, semantic tokens, and CSS media queries.
- Ensure icons, code blocks, evidence surfaces, borders, and focus states remain legible in both modes.

---

## 8. Materials, surfaces, and depth

The browser extension may use a restrained web adaptation inspired by Apple’s material hierarchy, while prioritizing solid surfaces for dense evidence, tables, and code:

- **Content layer:** findings, evidence, tables, code, and explanatory text. Use solid or lightly tinted surfaces.
- **Functional layer:** toolbar, filters, navigation, and primary actions. Use a distinct surface, subtle border, or controlled translucency.
- **Transient layer:** popovers, menus, alerts, and dialogs. Use an elevated opaque or semi-opaque surface with strong contrast.

Rules:

- Do not use Liquid Glass-like effects throughout the content layer.
- Use translucency only when it clearly separates controls from content and preserves legibility.
- Use a regular/opaque surface when text is dense or the background is visually complex.
- Use a clear/translucent treatment only over visually rich backgrounds and only with sufficient contrast.
- Do not rely on a fixed opacity to guarantee legibility. Test translucent surfaces against representative worst-case backgrounds and provide an opaque fallback.
- Treat blur and translucency as progressive enhancement. The opaque or semi-opaque fallback must preserve hierarchy, contrast, and separation on its own, including in forced-colors mode.
- Prefer semantic material choice over choosing a material because of the color it appears to create.
- Use shadows sparingly: whisper-soft elevation for popovers, no heavy decorative shadows on ordinary rows.
- Use borders and surface contrast before adding a shadow.

---

## 9. Controls and interaction patterns

### Buttons

- Use a descriptive verb: **Scan**, **Export**, **Clear**, **Open details**.
- Make the primary action visually distinct from secondary actions.
- Keep button hierarchy stable across views.
- Show disabled, hover, pressed, focus, loading, and success states.
- Do not make a destructive action look like the primary action.
- Keep destructive actions explicit and, where appropriate, confirmable.

### Toggles and settings

- Use toggles for immediate on/off settings.
- Use a checkbox when selection is part of a larger form or can be submitted together.
- Explain the effect of a setting next to its label.
- Do not hide important consequences in a tooltip.
- Reflect the actual state immediately after interaction.

### Finding rows

Each row should expose:

- Finding title.
- Module.
- Severity label.
- Confidence label.
- Short evidence/location summary.
- A clear affordance to open details.

Fingerprint findings must also show the detection source when available: global, script URL, header, header comment, meta tag, or cookie. If no version is known, say **Version unknown** rather than implying a match. CVE details must distinguish **No mapping**, **No known CVEs**, **Cached result**, and **Lookup failed**.

Do not put the entire meaning of a finding only in a color stripe or icon.

### Filters

- Prefer simple, visible filters for severity, module, and confidence.
- Preserve selection and scroll position when filters change when possible.
- Show the number of matching findings.
- Provide a clear way to remove filters.
- Do not make filtering the only way to discover the complete result set.

### Empty, loading, and error states

- Empty state: explain why it is empty and provide the next useful action.
- Loading state: show what is being scanned and whether partial results are available.
- Error state: say what failed, whether other modules completed, and what the user can do next.
- Permission state: explain why access is needed and what functionality is reduced if access is denied.
- Offline/lookup state: distinguish bundled data, cached data, unavailable lookup, no mapping, and no matching data.
- Keep a failed analyzer visible as an analyzer status; do not present its failure as a clean security result.
- For scripts larger than the supported cap, show that the file was skipped and why. For chunked scans, show progress and partial findings.
- Do not put crucial information only in an empty state because the state may disappear.

---

## 10. Gesture, pointer, and input rules

The MVP requires keyboard and pointer access. Touch, gesture, and hover guidance below is a quality target for compatible browser surfaces, not a reason to expand MVP scope.

Although this project is primarily a pointer and keyboard extension, support touch-capable trackpads and touchscreens where available.

- Provide more than one way to perform important actions.
- Keep custom gestures supplementary; never make them the only route to core functionality.
- Prefer familiar gestures and browser conventions.
- Make custom interactions discoverable through visible affordances and feedback.
- Do not conflict with browser edge gestures, scrolling, back navigation, or system UI.
- Make tap/click targets larger through hit regions, not by distorting the visual control.
- Provide keyboard alternatives for click, hover, drag, and gesture actions.
- Do not hide critical controls behind hover alone.
- Use pointer hover for preview or emphasis, never for essential information.
- Preserve selection when revealing a hidden control.
- Make context-menu actions available through visible controls when they are essential.
- Keep focus and hover effects from obscuring adjacent content.
- Avoid assigning a primary action to a list or scroll surface in a way that conflicts with row selection or scrolling.

---

## 11. Writing and content rules

### General

- Put the most important information first.
- Use fewer words when possible.
- Write in plain language and avoid jargon or define it on first use.
- Use direct, active voice.
- Address the person as **you** and **your**; avoid referring to them as “the user.”
- Avoid unnecessary humor, idioms, slang, and culture-specific expressions.
- Use sentence case consistently for labels and headings.
- Keep copy concise on narrow surfaces.
- Read important copy aloud during review.

### Buttons and links

- Use verbs for buttons and descriptive phrases for links.
- Do not use **Click here**.
- Use **Continue**, **Next**, **Get started**, and **Done** only when they accurately describe the step.
- A button should tell the person what happens next.

### Errors and warnings

- Place the error close to the problem.
- Explain how to fix it.
- Do not blame the person.
- Avoid “Oops!”, “Uh-oh!”, and robotic messages such as “Invalid input.”
- Prefer: **Enter a site or origin before scanning.**
- Distinguish an analyzer failure from a security finding.

### Privacy and permissions

- State what access is requested and why.
- Explain what data stays on-device.
- Explain the narrow external calls the extension makes.
- Never make permission copy sound broader than the actual behavior.

### Inclusive language

- Use clear, inclusive, gender-neutral language.
- Avoid stereotypes in examples and imagery.
- For the MVP, keep copy ready for localization: allow text expansion, variable word order, and locale-specific dates/numbers. Full translated-label and RTL validation is post-MVP until supported locales are defined.
- Avoid idioms and culturally specific metaphors.
- Use icons as support, not as an untranslatable replacement for text.

---

## 12. Internationalization and right-to-left layout

This section is a quality target and post-MVP readiness guidance until supported locales and acceptance tests are defined.

- Use logical CSS properties such as `margin-inline`, `padding-inline`, `inset-inline`, `border-start-start-radius`, and `text-align: start`.
- Do not encode left/right meaning into icon names or layout assumptions unless the icon is directional and mirrored intentionally.
- As a minimum test heuristic, exercise strings expanded 30–50% beyond English, then test representative long translations and every supported locale.
- Test right-to-left languages with the complete finding flow, filters, tables, and code/evidence blocks.
- Keep URLs, hashes, code, and technical identifiers readable without accidental bidi reordering.
- Use locale-aware formatting for dates and counts.
- Do not let translated text break controls or change the action’s meaning.

---

## 13. Motion system

Motion must be functional, brief, and interruptible.

Use motion for:

- Scan progress and completion.
- Expanding/collapsing details.
- Navigating between list and detail states.
- Updating a result count or filter state.
- Showing focus, hover, pressed, or selected state.

Avoid motion for:

- Decoration.
- Repeated scanning or flashing.
- Conveying severity without text.
- Forcing people to wait before interacting.
- Large peripheral or continuous movement.

Rules:

- Use the shortest duration that preserves comprehension; **120–240 ms** is an initial tuning range for ordinary state changes, not a universal rule.
- Let users interrupt or cancel gesture-driven or reversible transitions; simple state changes should remain short and non-blocking.
- Use opacity/position changes rather than large-scale movement when no spatial meaning exists.
- Respect `prefers-reduced-motion: reduce`.
- Disable continuous progress animation when reduced motion is requested; use a static status and textual progress instead.
- Do not use a looping indeterminate animation as the only scan-progress signal; pair it with a textual status and, when possible, completed/total work.
- Never use an animation as the sole confirmation of a completed scan.
- Keep the frame rate smooth and avoid layout-thrashing animations.

---

## 14. Security and privacy boundaries reflected in the UI

These are product boundaries, not optional styling:

- No active exploitation, fuzzing, or attack traffic.
- Full navigation URLs, cookie values, and script text do not leave the device for analysis services. An opt-in external-script re-fetch may send the observed script URL to its target site and may appear in that site's logs.
- Retire.js is the primary offline source for JavaScript-library CVE mapping. OSV.dev is the fallback and receives only the mapped package name/ecosystem and version. The CISA KEV refresh carries no page data.
- Cache OSV results and the KEV list for 24 hours; show whether CVE data came from bundled data, cache, a live lookup, or a failed lookup.
- Only the mapped package name/ecosystem and version may be sent to OSV.dev for the MVP fallback lookup; full navigation URLs, cookie values, and script text remain local.
- Never store a full secret; store only a hash and masked value.
- Never execute strings from page content as code.
- Never expose raw or reconstructable secrets in the UI, Markdown export, JSON export, logs, screenshots, or accessibility labels; accessible names should identify the secret type or state without repeating masked token characters.
- Never imply exploitability from version matching alone.
- A failure in one analyzer must not block results from other analyzers.
- Permission-denied and incomplete-data states must be visible and understandable.
- After Scan is clicked, explain and request access for the current origin. If granted, persist the scan state, confirm the global non-blocking header listener is active, then reload once to capture response headers on a best-effort basis; if denied, continue with reduced coverage and identify which modules have reduced coverage. Cross-origin frames and external script bodies must be described as unavailable when permission prevents access. If external script text is unavailable, offer its re-fetch only as an explicit opt-in with a target-site logging warning.
- Keep the extension disabled in incognito by default.
- The UI must not add risk to the pages being analyzed.

---

## 15. Review checklist

### Hierarchy and layout

- [ ] The primary action is obvious and available without hunting.
- [ ] The first useful finding appears without excessive decoration or scrolling.
- [ ] Related items are grouped by alignment, spacing, or container.
- [ ] Long findings, URLs, evidence, and translations do not overflow.
- [ ] Layout changes with available space but preserves functionality and meaning.
- [ ] No important control is clipped by popup or viewport boundaries.

### Accessibility

- [ ] Keyboard-only flow works from scan through export.
- [ ] Focus order matches reading order.
- [ ] Focus indicators are visible.
- [ ] Contrast meets WCAG targets: 4.5:1 for normal text, 3:1 for large text, and 3:1 for meaningful non-text UI components/graphics.
- [ ] Severity and confidence are not communicated by color alone.
- [ ] Essential controls meet the project’s 28×28 CSS px minimum and applicable WCAG target-size guidance; primary touch controls target 44×44 CSS px.
- [ ] Text survives zoom and large-text testing.
- [ ] Screen-reader names and live announcements are present.
- [ ] Reduced motion, forced colors, dark mode, and increased contrast are tested.

### Content and trust

- [ ] Labels use clear verbs.
- [ ] Errors explain recovery.
- [ ] Confidence and severity are separate.
- [ ] Evidence is masked and verifiable.
- [ ] Privacy and permission rationale is clear.
- [ ] Version matches are not presented as confirmed vulnerabilities.
- [ ] Empty, loading, partial, offline, skipped-large-file, analyzer-failure, and permission-denied states are complete.
- [ ] Permission flow explains current-origin access, header-capture reload, reduced coverage after denial, and incognito behavior.
- [ ] Fingerprint detection source and version-unknown states are visible.
- [ ] CVE states distinguish bundled, cached, live, no mapping, no known CVEs, and lookup failure.

### Performance and resilience

- [ ] First cookie/header result appears within the **2-second** target after Scan is clicked.
- [ ] No content-script scan step blocks the page main thread for more than **50 ms**.
- [ ] Secret scanning uses **256 KB** chunks with progress and partial-result feedback.
- [ ] Scripts over the configured size cap show a visible skipped-file warning; the cap and 5 MB behavior are documented before release.
- [ ] Each analyzer can fail independently without hiding results from the others.

### Motion and polish

- [ ] Motion has a purpose.
- [ ] Motion is interruptible and reduced for `prefers-reduced-motion`.
- [ ] No flashing or continuous decorative animation exists.
- [ ] Hover is never the only way to access information.
- [ ] The extension has no avoidable console errors or accessibility violations.

---

## 16. Source pages

- [Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines)
- [Design principles](https://developer.apple.com/design/human-interface-guidelines/design-principles)
- [Layout](https://developer.apple.com/design/human-interface-guidelines/layout)
- [Typography](https://developer.apple.com/design/human-interface-guidelines/typography)
- [Color](https://developer.apple.com/design/human-interface-guidelines/color)
- [Materials](https://developer.apple.com/design/human-interface-guidelines/materials)
- [Accessibility](https://developer.apple.com/design/human-interface-guidelines/accessibility)
- [Gestures](https://developer.apple.com/design/human-interface-guidelines/gestures)
- [Pointing devices](https://developer.apple.com/design/human-interface-guidelines/pointing-devices)
- [Writing](https://developer.apple.com/design/human-interface-guidelines/writing)
- [Motion](https://developer.apple.com/design/human-interface-guidelines/motion)
- [Dark Mode](https://developer.apple.com/design/human-interface-guidelines/dark-mode)
- [Inclusion](https://developer.apple.com/design/human-interface-guidelines/inclusion)
- [Right to left](https://developer.apple.com/design/human-interface-guidelines/right-to-left)
- [Spatial layout](https://developer.apple.com/design/human-interface-guidelines/spatial-layout)
- [App icons](https://developer.apple.com/design/human-interface-guidelines/app-icons)
