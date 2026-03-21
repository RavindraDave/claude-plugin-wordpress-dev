---
description: "Start a new WordPress project — handles both new builds and existing site redesigns"
---

# /wp-start — Start a New WordPress Project

## Step 0 — Detect Scenario and Build Approach

**Scenario detection:**
- User provides a website URL → **Scenario 1 (Redesign)**
- No URL, just a business description → **Scenario 2 (New Build)**

**Build approach detection:**
- User mentions "Elementor", "page builder", "drag and drop", or "client will edit layouts" → **Elementor build**
- Existing site being redesigned is built on Elementor AND client wants to keep editor access → **Elementor build**
- No mention of page builder, OR user says "custom", "fast", "performance", "SEO" → **Custom PHP build** (default)
- If unclear — present the trade-offs from `references/elementor-vs-custom.md` and ask the client to decide. Record their choice before proceeding.

Both scenarios follow the same intake, audit, and design system derivation (Phases 1–2).
The build approach determines the scaffold and build steps (Phases 3–4).

---

## SCENARIO 1: Existing Site Redesign

### Phase 1A — Intake

Extract from what the user provides:
- Client/business name → `slug` (lowercase, hyphenated)
- Existing site URL
- Competitor URLs (0–3)
- What's wrong with the current site / what they want improved
- Target audience
- Budget/scope context (full redesign vs specific pages)
- Hosting platform (for deployment packaging later)
- Any brand guidelines, assets, or constraints
- **Build approach** — custom PHP (default) or Elementor? If not stated, detect from Step 0 signals. If still unclear, include in the clarifying questions batch.

**Ask ALL clarifying questions in ONE batch — max 5.** Smart defaults:
- Port: auto-select (read `~/clients/.port-registry.json` and `ls ~/clients/`)
- Contact form: always include
- SEO plugin: always include
- Permalinks: `/%postname%/`
- Theme type: classic PHP (for rebuilds)
- Blog: include if "news/articles/content" mentioned

### Phase 1B — Full Site Scrape & Analysis

**Step 1 — Discover all pages**

Fetch the homepage and extract every internal link:
```
WebFetch(<client_url>)
```
Parse `<a href>` tags. Build a deduplicated list of internal URLs (same domain, no anchors, no admin paths). Limit to 30 pages maximum — prioritise pages in the main navigation.

**Step 2 — Crawl every discovered page**

For each URL in the list, `WebFetch(<url>)` and extract:
- Full page title and meta description
- All headings (H1–H3) in order
- Body copy per section (full text, not truncated)
- All CTAs (button text + destination)
- Contact details (address, phone, email, hours)
- Form fields present
- Images (alt text, likely subject)
- Any testimonials, stats, certifications, accreditations, awards
- Navigation items and footer links
- **CSS/style clues:** inline styles, linked stylesheets, colour hex values, font names, class patterns

Store extracted content in a structured scratch object keyed by URL. This becomes the **content source of truth** for the new build — use real copy everywhere, never placeholder text.

**Step 3 — Evaluate the site as a whole**

Synthesise the crawl data into a structured audit:

1. **Visual Design Audit**
   - Actual hex/RGB colour values found in CSS or inline styles
   - Logo colours (note dominant hues — these are brand colours)
   - Typography (fonts, hierarchy, sizing)
   - Layout patterns (is it generic Elementor templates?)
   - Spacing and density
   - Imagery style and quality
   - Overall aesthetic: what's dated, generic, or broken

