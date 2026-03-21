---
description: "Post-build UI/UX/CX review — crawl the newly built site, score every aspect, and refine autonomously"
---

# /wp-refine — Post-Build UI/UX/CX Review & Refinement

Use this after a build is complete (or after `/wp-resume` reaches Phase 5+).
Claude will crawl the live dev site, audit every page against the PRD, score each dimension,
then fix every issue found — all in one autonomous pass.

---

## Step 1 — Identify the project

If the user specifies a client name, use it.
Otherwise read `SESSION_STATE.json` from the most recently active client:

```bash
ls ~/clients/
cat ~/clients/<slug>/SESSION_STATE.json
```

Confirm:
- WordPress port (from `SESSION_STATE.json → ports.wp`)
- Theme slug and theme directory path
- PRD exists at `~/clients/<slug>/PRD.md`
- Docker is running — if not, start it:
  ```bash
  cd ~/clients/<slug> && docker compose up -d
  ```

---

## Step 2 — Crawl the built site

Since WebFetch does not support localhost URLs, use curl + Python to fetch and parse each page:

```bash
curl -s http://localhost:<port><path> | python3 -c "
import sys, re
html = sys.stdin.read()
def clean(t): return re.sub(r'<[^>]+>', '', t).strip()
print('TITLE:', clean(re.findall(r'<title>(.*?)</title>', html, re.S)[0]) if re.findall(r'<title>(.*?)</title>', html, re.S) else 'N/A')
print('H1:', [clean(h) for h in re.findall(r'<h1[^>]*>(.*?)</h1>', html, re.S)])
print('H2:', [clean(h) for h in re.findall(r'<h2[^>]*>(.*?)</h2>', html, re.S)][:8])
print('H3:', [clean(h) for h in re.findall(r'<h3[^>]*>(.*?)</h3>', html, re.S)][:10])
btns = [clean(b) for b in re.findall(r'<a[^>]*class=\"[^\"]*btn[^\"]*\"[^>]*>(.*?)</a>', html, re.S|re.I)]
print('CTAs:', btns[:10])
checks = ['Lorem ipsum', 'placeholder', 'TODO', 'dummy']
print('PLACEHOLDERS:', [w for w in checks if w.lower() in html.lower()])
"
```

Fetch all pages listed in `SESSION_STATE.json → sitemap`. For each page record:
- Headings hierarchy (H1–H3)
- Visible copy per section
- CTA text and placement
- Navigation state
- Footer completeness
- Any visible errors (broken layouts, missing images, empty sections)

---

## Step 3 — Colour System Audit

This is a mandatory step. Run it against the theme's CSS variables file before scoring.

### 3a — Extract palette from `_variables.css`

Read `~/clients/<slug>/wp-content/themes/<theme-slug>/assets/css/_variables.css` and identify all `--color-*` tokens.

### 3b — Run contrast checker

```python
python3 << 'EOF'
def hex_to_rgb(h):
    h = h.lstrip('#')
    return tuple(int(h[i:i+2], 16) for i in (0, 2, 4))

def luminance(r, g, b):
    c = [x/255 for x in (r,g,b)]
    c = [x/12.92 if x <= 0.03928 else ((x+0.055)/1.055)**2.4 for x in c]
    return 0.2126*c[0] + 0.7152*c[1] + 0.0722*c[2]

def contrast(h1, h2):
    l1 = luminance(*hex_to_rgb(h1))
    l2 = luminance(*hex_to_rgb(h2))
    lighter, darker = max(l1,l2), min(l1,l2)
    return (lighter + 0.05) / (darker + 0.05)

# UPDATE these from _variables.css before running
palette = {
    'bg':        '#REPLACE',
    'surface':   '#REPLACE',
    'accent':    '#REPLACE',
    'accent-2':  '#REPLACE',
    'text':      '#REPLACE',
    'muted':     '#REPLACE',
    'subtle':    '#REPLACE',
    'cta':       '#REPLACE',
    'cta-text':  '#REPLACE',
}

WCAG_AA = 4.5
WCAG_AA_LARGE = 3.0

checks = [
    ('text',     'bg',      'Body text on background'),
    ('text',     'surface', 'Body text on cards'),
    ('muted',    'bg',      'Secondary text on background'),
    ('muted',    'surface', 'Secondary text on cards'),
    ('subtle',   'bg',      'Subtle text on background'),
    ('subtle',   'surface', 'Subtle text on cards'),
    ('accent',   'bg',      'Accent on background'),
    ('accent',   'surface', 'Accent on cards'),
    ('cta-text', 'cta',     'CTA button text on button bg'),
    ('accent',   'accent-2','Accent on accent-2 (hover pair)'),
]

print(f"{'Pair':<40} {'Ratio':>6}  {'AA Normal':>10}  {'AA Large':>9}")
print("-" * 70)
failures = []
for a, b, label in checks:
    if palette[a] == '#REPLACE' or palette[b] == '#REPLACE':
        print(f"{label:<40}  [palette not set — update hex values above]")
        continue
    r = contrast(palette[a], palette[b])
    aa  = "PASS" if r >= WCAG_AA       else "FAIL"
    aal = "PASS" if r >= WCAG_AA_LARGE else "FAIL"
    flag = " ⚠" if aa == "FAIL" else ""
    print(f"{label:<40} {r:>6.2f}  {aa:>10}  {aal:>9}{flag}")
    if aa == "FAIL":
        failures.append((label, a, b, r))

print()
if failures:
    print("FAILURES requiring fixes:")
    for label, a, b, r in failures:
        print(f"  - {label}: {r:.2f}:1 (need 4.5:1)")
else:
    print("All pairs pass WCAG AA.")
EOF
```

