# wave-contrast-audit

A Claude Code skill that audits a live webpage for WCAG 2.2 color contrast failures using axe-core injected via Playwright, maps every failing color pair to its exact CSS rule, and applies compliant fixes.

## Install

```bash
npx skills add kaipengyu/WAVE-contrast-audit-skill
```

## When to use

- A WAVE report shows contrast errors on a deployed page
- You share a WAVE report URL (`wave.webaim.org/report#/...`)
- Asked to "fix contrast issues" or "make the site WCAG AA compliant"
- After a design system token change that may have introduced failures

## How it works

1. **Navigate** to the live page (not the WAVE report wrapper)
2. **Inject axe-core** via Playwright to get precise foreground/background hex values, contrast ratios, CSS selectors, and font sizes
3. **Deduplicate** violations by unique color pair to surface root causes
4. **Grep the CSS source** to locate the exact rule producing each failure
5. **Calculate compliant replacements** using WCAG luminance math
6. **Batch-apply fixes** in a single pass to avoid 50+ individual edits
7. **Verify** by re-running axe-core — target is 0 color-contrast violations

## WCAG 2.2 AA thresholds

| Text size | Required ratio |
|-----------|---------------|
| Normal (< 18px regular, < 14px bold) | 4.5 : 1 |
| Large (>= 18px regular, >= 14px bold) | 3.0 : 1 |
| UI components and icons | 3.0 : 1 |

## Common failure patterns

| Pattern | Failing value | Fix |
|---------|--------------|-----|
| Sidebar nav links on dark bg | `rgba(255,255,255,.5-.62)` | Raise alpha to `.82` |
| Section labels on dark bg | `rgba(255,255,255,.18-.25)` | Raise alpha to `.75` |
| Body text on dark bg | `rgba(255,255,255,.52-.62)` | Raise alpha to `.85` |
| CTA button (white on brand blue) | 3.1:1 | Darken bg to `#0d77c4` |
| Muted gray on white | `#7a7878` | Darken to `#686666` |
| Brand blue as text on white | `#1499e8` | Change to `#0d77c4` |
| Brand blue as text on dark bg | `#1499e8` | Change to `#8accf4` |

## Limitations

- axe-core skips elements where the background is a CSS gradient or `background-image` — use WAVE's eyedropper for those manually
- Hover and focus states are not evaluated — test manually
- Decorative elements should receive `aria-hidden="true"` rather than contrast fixes

## Resources

- [WAVE Web Accessibility Evaluator](https://wave.webaim.org/)
- [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/)
- [WCAG 2.2 — 1.4.3 Contrast Minimum](https://www.w3.org/WAI/WCAG22/Understanding/contrast-minimum)
