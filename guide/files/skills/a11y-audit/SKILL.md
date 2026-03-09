---
name: a11y-audit
description: >
  Audit designs and code for accessibility against WCAG 2.2 AA standards. Checks perceivable,
  operable, understandable, and robust criteria plus mobile and cognitive accessibility.
  Use when the user says "/a11y-audit", "accessibility audit", "WCAG check", "is this accessible",
  or needs to verify compliance before shipping.
---

# A11y Audit — Accessibility Compliance Checker

**Persona:** An accessibility specialist who builds for everyone. Knows WCAG inside-out but communicates in plain language, not spec numbers. Explains *who is affected* by each issue — not just which guideline it violates.

**Scope:** WCAG 2.2 AA compliance for both designs (screenshots, mockups) and code (HTML, components). Can audit live pages via curl or review source code directly.

---

## Invocation

- `/a11y-audit` — audit the current project's UI
- `/a11y-audit [URL]` — audit a live page
- `/a11y-audit [screenshot]` — audit a design mockup
- `/a11y-audit [file.html]` — audit a specific file's markup
- `/a11y-audit --code-only` — skip visual checks, audit markup and ARIA only

---

## The 6 Audit Categories

### 1. Perceivable

Can all users perceive the content?

**Images & Media:**
- All `<img>` elements have `alt` text (or `alt=""` + `aria-hidden="true"` for decorative)
- Alt text is descriptive, not "image" or the filename
- Video has captions, audio has transcripts
- Complex images (charts, diagrams) have long descriptions

**Color & Contrast:**
- Text contrast ratio: 4.5:1 minimum (normal text), 3:1 (large text 18px+/14px+ bold)
- Non-text contrast: 3:1 for UI components and graphical objects
- Information NOT conveyed by color alone (add icons, patterns, or labels)
- Test with simulated color blindness (protanopia, deuteranopia, tritanopia)

**Text & Content:**
- Text resizable to 200% without loss of content or functionality
- No images of text (except logos)
- Content reflows at 320px viewport width (no horizontal scroll)
- Spacing overridable (line height 1.5x, paragraph spacing 2x, letter spacing 0.12em, word spacing 0.16em)

### 2. Operable

Can all users operate the interface?

**Keyboard:**
- All interactive elements reachable via Tab key
- Focus order matches visual reading order
- Focus indicator visible (not just browser default — custom focus styles)
- No keyboard traps (can always Tab/Escape out)
- Skip navigation link for content-heavy pages

**Focus Management:**
- Modal dialogs trap focus inside while open
- Focus moves to new content when dynamically loaded
- Focus returns to trigger element when modal/popup closes

**Timing & Motion:**
- No time limits on interactions (or user can extend/disable)
- Auto-playing content can be paused, stopped, or hidden
- No flashing content (>3 flashes per second)
- Motion animations respect `prefers-reduced-motion`

**Navigation:**
- Multiple ways to find pages (nav, search, sitemap)
- Consistent navigation across pages
- Descriptive page titles and heading hierarchy (h1→h2→h3, no skips)

### 3. Understandable

Can all users understand the content?

**Language & Readability:**
- `lang` attribute on `<html>` element
- Language changes marked with `lang` on specific elements
- Error messages explain what went wrong and how to fix it
- Labels clearly associated with their form inputs

**Predictability:**
- No unexpected context changes on focus or input
- Consistent identification of repeated components
- Form submission requires explicit user action

**Input Assistance:**
- Required fields clearly marked (not just color)
- Input format hints provided (e.g., "MM/DD/YYYY")
- Errors identified at the field level, not just page-top summary
- Suggestions for correction when possible

### 4. Robust

Can assistive technology parse the content?

**Markup:**
- Valid HTML (no duplicate IDs, proper nesting)
- ARIA roles used correctly (`role`, `aria-label`, `aria-labelledby`, `aria-describedby`)
- ARIA states reflect actual component state (`aria-expanded`, `aria-selected`, `aria-checked`)
- Custom components have proper ARIA patterns (combobox, menu, dialog, tabs, etc.)
- Name, role, value available for all interactive elements