### 3c — Audit accent overuse

```bash
grep -h "var(--color-accent" ~/clients/<slug>/wp-content/themes/<theme>/assets/css/*.css \
  | grep -oP 'var\(--color-[^)]+\)' | sort | uniq -c | sort -rn
```

**Thresholds:**
- `--color-accent` used >30 times → overused; flag for reduction
- `--color-accent` used >50 times → severely overused; fix in Step 4

### 3d — Palette hue variety check

Read all hex values from `--color-*` tokens. Convert to HSL. If >80% of tokens share a hue within ±30°, flag as **monotone palette** — note in scorecard, flag for client decision (cannot auto-fix without brand direction).

### 3e — Semantic colour check

- Is `--color-success` a distinctly different hue from `--color-accent`? If not, flag — success states will be indistinguishable from normal accent elements.
- Is `--color-error` clearly red/warm? If not, flag.
- Are hover states (`--color-cta-hover`, `--color-accent-2`) meaningfully different from base? Contrast between hover and base < 1.5:1 means hover has no visual feedback value.

---

## Step 4 — Security Audit

Run before scoring. Fix critical issues immediately — do not defer to the fix pass.

### 4a — PHP output escaping scan

```bash
THEME_DIR=~/clients/<slug>/wp-content/themes/<slug>-theme

# Find echo statements without escaping functions
echo "=== Potentially unescaped output ==="
grep -rn 'echo \$' "$THEME_DIR" --include="*.php" | grep -v 'esc_\|wp_kses\|absint\|intval\|wp_json_encode'

# Find raw superglobal access
echo "=== Raw superglobal access ==="
grep -rn '\$_GET\|\$_POST\|\$_REQUEST' "$THEME_DIR" --include="*.php" | grep -v 'wp_verify_nonce\|isset.*wp_verify\|sanitize\|absint'

# Find direct SQL without prepare()
echo "=== Potentially unsafe SQL ==="
grep -rn '\$wpdb->' "$THEME_DIR" --include="*.php" | grep -v 'prepare\|insert\|update\|delete\|get_var\|get_row\|get_results\|get_col' | grep 'query\|get_var\|get_row'
```

Fix every finding — apply the correct `esc_*()` or `sanitize_*()` function per context. See `coding-standards.md` for the complete reference.

### 4b — WordPress hardening check

```bash
cd ~/clients/<slug>

# Is XMLRPC disabled? (should be in functions.php)
grep -r 'xmlrpc_enabled' wp-content/themes/<slug>-theme/functions.php && echo "XMLRPC filter: FOUND" || echo "XMLRPC filter: MISSING — add to functions.php"

# Are security headers sent? (should be in functions.php)
grep -r 'X-Frame-Options\|X-Content-Type\|Referrer-Policy' wp-content/themes/<slug>-theme/functions.php && echo "Security headers: FOUND" || echo "Security headers: MISSING — add to functions.php"

# Is version removed from head?
grep -r 'wp_generator\|the_generator' wp-content/themes/<slug>-theme/functions.php && echo "Version removal: FOUND" || echo "Version removal: MISSING"

# Is user enumeration blocked?
grep -r 'author.*redirect\|rest_endpoints.*users' wp-content/themes/<slug>-theme/functions.php && echo "User enumeration: BLOCKED" || echo "User enumeration: NOT BLOCKED"
```

