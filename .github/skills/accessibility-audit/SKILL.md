---
name: accessibility-audit
description: Audit and remediate web accessibility with Playwright MCP and axe-core. Use for accessibility reviews, WCAG fixes, keyboard or focus defects, and browser-based verification of frontend changes.
---

# Accessibility audit and remediation

Evaluate the rendered application in a real browser. Static source inspection may
support the investigation, but it never replaces Playwright MCP and axe-core.

## Before auditing

1. Read repository instructions and identify the application entry point, start
   command, target URL, and existing accessibility tests.
2. Start or reuse the smallest suitable local server. Wait until its health check
   succeeds before opening the browser.
3. Record the URL, viewport, color scheme, authentication state, and user flow.
   Keep these conditions identical for the baseline and final scan.
4. Use Playwright MCP for navigation, interaction, accessibility snapshots, and
   JavaScript execution. Do not substitute curl, static HTML parsing, Lighthouse,
   or a unit-test DOM for the required browser scan.

## Load axe-core

Prefer an existing repository dependency:

```text
node -p "require.resolve('axe-core/axe.min.js')"
```

If it is unavailable, obtain a pinned stable `axe-core` package in a temporary
directory without changing the repository's dependency manifests or lockfiles.
Use Playwright MCP to inject that local `axe.min.js` into the rendered page, for
example with `page.addScriptTag({ path: axePath })` through the MCP tool that can
run Playwright code. Avoid a CDN when the browser is restricted to localhost.

After injection, assert that `typeof globalThis.axe === "object"`. If axe cannot
be loaded, report the exact error and stop; never claim that the page passed.

## Capture a baseline

Wait for the page and target flow to settle, then run the default rules:

```javascript
async () => {
  const result = await axe.run(document);
  return {
    testEngine: result.testEngine,
    testEnvironment: result.testEnvironment,
    testRunner: result.testRunner,
    timestamp: result.timestamp,
    url: result.url,
    violations: result.violations.map((violation) => ({
      id: violation.id,
      impact: violation.impact,
      description: violation.description,
      help: violation.help,
      helpUrl: violation.helpUrl,
      tags: violation.tags,
      nodes: violation.nodes.map((node) => ({
        impact: node.impact,
        target: node.target,
        html: node.html,
        failureSummary: node.failureSummary
      }))
    })),
    incomplete: result.incomplete
  };
}
```

Retain every violation node and its target. Record incomplete results separately;
they require manual review and must not be reported as confirmed violations.

## Remediation loop

Repeat these steps until the exit criteria are met:

1. Sort confirmed violations by impact: `critical`, `serious`, `moderate`, then
   `minor`. Group related nodes by root cause.
2. Reproduce the highest-priority group in Playwright. Inspect the accessible
   name, role, value, state, focus order, and keyboard behavior as applicable.
3. Find the owning source and make the smallest coherent fix. Prefer native HTML
   semantics, programmatic labels, logical structure, visible focus, and valid
   state announcements over ARIA patches.
4. Reload or restart as required, restore the same state and viewport, inject
   axe-core again, and rerun the default `axe.run(document)` scan.
5. Compare `(rule id, target)` fingerprints with the previous pass. Verify that
   the intended nodes disappeared and that no new violations were introduced.
6. Exercise each changed interaction with keyboard-only input. At minimum check
   focus visibility and order; for dialogs check initial focus, trapping, Escape,
   and focus return; for dynamic updates check an appropriate live announcement.

Stop when one of these conditions is true:

- The default axe scan returns no violations and manual checks pass.
- Every remaining issue is verified as outside repository control or an
  explicitly preserved negative-test fixture.
- Two consecutive passes produce the same fingerprints after attempted fixes.
- Five remediation passes have completed.

For the last three conditions, report the residual findings and blocker. Never
loop indefinitely or describe an unresolved result as a pass.

## Axe guidance

- Run the default rule set first and last. A focused `runOnly` scan may diagnose
  a fix, but it is not evidence that the whole page passes.
- Inspect every node. Fixing one selector does not prove the rule is resolved.
- Do not suppress rules, exclude selectors, remove content, or add
  `aria-hidden` merely to reduce the violation count.
- Automated results cover only part of WCAG. Never claim full WCAG conformance
  from axe-core alone.
- Test meaningful states, not only initial load: open menus and dialogs, submit
  forms, trigger errors, and scan route changes relevant to the requested flow.

## Reporting

Report:

- URL, viewport, state, axe-core version, and number of remediation passes.
- Baseline and final violation counts by impact.
- Each corrected rule id, impact, affected target, and applied source fix.
- New violations introduced during a pass, even if subsequently fixed.
- Manual keyboard, focus, dialog, and announcement checks performed.
- Remaining violations and incomplete results with their exact reason.