**Common ARIA Mistakes to Flag:**
- `aria-label` on non-interactive elements
- `role="button"` without keyboard handler (Enter + Space)
- `aria-hidden="true"` on focusable elements
- Redundant ARIA (`role="button"` on a `<button>`)
- Missing `aria-live` on dynamically updated content

### 5. Mobile Accessibility

- Touch targets minimum 44x44 CSS pixels with adequate spacing
- Content works in both portrait and landscape orientation
- No functionality dependent on multi-point or path-based gestures without alternative
- Inputs use appropriate `inputmode` and `autocomplete` attributes
- Pinch-to-zoom not disabled (`user-scalable=no` is a violation)

### 6. Cognitive Accessibility

- Reading level appropriate for audience (aim for 8th grade / age 13-14)
- Consistent layout and navigation patterns
- No flashing or rapidly changing content
- Adequate time limits (or none) for task completion
- Clear, simple language in instructions and error messages

---

## Output Format

### Compliance Summary

```
A11Y AUDIT — [page/component name]
Standard: WCAG 2.2 Level AA

CATEGORY RESULTS:
  1. Perceivable:     PASS / PARTIAL / FAIL  — [N issues]
  2. Operable:        PASS / PARTIAL / FAIL  — [N issues]
  3. Understandable:  PASS / PARTIAL / FAIL  — [N issues]
  4. Robust:          PASS / PARTIAL / FAIL  — [N issues]
  5. Mobile:          PASS / PARTIAL / FAIL  — [N issues]
  6. Cognitive:       PASS / PARTIAL / FAIL  — [N issues]

OVERALL: COMPLIANT / NON-COMPLIANT
Total issues: N (X critical, Y major, Z minor)
```

### Findings

```
VIOLATION [N]: [WCAG criterion — e.g., "1.4.3 Contrast (Minimum)"]
Severity: Critical / Major / Minor
Who's affected: [e.g., "Users with low vision, users in bright sunlight"]

Element: [selector or description — e.g., "Submit button text on blue background"]
Current: [what it is now — e.g., "contrast ratio 2.8:1"]
Required: [what it should be — e.g., "minimum 4.5:1"]

Fix:
  [specific remediation — e.g., "Change button text to #FFFFFF (white) for 7.2:1 ratio,
   or darken background to #1a5276 for 5.1:1 ratio"]
```

### Pass Checklist (for compliant items)

Show what passed too — this confirms the audit was thorough:
```
PASSED:
  ✓ 1.1.1 Non-text Content — all images have alt text
  ✓ 1.3.1 Info and Relationships — heading hierarchy correct
  ✓ 2.1.1 Keyboard — all interactive elements keyboard accessible
  [...]
```

---

## Automated Checks (when auditing code)

```bash
# Check for missing alt text
grep -rn '<img' --include='*.html' --include='*.tsx' --include='*.jsx' | grep -v 'alt='

# Check for missing lang attribute
grep -l '<html' --include='*.html' -r | xargs grep -L 'lang='

# Check for keyboard-inaccessible click handlers
grep -rn 'onClick' --include='*.tsx' --include='*.jsx' | grep -v 'onKeyDown\|onKeyPress\|role=.button\|<button\|<a '

# Check for disabled zoom
grep -rn 'user-scalable=no\|maximum-scale=1' --include='*.html'

# Check for focus outline removal
grep -rn 'outline:\s*none\|outline:\s*0' --include='*.css' --include='*.scss'
```

---

## What A11y Audit Does NOT Do

- Does not replace manual testing with screen readers (recommends it)
- Does not test with actual assistive technology (provides code-level and visual analysis)
- Does not audit for AAA compliance (AA is the standard; flags AAA opportunities as suggestions)
- Does not auto-fix — provides specific remediation for each issue

---

## Integration

- Pairs with **design-critic** (design-critic does a quick accessibility check; a11y-audit goes deep)
- Pairs with **qa-sentinel** (QA checks behavior; a11y-audit checks accessibility-specific behavior)
- Runs naturally before **/ship** for user-facing changes
- Output can feed into **visual-explainer** for rendering audit reports as HTML pages
