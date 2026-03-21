---
description: "Run a website audit only — no project scaffolding or build"
---

# /wp-review — Site Audit Only

Use this for initial client consultations — analyze a site without committing to a build.

## Gather inputs

- Site URL to audit (required)
- Competitor URLs (optional, 0–3)
- Industry context (optional)

## Execute audit

Use WebFetch on each URL. Produce structured analysis:

### Site Audit Report

1. **Visual Design** — color palette, typography, layout patterns, imagery, overall aesthetic
2. **Content** — pages, sections, CTAs, trust signals, content quality
3. **UX/CX** — navigation, mobile responsiveness, conversion paths, accessibility
4. **Technical** — page builder detected, plugin bloat, performance, SEO basics
5. **Colour System** — see dedicated section below
6. **Strengths** — what's working and should be preserved
7. **Weaknesses** — what's failing and needs to change

---

### Colour System Audit

For every site reviewed, perform this structured colour analysis. Extract colours from the page source (CSS variables, inline styles, stylesheet links) and evaluate:

#### A — Contrast (WCAG AA compliance)

Identify the background colour, card/surface colour, primary text colour, secondary text colour, and CTA button colour+text. Run contrast ratios:

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

# Fill in from site inspection:
checks = [
    ('#TEXT_HEX',    '#BG_HEX',      'Body text on background'),
    ('#MUTED_HEX',   '#BG_HEX',      'Secondary text on background'),
    ('#CTA_TEXT',    '#CTA_BG',      'CTA button text on button'),
    ('#ACCENT_HEX',  '#BG_HEX',      'Accent on background'),
]

print(f"{'Pair':<35} {'Ratio':>6}  {'AA (4.5:1)':>11}  {'AA Large (3:1)':>14}")
print("-" * 72)
for text, bg, label in checks:
    r = contrast(text, bg)
    aa  = "PASS" if r >= 4.5 else "FAIL ⚠"
    aal = "PASS" if r >= 3.0 else "FAIL ⚠"
    print(f"{label:<35} {r:>6.2f}  {aa:>11}  {aal:>14}")
EOF
```

Report each pair with pass/fail and ratio.

#### B — Accent overuse

Count how many distinct elements use the primary accent colour (buttons, links, headings, borders, icons, backgrounds). Rate:
- 1–10 uses: well-controlled
- 11–25 uses: moderate — may dilute impact
- 26–50 uses: overused — accent loses meaning
- 50+ uses: severely overused — redesign needed

#### C — Palette hue variety

List all unique hues in the palette. Flag if:
- All colours within ±30° of a single hue → **monotone palette** — low visual hierarchy
- No warm tones present in a B2B/industrial site → **cold/clinical** — may undermine trust
- More than 5 distinct accent colours → **fragmented** — no visual coherence

#### D — Industry fit

Assess whether the palette matches the emotional expectations of the target audience:
- **B2B industrial / infrastructure**: trust, precision, reliability → navy, white, warm accent (amber/orange/green)
- **Technology / SaaS**: innovation, speed → dark mode, electric blue/purple are common but overused
- **Finance / legal**: authority, stability → conservative navy, dark grey, gold
- **Healthcare**: clean, trustworthy → white/light, teal or blue, avoid reds
- **E-commerce / consumer**: energy, conversion → high contrast, warm CTAs (orange/red)

Note if the current palette signals the wrong emotional register for the industry.

#### E — Semantic clarity

- Is success (green/teal) visually distinct from the accent colour?
- Is error (red) clearly different from both?
- Are hover states meaningfully different from base states (>1.5:1 contrast between them)?

---

### Colour Audit Output Format

```
COLOUR SYSTEM ANALYSIS
=======================

Palette identified:
  Background:    #HEX — [description]
  Surface:       #HEX — [description]
  Primary text:  #HEX
  Secondary text:#HEX
  Accent:        #HEX
  CTA button:    #HEX / text: #HEX

Contrast results:
  Body text on bg:        X.XX:1  [PASS/FAIL AA]
  Secondary text on bg:   X.XX:1  [PASS/FAIL AA]
  CTA button text:        X.XX:1  [PASS/FAIL AA]
  Accent on bg:           X.XX:1  [PASS/FAIL AA]

Accent usage:      [N] elements — [ok / overused / severely overused]
Hue variety:       [monotone / limited / varied]
Industry fit:      [aligned / misaligned — reason]
Semantic clarity:  [clear / issues noted]

COLOUR SCORE: [X]/10
  Key issues:
  - [specific finding]
  - [specific finding]
```

---

### Competitor Comparison (if URLs provided)

For each competitor: what they do better, design patterns worth learning from,
features the client is missing. Include a brief colour comparison — note if
competitors have better contrast compliance or more distinctive palette choices.

### Recommendations

Priority-ranked list of improvements with rationale. For colour issues, always specify:
- The exact pairs failing contrast and by how much
- Which accent colour to use and where to restrict it
- Whether the palette needs a full rethink or just calibration

## Output format

Present the audit directly in the chat — clean, structured markdown.
Do NOT create project files or directories. This is a consultation tool only.

If the user wants to proceed with a build after the review, tell them to run `/wp-start`.