If any of the above are MISSING, add the full hardening block from `coding-standards.md → WordPress Hardening` to `functions.php`. This is a single commit: `fix: security hardening — XMLRPC, headers, enumeration`

### 4c — JavaScript security scan

```bash
THEME_DIR=~/clients/<slug>/wp-content/themes/<slug>-theme

# console.log remaining
grep -rn 'console\.log' "$THEME_DIR" --include="*.js"

# innerHTML with dynamic data (potential XSS)
grep -rn 'innerHTML' "$THEME_DIR" --include="*.js"

# eval or new Function
grep -rn '\beval\b\|new Function' "$THEME_DIR" --include="*.js"

# Hardcoded URLs or credentials in JS
grep -rn 'http://\|https://' "$THEME_DIR" --include="*.js" | grep -v 'fonts.googleapis\|schema.org\|example.com'
```

Fix all `console.log` — remove them. Flag any `innerHTML` with dynamic data for review. Replace with `textContent` or safe DOM construction.

### 4d — Forms and nonces

Identify every form in the theme:
```bash
grep -rn '<form' ~/clients/<slug>/wp-content/themes/<slug>-theme/ --include="*.php"
```

For each form that submits data:
- Is `wp_nonce_field()` present inside the form? → If not, add it.
- Is the handler calling `wp_verify_nonce()` before processing? → If not, add it.
- Is input sanitized before use or storage? → If not, apply `sanitize_text_field()` / `sanitize_email()` etc.

**Note:** Contact Form 7 handles its own nonces internally. Only custom forms need manual nonce implementation.

### 4e — Security scorecard entry

```
SECURITY               [score]/10
  PHP Output:
  - [all escaped / N unescaped echoes found → fixed]
  WordPress Hardening:
  - XMLRPC: [disabled / missing]
  - Security headers: [present / missing]
  - Version disclosure: [removed / present]
  - User enumeration: [blocked / open]
  Forms & Nonces:
  - [N forms found, all have nonces / N forms missing nonces → fixed]
  JavaScript:
  - console.log: [none / N found → removed]
  - innerHTML risks: [none / flagged]
```

**Scoring guide:**
- 10: All checks pass, zero unescaped output, all hardening present, all forms nonce-protected
- 8–9: Minor gaps (one missing header, one console.log) — all fixed
- 6–7: Some hardening missing, fixed this pass
- 4–5: Unescaped user output found — critical, must fix before packaging
- 1–3: Multiple critical issues — XSS vectors, no nonces, no hardening

---

## Step 5 — Score each dimension

Rate each area 1–10 and list specific issues found. Use this exact scorecard:

```
UI/UX/CX SCORECARD — [CLIENT_NAME]
====================================

VISUAL DESIGN          [score]/10
  Issues:
  - [specific finding with page and element]

TYPOGRAPHY             [score]/10
  Issues:
  - [hierarchy problems, sizing inconsistencies, readability]

SPACING & RHYTHM       [score]/10
  Issues:
  - [inconsistent padding, cramped sections, excessive gaps]

COLOUR SYSTEM          [score]/10
  Contrast failures:
  - [pair]: [ratio]:1 — FAIL (need 4.5:1 for normal text, 3:1 for large)
  Accent usage:
  - --color-accent: [N] uses — [ok / overused / severely overused]
  Palette issues:
  - [monotone / semantic gaps / hover indistinguishable]

NAVIGATION & WAYFINDING [score]/10
  Issues:
  - [unclear active states, missing breadcrumbs, dead-end pages]

MOBILE EXPERIENCE      [score]/10
  Issues:
  - [breakpoint failures, touch targets, overflow]

CONVERSION PATHS (CX)  [score]/10
  Issues:
  - [weak CTAs, buried contact info, missing trust signals]

CONTENT QUALITY        [score]/10
  Issues:
  - [placeholder text remaining, thin copy, missing sections]

ACCESSIBILITY          [score]/10
  Issues:
  - [contrast, focus styles, alt text, landmarks, heading order]

SEO BASICS             [score]/10
  Issues:
  - [missing meta, duplicate H1s, poor titles, no structured data]

SECURITY               [score]/10
  (Carried from Step 4 audit — include findings summary here)

OVERALL                [avg]/10
```