2. **Content Inventory**
   - Full page list with purpose and key sections
   - CTAs, forms, trust signals across all pages
   - Content to carry over vs discard
   - Content gaps (pages a business like this should have but doesn't)

3. **UX/CX Assessment**
   - Navigation structure and clarity
   - Mobile responsiveness signals
   - Conversion path: can a user easily take the desired action?
   - Trust signals: testimonials, certifications, social proof
   - Brand consistency across pages

4. **Technical Snapshot**
   - Page builder detected (Elementor/Divi/WPBakery class patterns)
   - Plugin bloat signals
   - Performance observations (heavy images, render-blocking resources)
   - SEO basics (meta tags, headings, structured data present?)

**Step 4 — Competitor analysis** (if URLs provided)

For each competitor URL, `WebFetch(<url>)` and analyse:
- What they do better than the client
- Design patterns worth learning from (NOT copying)
- Features the client is missing
- Layout/UX approaches that convert
- Colour and typography choices (for differentiation, not copying)

**Output:**
- Write `~/clients/<slug>/reports/SITE_AUDIT.md` — full crawl results + evaluation
- If competitors: write `~/clients/<slug>/reports/COMPETITOR_ANALYSIS.md`
- Carry the extracted content object forward — Phase 4 build steps must use this real copy

### Phase 1C — Expert Design System Derivation

This is not a template fill-in. Apply expert colour psychology and typography science.

**Step 1 — Brand Asset Detection**

From the site crawl, extract:
- Actual hex/RGB values from CSS stylesheets or inline styles
- Logo colours (if SVG, parse fill values; note dominant colours)
- Any stated brand guidelines or colour codes in page content
- Fonts currently in use (Google Fonts URL params, CSS font-family declarations)

**Decision:** If brand colours are found → derive the new palette FROM those colours (adjusting lightness/saturation, finding accessible complements). Do NOT invent a new palette that abandons the brand.

**Step 2 — Industry + Emotional Register**

If no brand colours found, or the existing palette is broken/generic (pure white + one random blue), derive from industry:

```
INDUSTRY                 EMOTIONAL REGISTER      PRIMARY RANGE           ACCENT STRATEGY
──────────────────────────────────────────────────────────────────────────────────────────
B2B Industrial /         Trust, precision,       Navy #0D1B2E–#1E3A5F    Warm amber #D97706
Infrastructure /         authority, capability   OR steel grey #2C3E50   OR steel blue #0369A1
Engineering                                      Light base preferred

Technology / SaaS /      Innovation, speed,      Dark #0A0F1C–#1A1F35    Electric blue #0EA5E9
Developer Tools          intelligence            Dark mode valid          OR violet #7C3AED

Healthcare / Medical /   Clinical, clean,        Light base #F8FAFC–     Teal #0D9488
Wellness / Pharmacy      trustworthy, calm       #FFFFFF mandatory        OR blue #2563EB
                                                 NO dark mode

Finance / Legal /        Authority, stability,   Navy #1E293B or         Gold #B45309
Professional Services    precision, premium      dark grey #1F2937        OR deep green #065F46

Retail / E-commerce /    Engagement, energy,     Brand-driven.           Warm CTA: orange
Consumer                 desire, conversion      Light base.             #EA580C or red #DC2626

Food / Hospitality /     Warmth, experience,     Cream, mocha, earth,    Saffron, terracotta,
Travel / Lifestyle       aspiration, pleasure    forest — warm bases      olive, forest green

Education / Non-profit   Inclusive, accessible,  Light, open base.       Trustworthy blue
                         trustworthy             WCAG AAA target          or forest green

Real Estate /            Professional,           Warm grey #374151        Gold #92400E
Construction             aspirational, quality   or navy #1E3A5F          or earth brown

Creative / Agency /      Distinctive, bold,      Any base — must stay    Single bold accent,
Media / Design           memorable               accessible               maximum contrast
──────────────────────────────────────────────────────────────────────────────────────────
KEY RULE: Pure dark mode = tech startup or gaming signal. For healthcare, finance,
          construction, B2B services — default LIGHT BASE with dark accents unless
          client explicitly requests dark and understands the brand implication.
```

**Step 3 — Build the Full Palette**

Construct all tokens. Every value must be a deliberate choice, not a random hex.

```
Token                 Purpose                       Required Contrast
────────────────────────────────────────────────────────────────────────
--color-bg            Page background               —
--color-surface       Cards, alternate sections     —
--color-surface-2     Subtle elevation / dividers   —
--color-text          Primary body copy             ≥7.0:1 on bg (AAA preferred)
--color-muted         Secondary text, captions      ≥4.5:1 on bg AND on surface (AA)
--color-subtle        Labels, placeholders          ≥4.5:1 on bg AND on surface (AA)
--color-accent        Brand accent (display only)   ≥3.0:1 on bg (icons/large text)
--color-cta           CTA button background         —
--color-cta-text      CTA button text               ≥4.5:1 on --color-cta (AA)
--color-cta-hover     Button hover background       ≥4.5:1 with cta-text; differs ≥1.5:1 from cta
--color-border        Default dividers              —
--color-border-2      Stronger borders              —
--color-dark          Footer, darkest areas         —
--color-success       Success states                ≥4.5:1 on surface; distinct hue from accent
--color-warning       Warning states                ≥3.0:1 on bg; warm/amber hue
--color-error         Error states                  ≥4.5:1 on surface; clearly red/warm
```

**Step 4 — WCAG Validation (mandatory — do not skip)**

Run the Python validator. Fix ALL failures before proceeding:

```python
python3 << 'EOF'
def lum(h):
    h = h.lstrip('#')
    c = [int(h[i:i+2],16)/255 for i in (0,2,4)]
    c = [x/12.92 if x<=0.03928 else ((x+0.055)/1.055)**2.4 for x in c]
    return 0.2126*c[0]+0.7152*c[1]+0.0722*c[2]
def cr(a,b):
    L=[lum(a),lum(b)]; return (max(L)+0.05)/(min(L)+0.05)

# Fill in actual palette values:
palette = dict(
    bg='#HEX', surface='#HEX', text='#HEX', muted='#HEX',
    subtle='#HEX', cta='#HEX', cta_text='#HEX', cta_hover='#HEX', accent='#HEX'
)
checks = [
    ('text','bg',7.0,'Body text on background (AAA)'),
    ('muted','bg',4.5,'Secondary text on background (AA)'),
    ('muted','surface',4.5,'Secondary text on card (AA)'),
    ('subtle','bg',4.5,'Subtle text on background (AA)'),
    ('subtle','surface',4.5,'Subtle text on card (AA)'),
    ('cta_text','cta',4.5,'CTA button text (AA)'),
    ('cta_text','cta_hover',4.5,'CTA hover text (AA)'),
    ('accent','bg',3.0,'Accent on background (large/icon)'),
]
failures=[]
for a,b,req,label in checks:
    r=cr(palette[a],palette[b])
    ok=r>=req
    print(f"{'PASS' if ok else 'FAIL ⚠'} {label:<40} {r:.2f}:1 (need {req}:1)")
    if not ok: failures.append(label)
if failures:
    print(f"\n{len(failures)} failure(s). Fix palette before proceeding.")
else:
    print("\nAll pairs pass. Palette approved.")
EOF
```

**Step 5 — Typography Selection**

Never "just pick a Google Font." Apply industry-matched science:

Pairing rule: Display font = personality (weights 700–800, tight tracking). Body font = readability (weights 300–600, neutral tracking). They must be visually distinct.

```
Industry Type          Display (headings)            Body (copy)
────────────────────────────────────────────────────────────────────────
Industrial / B2B       Syne, Barlow, DM Sans         Inter, Source Sans Pro
Technology / SaaS      Space Grotesk, Plus Jakarta   Inter, DM Sans
Healthcare             Nunito, DM Sans               Open Sans, Lato
Finance / Legal        Libre Baskerville, Merriweather  Source Sans Pro, Lato
Retail / Consumer      Outfit, Josefin Sans          Inter, Nunito
Luxury / Premium       Cormorant Garamond, Playfair  Lato, Raleway
Education              Poppins, Nunito               Open Sans, Lato
Creative / Agency      Clash Display, Cabinet Grotesk   Inter, Satoshi
────────────────────────────────────────────────────────────────────────
```

**Step 6 — Accent Usage Budget**

Write into CONTEXT.md and enforce during build:

```
Accent colour budget: ≤20 CSS rules total

ALLOWED:
  ✓ CTA button backgrounds and primary link text
  ✓ Active/current navigation states
  ✓ Hero headline accent spans and eyebrow rule lines
  ✓ Key stat numbers and primary counters
  ✓ Focus outlines (accessibility non-negotiable)
  ✓ Section dividers (.divider--accent variant only)
  ✓ Hover states on interactive cards (border OR glow — not both)

NOT ALLOWED:
  ✗ Decorative card icons in grids (use --color-muted)
  ✗ Every section eyebrow/label (alternate with --color-muted)
  ✗ Hover borders when box-shadow glow already signals hover
  ✗ Footer utility icons, contact icons, informational icons
  ✗ Partner/client logo hover effects (opacity alone is sufficient)
```

**Output — Design Direction Block:**

```
DESIGN DIRECTION — [CLIENT NAME]
=================================
Industry context:  [detected industry + emotional register]
Brand colours:     [found / not found — source]

PALETTE (all WCAG-validated)
  --color-bg:        #HEX  — [role]
  --color-surface:   #HEX  — [role]
  --color-text:      #HEX  — [contrast ratio]:1 on bg
  --color-muted:     #HEX  — [contrast ratio]:1 on bg / [ratio]:1 on surface
  --color-subtle:    #HEX  — [contrast ratio]:1 on bg / [ratio]:1 on surface
  --color-accent:    #HEX  — [contrast ratio]:1 on bg (large text/icons)
  --color-cta:       #HEX  — CTA background
  --color-cta-text:  #HEX  — [contrast ratio]:1 on cta
  --color-dark:      #HEX  — footer/darkest

TYPOGRAPHY
  Display: [Font Name] — [why: personality match, visual character]
  Body:    [Font Name] — [why: readability, weight range]
  Google Fonts URL: https://fonts.googleapis.com/css2?...

LAYOUT
  [2-3 sentences: asymmetry, density, whitespace, grid approach]

ANIMATION
  [none/subtle/moderate — key effects and rationale]

BRAND PERSONALITY
  [one line: what the site should feel like to a visitor]

HERO TREATMENT
  [A — Geometric / B — Split BG / C — Gradient mesh / D — Editorial type]
  Choice: [X] — [reason tied to brand and industry]

DIFFERENTIATION
  vs existing site: [specific improvement]
  vs competitor(s): [how this stands out]

STRUCTURED DATA
  Schema type: [LocalBusiness/ProfessionalService/etc — by industry]
```

### Phase 1D — Present & Confirm

Show the user:
1. Key findings from the site audit (what's wrong, what to keep)
2. The complete Design Direction block above
3. Proposed sitemap (pages + sections)
4. Plugin recommendations with rationale
5. Any questions (max 5, ideally 0–3)

After user confirms → proceed autonomously.

---

## SCENARIO 2: New Build from Scratch

### Phase 1A — Intake

If the user hasn't provided enough detail, present the key questions in ONE batch:

- Business name, industry, nature of business
- Target audience
- Site goal (leads, sales, portfolio, bookings, etc.)
- Pages/sections needed
- Features (forms, blog, e-commerce, booking, gallery, etc.)
- Design preferences (colours, style, reference sites they like)
- Brand guidelines or logo (ask for file upload or describe)
- Content availability (do they have copy/images, or need placeholders?)
- Hosting platform
- **Build approach** — custom PHP or Elementor? (include in questions batch if unclear from Step 0)

**Smart defaults** — decide without asking:
- Port, tagline, contact form, permalinks, SEO plugin
- Colours from industry framework if none specified (Step 2 above)
- Typography from industry-matched pairings if none specified
- Blog if content marketing mentioned
- Standard pages (Home, About, Services, Contact) if not specified

### Phase 1B — Expert Design System Derivation

No existing site to crawl. Apply the same 6-step expert derivation:

1. **Brand asset check** — did user provide hex codes, brand colours, logo? Use them as the anchor.
2. **Industry framework** — if no colours given, apply the table above
3. **Build full palette** — all 16 tokens
4. **WCAG validation** — run Python checker, fix failures
5. **Typography selection** — industry-matched pairing
6. **Accent budget** — document in CONTEXT.md

### Phase 1C — Present & Confirm

Show:
1. Your understanding of the project
2. Full Design Direction block (same format as Scenario 1)
3. Proposed sitemap with URL structure
4. Plugin list with rationale
5. Content strategy (hero headline formula, CTA copy, trust signals)
6. Questions (max 5)

After user confirms → proceed autonomously.

---

## Phase 2 — PRD Generation

**For both scenarios**, generate a PRD at `~/clients/<slug>/PRD.md`.

Use the template at `~/wordpress/templates/PRD_TEMPLATE.md` as the structure.

The PRD must include:
- Project overview (client, goals, audience, scope)
- **Build approach:** `custom-php` or `elementor` — confirmed by client
- Complete sitemap with URL structure
- Page-by-page specs (purpose, sections, content blocks, interactive elements)
- Design system (complete validated palette, typography, spacing, components, breakpoints)
- Accent usage budget (≤20 rules for custom-php; Global Colors tokens for elementor)
- Technical architecture (theme structure, CPTs if needed, plugin list)
- Content plan (what carries over, what's new, what's placeholder)
- SEO requirements (per-page targets if known)
- Structured data schema type
- If Elementor: Elementor Pro licence available? (yes/no — affects which widgets are available)

The PRD is the **single source of truth** for all development. Every design and content decision references it.

---

## Phase 3 — Scaffold

### Step 1 — Port allocation

```bash
# Check existing clients
ls ~/clients/ 2>/dev/null
cat ~/clients/.port-registry.json 2>/dev/null
```

Pick next available port (8082, 8084, 8086...).

### Step 2 — Run scaffold script

```bash
cd ~/wordpress && ./scripts/new-client.sh <slug> <port>
```

### Step 3 — Create reports directory

```bash
mkdir -p ~/clients/<slug>/reports ~/clients/<slug>/docs
```

### Step 4 — Move/write project files

- Move SITE_AUDIT.md and COMPETITOR_ANALYSIS.md into `reports/` (if Scenario 1)
- Write PRD.md at project root
- Write CONTEXT.md (condensed design reference — read at start of every session)
- Write SESSION_STATE.json (initial state)

### Step 5 — Git branches

```bash
cd ~/clients/<slug>
git checkout -b staging
git checkout -b dev
git add . && git commit -m "docs: add PRD, design context, and site audit reports"
```

### Step 6 — Update port registry

Write/update `~/clients/.port-registry.json`:
```json
{
  "<slug>": { "wp": <port>, "pma": <port+1>, "created": "YYYY-MM-DD" }
}
```

---

## Phase 4 — Theme Development

**Read the skill references before building:**
- `references/theme-scaffold.md` — file structure and PHP boilerplate
- `references/design-tokens.md` — CSS custom property system
- `references/coding-standards.md` — PHP/CSS/JS standards

Build in this exact sequence. Commit after each step.

### Step 1: Theme skeleton

Create `~/clients/<slug>/wp-content/themes/<slug>-theme/` with all directories and boilerplate files per `references/theme-scaffold.md`.

- `style.css` — theme header
- `functions.php` — setup, enqueue chain, menu registration, theme supports
- `header.php`, `footer.php` — skeleton with `wp_head()`, `wp_footer()`, skip link
- `index.php`, `front-page.php`, `page.php`, `single.php`, `archive.php`, `404.php`
- `template-parts/` — empty PHP files for each section
- `assets/css/` — empty partials
- `assets/js/` — empty scripts

Activate the theme:
```bash
cd ~/clients/<slug>
docker compose run --rm wpcli theme activate <slug>-theme
```

Commit: `chore: scaffold <slug>-theme classic PHP structure`

### Step 2: Design tokens (`assets/css/_variables.css`)

Fill in ALL values from the PRD/CONTEXT.md validated design system. The palette here must exactly match the WCAG-validated palette from Phase 1C. Includes:
- Google Fonts import (display + body)
- Font families and fluid clamp type scale
- Full colour palette (all 16 tokens, zero hardcoded values elsewhere)
- Spacing (4px base unit scale)
- Layout (container widths, section padding, grid gap)
- Effects (shadows, radius, transitions, z-index layers)

Also build `_reset.css` and `_animations.css`.

Update `functions.php` to enqueue Google Fonts + all CSS partials in dependency order.

Commit: `feat: design token system — CSS custom properties, reset, animations`

### Step 3: Grid system (`assets/css/_grid.css` + `_utilities.css`)

- Container system (default/narrow/wide)
- CSS Grid layouts (grid-2, grid-3, grid-4, grid-auto, grid-hero, grid-feature)
- Section wrappers with background variants
- Flex helpers
- Spacing utilities, typography utilities, button system, visibility utilities

Mobile-first always: base = 375px, `min-width` queries only.

Commit: `feat: responsive grid system and utility classes`

### Step 4: Navigation

- `header.php` — logo + `wp_nav_menu()` + hamburger button + mobile overlay
- `assets/css/_nav.css` — sticky header, desktop nav, hamburger animation, mobile overlay with staggered links
- `assets/js/nav.js` — scroll detection, hamburger toggle, Escape close, focus trap, body scroll lock

Commit: `feat: sticky navigation with mobile hamburger menu`

### Step 5: Hero section

- `template-parts/hero.php` — asymmetric layout (NOT centered text on bg image), eyebrow + headline + subheadline + dual CTAs + trust indicators
- `assets/css/_hero.css` — hero layout, visual treatment (per PRD choice A/B/C/D), responsive breakpoints, page-load animation sequence

Write **real, client-specific copy** using the content strategy formula:
- Headline: [Specific outcome] for [audience] — [differentiator or proof]
- Primary CTA: Verb + specific outcome ("Plan Your Fit-Out", "Get a Free Quote")
- Secondary CTA: Softer exploration ("See Our Work", "How It Works")
- Trust signals: years trading, project count, certifications, geographic proof

Do NOT use: "Welcome to Our Company", "Your Trusted Partner", "Learn More", "Click Here"

Commit: `feat: hero section with [treatment] visual approach`

### Step 6: Interior sections

Build each section specified in the PRD. Common ones:

- `template-parts/services.php` — grid cards with hover lift, numbered counters
- `template-parts/about.php` — split image/text with stat highlights
- `template-parts/testimonials.php` — cards with quote marks, stars, author info
- `template-parts/cta.php` — full-width band, high contrast, strong CTA
- `template-parts/contact.php` — form embed + contact info
- Additional sections per PRD (team, gallery, FAQ, pricing, etc.)

All CSS in `assets/css/_sections.css`.
Real, industry-appropriate copy everywhere. Trust signals present before first fold break.
Commit after each section.

### Step 7: Footer

- `footer.php` — 4-col grid (brand+social, nav, services, contact), bottom bar with copyright + legal links
- `assets/css/_footer.css`
- Inline SVG social icons (no icon font libraries)
- Contact icons use `--color-muted` (not accent)

Commit: `feat: responsive footer with inline SVG social icons`

### Step 8: Interior page templates

- `page.php` — generic page template
- `front-page.php` — assembles sections via `get_template_part()`
- `single.php` — blog post template (if blog needed)
- `archive.php` — blog archive (if blog needed)
- `404.php` — error page with search + nav links
- Any custom page templates per PRD

Commit: `feat: complete page templates per PRD`

### Step 9: WordPress configuration via WP-CLI

**IMPORTANT: Use this exact format. NEVER use `--profile cli`.**

```bash
cd ~/clients/<slug>

# Site identity
docker compose run --rm wpcli option update blogname "<title>"
docker compose run --rm wpcli option update blogdescription "<tagline>"

# Permalinks
docker compose run --rm wpcli option update permalink_structure "/%postname%/"
docker compose run --rm wpcli rewrite flush

# Create all pages from PRD
HOME_ID=$(docker compose run --rm wpcli post create \
  --post_type=page --post_status=publish --post_title="Home" --porcelain)
# ... create all other pages

# Set front page
docker compose run --rm wpcli option update show_on_front page
docker compose run --rm wpcli option update page_on_front $HOME_ID

# Install plugins per PRD
docker compose run --rm wpcli plugin install contact-form-7 --activate
docker compose run --rm wpcli plugin install wordpress-seo --activate
# ... others per PRD

# Create and assign nav menus
MENU_ID=$(docker compose run --rm wpcli menu create "Primary Menu" --porcelain)
docker compose run --rm wpcli menu location assign $MENU_ID primary
# ... add pages to menu

# SEO meta per page (use actual page IDs)
docker compose run --rm wpcli post meta update <PAGE_ID> _yoast_wpseo_metadesc "<meta description>"
```

**DB export (wpcli container lacks /backups mount — use db service directly):**
```bash
docker compose exec db mysqldump -u wpuser -pwppass123 wordpress > database/seed.sql
```

Commit: `feat: WordPress configuration — pages, menus, plugins, settings`

### Step 10: Structured data (JSON-LD)

Add to `header.php` or `template-parts/schema.php`:

```php
<?php
$schema = [
    '@context' => 'https://schema.org',
    '@type'    => 'LocalBusiness', // or ProfessionalService, MedicalBusiness, etc. — per PRD
    'name'     => get_bloginfo('name'),
    'url'      => home_url('/'),
    // Add: address, telephone, description, sameAs social URLs
];
echo '<script type="application/ld+json">' . wp_json_encode($schema, JSON_UNESCAPED_SLASHES) . '</script>';
?>
```

Schema type selection:
- B2B Services: `LocalBusiness` + `ProfessionalService`
- Healthcare: `MedicalBusiness`
- Restaurant: `Restaurant`
- Real Estate: `RealEstateAgent`
- SaaS/Tech: `SoftwareApplication` or `Organization`

Commit: `feat: JSON-LD structured data`

### Step 11: Scroll reveal JS

Add to `assets/js/main.js`:
- IntersectionObserver for `.reveal` elements
- Adds `.is-visible` class on scroll into viewport
- Respects `prefers-reduced-motion`

Commit: `feat: scroll-reveal animations via IntersectionObserver`

---

## Phase 4 (Elementor) — Elementor Build Path

**Only follow this phase if `build_approach: elementor` is confirmed in the PRD.**
Skip this entirely for custom PHP builds — they use the Phase 4 steps above.

### Step 1: Install Hello Elementor parent theme + child theme

```bash
cd ~/clients/<slug>

# Install and activate Hello Elementor (minimal, Elementor-optimised base)
docker compose run --rm wpcli theme install hello-elementor --activate

# Install Elementor (and Pro if licence provided)
docker compose run --rm wpcli plugin install elementor --activate
# If Elementor Pro licence key is available:
# Upload elementor-pro.zip via WP Admin → Plugins → Add New → Upload
# Then: docker compose run --rm wpcli plugin activate elementor-pro
```

Create a child theme — security hardening and design overrides go here, not in the parent:

```
wp-content/themes/<slug>-elementor/
├── style.css          ← child theme header + base overrides
├── functions.php      ← security hardening + custom CSS injection
└── assets/
    └── css/
        └── custom.css ← global typography, spacing, utility overrides
```

**`style.css` header:**
```css
/*
Theme Name:  [CLIENT NAME] — Elementor Theme
Template:    hello-elementor
Description: Elementor child theme for [CLIENT NAME]. Custom CSS overrides only.
Version:     1.0.0
Text Domain: [SLUG]
*/
```

**`functions.php`** — full security hardening block (same as custom PHP, non-negotiable) + custom CSS + Elementor configuration:

```php
<?php
/**
 * [CLIENT NAME] Elementor Child Theme
 * Security hardening applied — identical standards to custom PHP builds.
 */

// Enqueue child theme CSS after Elementor's styles
add_action( 'wp_enqueue_scripts', function () {
    wp_enqueue_style(
        '[SLUG]-child',
        get_stylesheet_directory_uri() . '/assets/css/custom.css',
        [ 'hello-elementor-style' ],
        wp_get_theme()->get( 'Version' )
    );
} );

// Disable Elementor's default colour and typography schemes
// so our Global Colors/Fonts are the only source of truth
add_action( 'elementor/init', function () {
    update_option( 'elementor_disable_color_schemes', '1' );
    update_option( 'elementor_disable_typography_schemes', '1' );
} );

// ── Security Hardening (non-negotiable on all builds) ──────
remove_action( 'wp_head', 'wp_generator' );
add_filter( 'the_generator', '__return_empty_string' );
add_filter( 'xmlrpc_enabled', '__return_false' );
remove_action( 'wp_head', 'rsd_link' );
remove_action( 'wp_head', 'wlwmanifest_link' );
remove_action( 'wp_head', 'wp_shortlink_wp_head' );
add_action( 'init', function () {
    if ( ! is_admin() && isset( $_GET['author'] ) ) {
        wp_safe_redirect( home_url( '/' ), 301 ); exit;
    }
} );
add_filter( 'rest_endpoints', function ( $endpoints ) {
    if ( ! is_user_logged_in() ) {
        unset( $endpoints['/wp/v2/users'] );
        unset( $endpoints['/wp/v2/users/(?P<id>[\d]+)'] );
    }
    return $endpoints;
} );
add_action( 'send_headers', function () {
    header( 'X-Frame-Options: SAMEORIGIN' );
    header( 'X-Content-Type-Options: nosniff' );
    header( 'Referrer-Policy: strict-origin-when-cross-origin' );
    header( 'Permissions-Policy: camera=(), microphone=(), geolocation=()' );
} );
add_filter( 'login_errors', function () {
    return __( 'Incorrect credentials.', '[SLUG]' );
} );
```

Activate child theme:
```bash
docker compose run --rm wpcli theme activate <slug>-elementor
```

Commit: `chore: scaffold Elementor child theme with security hardening`

### Step 2: Configure Elementor settings

```bash
cd ~/clients/<slug>

# Enable Elementor's container (flexbox) mode — more modern than sections/columns
docker compose run --rm wpcli option update elementor_experiment-container active

# Set default content width (matches --container-max from design system)
docker compose run --rm wpcli option update elementor_container_width 1280

# Disable Elementor's own colour picker defaults (we control via Global Colors)
docker compose run --rm wpcli option update elementor_disable_color_schemes 1
docker compose run --rm wpcli option update elementor_disable_typography_schemes 1

# Set Google Fonts loading method (swap for performance)
docker compose run --rm wpcli option update elementor_google_font display=swap
```

Commit: `chore: Elementor settings — container mode, content width, colour/font control`

### Step 3: Wire design system into Elementor Global Colors and Fonts

Elementor stores its kit settings as post meta on a special `elementor_kit` post. We inject the validated design system palette directly:

```bash
cd ~/clients/<slug>

# Find the active kit ID
KIT_ID=$(docker compose run --rm wpcli post list --post_type=elementor_kit --post_status=publish --fields=ID --format=ids)
echo "Kit ID: $KIT_ID"
```

Then write the Global Colors as post meta (replace hex values with the WCAG-validated palette from Phase 1C):

```bash
docker compose run --rm wpcli post meta update $KIT_ID _elementor_page_settings \
'{"system_colors":[
  {"_id":"primary","title":"Primary \/ Background","color":"#HEX_BG"},
  {"_id":"secondary","title":"Surface","color":"#HEX_SURFACE"},
  {"_id":"text","title":"Text","color":"#HEX_TEXT"},
  {"_id":"accent","title":"Accent","color":"#HEX_ACCENT"},
  {"_id":"cta","title":"CTA Button","color":"#HEX_CTA"},
  {"_id":"muted","title":"Muted \/ Secondary Text","color":"#HEX_MUTED"},
  {"_id":"subtle","title":"Subtle \/ Captions","color":"#HEX_SUBTLE"},
  {"_id":"dark","title":"Dark \/ Footer","color":"#HEX_DARK"},
  {"_id":"success","title":"Success","color":"#HEX_SUCCESS"},
  {"_id":"warning","title":"Warning","color":"#HEX_WARNING"},
  {"_id":"error","title":"Error","color":"#HEX_ERROR"}
],"system_typography":[
  {"_id":"primary","title":"Display \/ Headings","typography_typography":"custom",
   "typography_font_family":"[DISPLAY_FONT]","typography_font_weight":"700"},
  {"_id":"secondary","title":"Body","typography_typography":"custom",
   "typography_font_family":"[BODY_FONT]","typography_font_weight":"400"},
  {"_id":"text","title":"Small \/ Captions","typography_typography":"custom",
   "typography_font_family":"[BODY_FONT]","typography_font_weight":"400"}
]}'
```

Add any CSS variables and global overrides to `assets/css/custom.css` in the child theme:

```css
/* Global typography and spacing tokens — referenced in Elementor widgets */
:root {
    --e-global-color-primary:   #HEX_BG;
    --e-global-color-secondary: #HEX_SURFACE;
    --e-global-color-text:      #HEX_TEXT;
    --e-global-color-accent:    #HEX_ACCENT;

    /* Spacing rhythm — use in Elementor widget padding/margin fields */
    --space-sm:  1.5rem;   /* 24px */
    --space-md:  3rem;     /* 48px */
    --space-lg:  5rem;     /* 80px */
    --space-xl:  8rem;     /* 128px */
}

/* Override Hello Elementor defaults */
body { font-family: '[BODY_FONT]', sans-serif; color: var(--e-global-color-text); }
h1, h2, h3, h4 { font-family: '[DISPLAY_FONT]', sans-serif; }

/* Button defaults */
.elementor-button { background-color: #HEX_CTA; color: #HEX_CTA_TEXT; border-radius: 6px; }
.elementor-button:hover { background-color: #HEX_CTA_HOVER; }

/* Ensure focus styles are visible (accessibility) */
:focus-visible { outline: 2px solid #HEX_ACCENT; outline-offset: 3px; }
```

Commit: `feat: design system wired into Elementor Global Colors and Fonts`

### Step 4: WordPress configuration via WP-CLI

```bash
cd ~/clients/<slug>

# Site identity
docker compose run --rm wpcli option update blogname "<title>"
docker compose run --rm wpcli option update blogdescription "<tagline>"

# Permalinks
docker compose run --rm wpcli option update permalink_structure "/%postname%/"
docker compose run --rm wpcli rewrite flush

# Create all pages from PRD with Elementor page template
for PAGE_TITLE in "Home" "About" "Services" "Contact"; do
    docker compose run --rm wpcli post create \
        --post_type=page \
        --post_status=publish \
        --post_title="$PAGE_TITLE" \
        --meta_input='{"_elementor_edit_mode":"builder","_elementor_template_type":"wp-page"}'
done

# Get the home page ID and set as front page
HOME_ID=$(docker compose run --rm wpcli post list --post_type=page --name=home --fields=ID --format=ids)
docker compose run --rm wpcli option update show_on_front page
docker compose run --rm wpcli option update page_on_front $HOME_ID

# Install SEO plugin
docker compose run --rm wpcli plugin install wordpress-seo --activate

# Set Yoast meta descriptions per page
docker compose run --rm wpcli post meta update <PAGE_ID> _yoast_wpseo_metadesc "<description>"
```

**DB export:**
```bash
docker compose exec db mysqldump -u wpuser -pwppass123 wordpress > database/seed.sql
```

Commit: `feat: WordPress configuration — pages, plugins, front page`

### Step 5: Structured data

Add JSON-LD to child theme's `functions.php` (same as custom PHP, same schema types):

```php
add_action( 'wp_head', function () {
    if ( ! is_front_page() ) return;
    $schema = [
        '@context' => 'https://schema.org',
        '@type'    => 'LocalBusiness', // per PRD
        'name'     => get_bloginfo( 'name' ),
        'url'      => home_url( '/' ),
    ];
    echo '<script type="application/ld+json">' . wp_json_encode( $schema, JSON_UNESCAPED_SLASHES ) . '</script>';
} );
```

Commit: `feat: JSON-LD structured data`

### Step 6: Build pages in Elementor editor

This step is performed in the Elementor visual editor (WP Admin → Pages → Edit with Elementor).

**For each page in the PRD:**

1. Open in Elementor editor
2. Build using **only Global Colors and Global Fonts** — never use the local colour picker
3. Follow the page structure from the PRD exactly
4. Apply content from the site crawl (Scenario 1) or brief (Scenario 2) — no placeholder copy
5. Use the content strategy formulas:
   - Hero headline: `[Outcome] for [audience] — [differentiator]`
   - Primary CTA: Verb + specific outcome
   - Trust signals above first fold break

**Widget-to-section mapping:**

| PRD section | Elementor widget(s) |
|-------------|-------------------|
| Hero | Container + Heading + Text + Button + Image |
| Services grid | Loop Grid or repeating Containers + Icon Box |
| Stats/counters | Counter widget |
| Testimonials | Testimonial Carousel or Slides |
| Team | Image Box in grid |
| CTA band | Container with background colour + Heading + Button |
| Contact form | WPForms or Contact Form 7 widget |
| Logo grid | Basic Gallery or Image widget in grid |

**Responsive checks inside Elementor:**
- Switch to Tablet (768px) and Mobile (375px) views in the editor
- Adjust padding/font size per breakpoint using Elementor's responsive controls
- Ensure all text remains readable and touch targets are ≥44px on mobile

**After each page is built:**
```bash
# Export page as Elementor template for version control
# WP Admin → Templates → Save as Template → export JSON
# Save to: ~/clients/<slug>/elementor-templates/<page-name>.json
git add elementor-templates/ && git commit -m "feat: Elementor template — [page name]"
```

### Step 7: Export and version-control all Elementor templates

```bash
mkdir -p ~/clients/<slug>/elementor-templates

# Export global kit settings (colours, fonts, global styles)
# WP Admin → Elementor → Tools → Export Kit → download and save:
# ~/clients/<slug>/elementor-templates/kit-settings.json

git add elementor-templates/
git commit -m "chore: export Elementor kit and page templates for version control"
```

**Note for deployment:** Elementor templates import via WP Admin → Templates → Import on the production server after DB import.

---

## Phase 5 — QA

### Security audit (run first)

```bash
THEME_DIR=~/clients/<slug>/wp-content/themes/<slug>-theme

# Unescaped output — must return nothing
grep -rn 'echo \$' "$THEME_DIR" --include="*.php" | grep -v 'esc_\|wp_kses\|absint\|intval\|wp_json_encode'

# Raw superglobal access — must return nothing
grep -rn '\$_GET\|\$_POST\|\$_REQUEST' "$THEME_DIR" --include="*.php" | grep -v 'wp_verify_nonce\|sanitize\|absint'

# Hardening present in functions.php
grep -c 'xmlrpc_enabled\|X-Frame-Options\|wp_generator\|author.*redirect' "$THEME_DIR/functions.php"

# console.log in JS — must return nothing
grep -rn 'console\.log' "$THEME_DIR" --include="*.js"
```

Fix all findings. The hardening block must be present in `functions.php` — add it from `coding-standards.md → WordPress Hardening` if missing.

Commit: `fix: security audit — escaping, hardening, nonces`

### Responsive audit
Scan every CSS file. Fix: hardcoded px → variables, max-width queries → min-width, overflow at 375px, touch targets < 44px. Verify all breakpoints (375, 768, 1024, 1440).

### Accessibility audit
Skip nav present and functional, keyboard navigation works throughout, visible focus styles on all interactive elements, colour contrast ≥ 4.5:1 (re-run Python validator), ARIA labels on icon buttons, correct landmarks, logical heading hierarchy (one H1 per page).

### Colour system audit

**Custom PHP:**
- Re-run Python WCAG validator on final `_variables.css`
- Count accent uses: `grep -r "var(--color-accent)" --include="*.css" | wc -l` — must be ≤ 20
- Verify no hardcoded hex values outside `_variables.css`

**Elementor:**
- Verify all widgets use Global Colors — no local colour picker overrides
- Check that no hardcoded hex values appear in `custom.css` outside `:root` token definitions
- Open Elementor editor → Style Guide to confirm Global Colors and Fonts are applied consistently

### Code quality
Remove console.log, dead code, duplicate rules. Verify all strings in `__()/_e()`, all output escaped with `esc_html()`/`esc_url()`/`esc_attr()`, `rel="noopener noreferrer"` on external links. No magic numbers in CSS.

### SEO basics
Meta descriptions set via Yoast, Open Graph tags, JSON-LD structured data, heading hierarchy correct, image alt text meaningful, permalink structure set.

Commit: `fix: QA audit — security, responsive, a11y, colour system, SEO`

---

## Phase 6 — Save & Handoff

### Update SESSION_STATE.json
Mark all steps complete, set phase to "complete".

### Generate handoff docs
- `docs/CHANGELOG.md` — all files created, design decisions, known limitations
- `docs/CLIENT_HANDOFF.md` — plain English: how to log in, edit content, what NOT to do

### Save progress
```bash
# DB export (correct method)
cd ~/clients/<slug>
docker compose exec db mysqldump -u wpuser -pwppass123 wordpress > database/seed.sql

# Git commit
git add -A
git commit -m "feat: v1.0.0 — complete build"
```

### Present handoff report

```
BUILD COMPLETE — [CLIENT_NAME]
================================
WordPress:   http://localhost:<port>
WP Admin:    http://localhost:<port>/wp-admin  (admin / admin123)
phpMyAdmin:  http://localhost:<port+1>

Theme:       <slug>-theme (classic PHP — no Elementor)
Pages:       [list]
Plugins:     [list]
Git:         main + staging + dev branches

DESIGN HIGHLIGHTS
  [2-3 bullets on key improvements vs old site / competitors]
  Palette: [n] tokens, all WCAG AA validated
  Accent:  [n] uses (≤20 budget)

NEXT STEPS
  Preview:   /wp-demo
  Save:      /wp-save
  Deploy:    /wp-package

TO CUSTOMISE (Custom PHP)
  - Upload real logo (Appearance → Customize → Site Identity)
  - Replace placeholder images with real photos
  - Update contact details (footer.php + contact page)
  - Configure CF7 email recipient (WP Admin → Contact → Mail tab)
  - Update social media URLs in footer.php
  - Set Yoast meta descriptions per page

TO CUSTOMISE (Elementor)
  - Upload real logo (Appearance → Customize → Site Identity)
  - Replace placeholder images in Elementor editor
  - Update Global Colors if brand evolves (Elementor → Site Settings → Global Colors)
  - Configure CF7/WPForms email recipient
  - Import elementor-templates/ JSON files on production after DB import
  - Set Yoast meta descriptions per page
  - Activate Elementor Pro licence on production domain (if Pro)
```
