---
name: wave-contrast-audit
description: Audit a live webpage for WCAG color contrast failures using axe-core injected via Playwright, then map every failing color pair to its exact CSS rule and apply compliant fixes. Use when a WAVE report shows contrast errors or when asked to fix accessibility contrast issues on a deployed page.
---

# WAVE Contrast Audit

Systematically find and fix every color contrast violation on a live page using axe-core for precise data, then trace failures back to CSS source rules.

## When to Use This Skill

- A WAVE report shows contrast errors on a deployed page
- User shares a WAVE report URL (`wave.webaim.org/report#/...`)
- Asked to "fix contrast issues" or "make the site WCAG AA compliant"
- After a design system token change that may have introduced failures

## Core Workflow

### Step 1 — Navigate to the live page (not the WAVE report)

Always run axe-core on the **actual page**, not the WAVE report wrapper.

```
Navigate to: https://example.com/page.html
```

### Step 2 — Inject and run axe-core for exact color pairs

This gives you precise foreground/background hex values, contrast ratios, CSS selectors, and font sizes — far more actionable than clicking WAVE icons one by one.

```javascript
async () => {
  const script = document.createElement('script');
  script.src = 'https://cdnjs.cloudflare.com/ajax/libs/axe-core/4.9.1/axe.min.js';
  document.head.appendChild(script);
  await new Promise(r => script.onload = r);

  const results = await axe.run({ runOnly: ['color-contrast'] });

  // Deduplicate by unique fg+bg pair
  const pairs = {};
  for (const v of results.violations) {
    for (const node of v.nodes) {
      for (const rel of node.any) {
        if (rel.data?.fgColor) {
          const key = `${rel.data.fgColor}__on__${rel.data.bgColor}`;
          if (!pairs[key]) {
            pairs[key] = {
              fg: rel.data.fgColor,
              bg: rel.data.bgColor,
              ratio: rel.data.contrastRatio,
              fontSize: rel.data.fontSize,
              fontWeight: rel.data.fontWeight,
              required: rel.data.expectedContrastRatio,
              count: 0,
              selector: node.target[0]
            };
          }
          pairs[key].count++;
        }
      }
    }
  }
  return {
    total: results.violations.reduce((a, v) => a + v.nodes.length, 0),
    pairs: Object.values(pairs).sort((a, b) => a.ratio - b.ratio)
  };
}
```

### Step 3 — Interpret the results

For each unique pair, note:
- **fg + bg** — the actual rendered hex values (rgba already resolved to hex)
- **ratio** — current contrast (e.g. 3.1:1)
- **required** — 4.5 (normal text) or 3.0 (large text >= 18px / >= 14px bold)
- **selector** — CSS selector pointing to the failing element
- **count** — how many elements share this exact pair

### Step 4 — Find the CSS source rules

Use `Grep` to locate every CSS rule producing the failing color:

```bash
# Search for the failing value (rgba or hex)
grep -n "rgba(255,255,255,.45)\|#7a7878\|var(--steel)" src/styles.css

# Or search by selector
grep -n "\.sb-num\|\.btn-water" src/styles.css
```

Read the surrounding context to confirm background color and element role.

### Step 5 — Calculate compliant replacement values

**For white text on dark backgrounds (`rgba(255,255,255, A)`):**

| Required | Min alpha on `#022b45` | Min alpha on `#034f7d` | Min alpha on `#7a4f99` |
|----------|------------------------|------------------------|------------------------|
| 4.5:1    | >= 0.75 (use **0.82**) | >= 0.75 (use **0.82**) | >= 0.85 (use **0.88**) |
| 3.0:1    | >= 0.50 (use **0.60**) | >= 0.52 (use **0.60**) | >= 0.60 (use **0.68**) |

**For brand blue on light backgrounds:**
- `#1499e8` on white = 3.1:1 (fail) — use `#0d77c4` for button bg (4.72:1)
- `#1499e8` as text on white — use `#0d77c4`

**For brand blue as text on dark backgrounds:**
- `#1499e8` on `#022b45` = 2.8:1 (fail) — use `#8accf4` (~5.0:1)