**Scoring guide for COLOUR SYSTEM:**
- 10: All pairs pass AA, accent used ≤20 times, palette has ≥2 distinct hue families, semantic colours are distinct
- 8–9: All AA pass, minor overuse or slight monotony
- 6–7: 1 contrast failure OR accent overuse (30–50 uses) OR monotone palette
- 4–5: CTA button fails AA OR accent severely overused (>50 uses)
- 1–3: Multiple AA failures including body text

---

## Step 6 — Fix everything

Work through every issue in the scorecard. Fix autonomously — do not ask for permission on individual items.

### Priority order
1. **Security issues** — unescaped output, missing nonces, no hardening (Step 4 findings)
2. **Broken / missing content** — blank sections, placeholder text still showing
3. **Conversion path failures** — weak/missing CTAs, buried contact info
4. **Accessibility & contrast blockers** — WCAG AA failures
5. **Mobile layout issues** — overflow, broken grid, tiny touch targets
6. **Colour system issues** — accent overuse, hover indistinguishability, semantic gaps
7. **Visual inconsistencies** — spacing anomalies, off-palette values, font size drift
8. **Copy refinements** — thin sections, generic phrasing, missing trust signals
9. **SEO gaps** — missing meta descriptions, heading hierarchy, structured data

### For each fix
- Read the relevant file(s) before editing
- Make the targeted change
- Do NOT refactor surrounding code — surgical edits only
- Commit after every 3–5 related fixes: `fix: refine [area] — [brief description]`

---

### Colour fixes — specific rules

#### Contrast failures

**CTA button text contrast failure** (`cta-text` on `cta` < 4.5:1):

Read `_variables.css`. Find `--color-cta`. Darken it step by step until the contrast with `--color-cta-text` reaches ≥4.5:1. Use this Python snippet to find the target:

```python
python3 << 'EOF'
def hex_to_rgb(h):
    h = h.lstrip('#')
    return tuple(int(h[i:i+2], 16)/255 for i in (0, 2, 4))

def luminance(r, g, b):
    c = [x/12.92 if x <= 0.03928 else ((x+0.055)/1.055)**2.4 for x in (r,g,b)]
    return 0.2126*c[0] + 0.7152*c[1] + 0.0722*c[2]

def contrast(h1, h2):
    l1 = luminance(*hex_to_rgb(h1))
    l2 = luminance(*hex_to_rgb(h2))
    return (max(l1,l2)+0.05) / (min(l1,l2)+0.05)

import colorsys

def darken_hex(h, factor):
    r,g,b = [x/255 for x in (int(h[1:3],16), int(h[3:5],16), int(h[5:7],16))]
    hh,s,v = colorsys.rgb_to_hsv(r,g,b)
    v2 = max(0, v * factor)
    r2,g2,b2 = colorsys.hsv_to_rgb(hh,s,v2)
    return '#{:02x}{:02x}{:02x}'.format(int(r2*255),int(g2*255),int(b2*255))

cta = '#CURRENT_CTA_HEX'
text = '#CURRENT_CTA_TEXT_HEX'
print(f"Current: {cta} → contrast with text: {contrast(cta,text):.2f}:1")
for f in [0.9, 0.8, 0.75, 0.7, 0.65, 0.6]:
    d = darken_hex(cta, f)
    r = contrast(d, text)
    print(f"  factor {f}: {d} → {r:.2f}:1 {'✓ AA' if r>=4.5 else ''}")
EOF
```

Pick the lightest option that achieves ≥4.5:1. Update `--color-cta` in `_variables.css`. Also update `--color-accent` if they share the same value.

**`--color-subtle` contrast failure** (on bg or surface < 4.5:1):

Lighten `--color-subtle` until it achieves 4.5:1 against the darker of `--color-bg` and `--color-surface`. Keep it visually distinct from `--color-muted` (ensure ≥1.3:1 between them).

**`--color-muted` failure** (< 4.5:1):

Same approach — lighten to achieve 4.5:1.

#### Accent overuse (>30 uses)

Identify which CSS rules use `--color-accent` for non-accent purposes (borders, background tints, decorative lines, icon strokes that are not CTAs or key headings). Replace those specific instances with `--color-border`, `--color-border-2`, `--color-muted`, or `--color-surface-2` as appropriate. Do not change uses on: primary headings, CTA elements, key stat numbers, active nav items.

Do NOT bulk-replace — read each CSS file section and make targeted substitutions.

#### Hover indistinguishable (accent vs accent-2 contrast < 1.5:1)

If the hover colour is too similar to the base, note it in the scorecard as "needs client decision on hover direction — cannot auto-fix without brand input." Do not guess a new hover colour.

#### Monotone palette

Note it in KNOWN LIMITATIONS. Do not invent brand colours — this requires client sign-off.

---

### Other specific checks per file type

**PHP templates:**
- All sections present and populated with real content
- No hardcoded inline styles (use CSS classes)
- Correct `esc_html()` / `esc_url()` on all output
- Image alt attributes meaningful, not empty or "image"
- CTAs have descriptive text (not just "Click here")

**CSS files:**
- No magic numbers — all values reference CSS custom properties
- Mobile-first confirmed — no `max-width` media queries (use `min-width`)
- `clamp()` on all font sizes
- Hover/focus states on all interactive elements
- Consistent section padding using spacing scale

**JS files:**
- No `console.log` remaining
- Scroll reveal targets `.reveal` class — verify elements have it
- No jQuery imports

**WordPress / WP-CLI checks:**
```bash
# NOTE: use this exact format — do NOT use --profile cli flag

# Verify all pages have meta descriptions set via Yoast
cd ~/clients/<slug> && docker compose run --rm wpcli post list --post_type=page --fields=ID,post_title,post_status

# Check for any posts still in draft
docker compose run --rm wpcli post list --post_status=draft

# Flush rewrites
docker compose run --rm wpcli rewrite flush

# Set missing Yoast meta descriptions
docker compose run --rm wpcli post meta update <ID> _yoast_wpseo_metadesc "<description>"
```

---

## Step 7 — Re-score after fixes

Re-run the contrast checker and accent count from Step 3. Re-fetch key pages. Re-score:

```
REFINED SCORECARD
==================
[repeat scorecard format]

IMPROVEMENTS
  [dimension]: [old score] → [new score] — [what was fixed]

KNOWN LIMITATIONS (cannot fix without client input)
  - [item requiring real assets, credentials, brand decisions, or real contact details]
```

---

## Step 8 — Update project state

Update `SESSION_STATE.json`:
```json
{
  "steps": {
    "ui_ux_refinement": "complete"
  },
  "refinement_scores": {
    "before": { "overall": X },
    "after":  { "overall": Y }
  },
  "colour_audit": {
    "contrast_failures_fixed": ["list of fixed pairs"],
    "accent_uses_before": N,
    "accent_uses_after": N,
    "monotone_palette": true_or_false
  },
  "security_audit": {
    "unescaped_output_fixed": N,
    "hardening_added": true,
    "forms_nonce_verified": N,
    "console_logs_removed": N,
    "remaining_risks": ["any items needing client-side config or server-level fixes"]
  },
  "notes": ["<any remaining items that need client input>"]
}
```

Final commit:
```bash
cd ~/clients/<slug>
git add .
git commit -m "refine: UI/UX/CX pass — overall score [before] → [after]/10"
```

---

## Step 8 — Present the refinement report

```
REFINEMENT COMPLETE — [CLIENT_NAME]
=====================================
Pages reviewed:   [n]
Issues found:     [n]
Issues fixed:     [n]
Overall score:    [before] → [after]/10

COLOUR SYSTEM
  Contrast failures fixed: [n]
  Accent overuse: [before] → [after] uses
  Palette notes: [pass / monotone flagged / semantic gaps noted]

TOP IMPROVEMENTS
  ✓ [specific improvement]
  ✓ [specific improvement]
  ✓ [specific improvement]

STILL NEEDS CLIENT INPUT
  • [item]
  • [item]

Preview: http://localhost:<port>
```

Do NOT proceed to packaging automatically. Wait for user sign-off.