**For muted gray on white/cream:**
- `#7a7878` on white = 4.4:1 (fail) — darken to `#686666` (5.7:1)
- `#7a7878` on `#ededed` = 3.7:1 (fail) — `#686666` also passes cream (4.9:1)

**Manual luminance calculation (when needed):**
```
Contrast = (L_lighter + 0.05) / (L_darker + 0.05)
L = 0.2126 * R_lin + 0.7152 * G_lin + 0.0722 * B_lin
R_lin = ((R/255 + 0.055) / 1.055)^2.4   for values > 0.04045
R_lin = (R/255) / 12.92                  for values <= 0.04045
```

### Step 6 — Apply fixes efficiently

Batch all changes in a single Python pass to avoid making 50+ Edit calls:

```python
with open('styles.css', 'r') as f:
    css = f.read()

changes = [
    ('--steel: #7a7878',         '--steel: #686666'),
    ('.btn-water { background: var(--vibrant-blue)',
     '.btn-water { background: #0d77c4'),
    # Add targeted string replacements for each failing rule...
]

results = []
for old, new in changes:
    if old in css:
        css = css.replace(old, new)
        results.append(f"  changed: {old[:50]}")
    else:
        results.append(f"  NOT FOUND: {old[:50]}")

with open('styles.css', 'w') as f:
    f.write(css)

print('\n'.join(results))
```

> **Warning:** `rgba(255,255,255,.X)` appears in backgrounds, borders, and
> box-shadows — not only text color. Always read context before replacing.
> Use targeted string matches (include surrounding lines) rather than broad
> single-value replacements.

### Step 7 — Verify fixes

Re-run the Step 2 script on the updated page. Target: **0 color-contrast violations**.

If the file is only local (not yet deployed), navigate Playwright directly:
```
Navigate to: file:///path/to/page.html
```

---

## Common Failure Patterns

| Pattern | Failing value | Fix |
|---------|--------------|-----|
| Sidebar nav links on dark bg | `rgba(255,255,255,.5-.62)` | Raise to `.82` |
| Section labels on dark bg | `rgba(255,255,255,.18-.25)` | Raise to `.75` |
| Body text on dark bg | `rgba(255,255,255,.52-.62)` | Raise to `.85` |
| Small labels/captions on dark bg | `rgba(255,255,255,.38-.45)` | Raise to `.82` |
| Any text on purple bg (#7a4f99) | alpha < .85 | Raise to `.88` |
| CTA button (white on #1499e8) | 3.1:1 | Change bg to `#0d77c4` |
| Muted gray on white | `#7a7878` | Darken to `#686666` |
| Brand blue as text on white | `#1499e8` | Change to `#0d77c4` |
| Brand blue as text on dark | `#1499e8` | Change to `#8accf4` |
| Brand palette 25% tint on color bg | e.g. `#ded3e6` on purple | Use `var(--white)` |

## WCAG 2.2 AA Thresholds (Quick Reference)

| Text size | Required ratio |
|-----------|---------------|
| Normal (< 18px regular, < 14px bold) | **4.5 : 1** |
| Large (>= 18px regular, >= 14px bold) | **3.0 : 1** |
| UI components and icons | **3.0 : 1** |

## Limitations

- axe-core **skips** elements where background is a CSS gradient, `background-image`,
  or `filter` — use WAVE's eyedropper tool for those manually
- WAVE counts each text node individually; axe deduplicates by color pair — expect
  axe to report fewer total violations than WAVE (e.g. 53 axe vs 187 WAVE)
- Hover and focus states are not evaluated by either tool — test manually
- Decorative elements (watermarks, icon chevrons, spacers) should receive
  `aria-hidden="true"` rather than contrast fixes

## Resources

- [WAVE Web Accessibility Evaluator](https://wave.webaim.org/)
- [axe-core CDN](https://cdnjs.cloudflare.com/ajax/libs/axe-core/4.9.1/axe.min.js)
- [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/)
- [WCAG 2.2 — 1.4.3 Contrast Minimum](https://www.w3.org/WAI/WCAG22/Understanding/contrast-minimum)
