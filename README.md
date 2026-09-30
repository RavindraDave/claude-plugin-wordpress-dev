# Claude Plugin — WordPress Development System

An end-to-end automated WordPress development plugin for [Claude Code](https://claude.ai/claude-code).
Turns Claude into a team of specialists — senior WordPress engineer, brand strategist, accessibility auditor, security engineer, and DevOps lead — operating as one.

> A free, personal open-source tool, not a service. Folder and script names use `client` to mean one site project.

## What this plugin does

- Audits an existing site and comparable sites for reference
- Derives expert colour palettes from brand guidelines or industry psychology
- Validates every palette against WCAG 2.1 AA automatically
- Builds production-quality custom PHP themes **or** Elementor sites
- Enforces security standards (OWASP, WordPress hardening) at every phase
- Generates deployment packages with `.htaccess` hardening included
- Keeps each site project in its own isolated Docker environment

---

## Installation

### Prerequisites

- [Claude Code](https://claude.ai/claude-code) installed and authenticated
- Docker + Docker Compose (for local WordPress environments)
- `cloudflared` (for shareable preview tunnels — `sudo snap install cloudflared`)
- `gh` CLI authenticated (`gh auth login`)

### Install the plugin

```bash
# Clone directly into Claude's plugin directory
git clone https://github.com/RavindraDave/claude-plugin-wordpress-dev \
    ~/.claude/plugins/wordpress-dev
```

### Set up the shared toolbox

The plugin expects a shared toolbox at `~/wordpress/` with scripts and templates.
On a fresh machine, scaffold it:

```bash
mkdir -p ~/wordpress/{scripts,templates}
mkdir -p ~/clients
echo '{}' > ~/clients/.port-registry.json
```

The toolbox scripts (`new-client.sh`, `save-client.sh`, `push-client.sh`, `deploy-package.sh`) are created during the first `/wp-start` session if not already present.

---

## Slash Commands

Invoke these inside Claude Code:

| Command | What it does |
|---------|-------------|
| `/wp-start` | Start a new project — redesign or new build, custom PHP or Elementor |
| `/wp-resume` | Resume an interrupted build from exactly where it stopped |
| `/wp-review` | Audit-only mode — analyse a site without starting a build |
| `/wp-refine` | Post-build review — score UI/UX/CX/Security, fix all issues autonomously |
| `/wp-save` | Save progress — DB export + git commit |
| `/wp-package` | Generate production-ready deployment archive with security hardening |
| `/wp-demo` | Start/stop Cloudflare Tunnel for a shareable preview |
| `/wp-status` | Show status for one or all site projects |

---

## Build Approaches

Two build paths are supported. Choose one based on the trade-offs
(see [elementor-vs-custom.md](skills/wordpress-dev/references/elementor-vs-custom.md)):

### Custom PHP Theme (default)
- Zero page builder dependency — fully hand-crafted
- Fastest page loads, best Core Web Vitals scores
- Layout changes require a developer
- Recommended for: performance-critical sites, security-sensitive industries, distinctive design requirements

### Elementor
- The site owner can edit layouts after launch with drag-and-drop
- Hello Elementor parent + child theme (security hardening still applied identically)
- Design system wired into Elementor Global Colors and Global Fonts
- Recommended for: marketing-led sites with frequent layout changes

---

## Scenarios

### Scenario 1 — Existing Site Redesign
Start from a live URL (+ optional comparable-site URLs).

Flow: Crawl → Competitor Analysis → Expert Design Direction → PRD → Scaffold → Build → QA → Package

### Scenario 2 — New Build from Scratch
Start from a written brief when there's no existing site.

Flow: Intake → Expert Design Direction → PRD → Scaffold → Build → QA → Package

---

## Expert Design System

Every build derives a production-ready design system — never guesses or uses random colours:

1. **Brand asset detection** — extracts hex values from existing site CSS or brand guidelines
2. **Industry colour psychology** — 9 industry categories mapped to emotional register, primary range, and accent strategy
3. **Full palette construction** — 16 design tokens with purpose and contrast requirements defined
4. **WCAG validation** — Python script validates all key colour pairs before build starts; failures block progress
5. **Accent usage budget** — ≤20 CSS rules using the accent colour; documented allowed vs not-allowed uses
6. **Typography science** — industry-matched display/body font pairing table

---

## Security Standards

Applied at every phase — not just packaging time:

**PHP:**
- All output escaped (`esc_html`, `esc_attr`, `esc_url`, `wp_kses_post`)
- All input sanitized (`sanitize_text_field`, `sanitize_email`, `absint`)
- All forms nonce-protected (`wp_nonce_field` + `wp_verify_nonce`)
- Capability checks on all data-mutation actions
- No raw SQL — `$wpdb->prepare()` always

**WordPress hardening (baked into every `functions.php` from day one):**
- XMLRPC disabled
- WordPress version removed from head (no fingerprinting)
- User enumeration via `?author=` blocked
- REST API user endpoint restricted to authenticated requests
- Security headers: `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy`
- Login errors obscured

**Deployment:**
- `.htaccess` hardening file generated per package
- `wp-config.php` template with `DISALLOW_FILE_EDIT` and `WP_DEBUG false`
- Pre-flight security grep scan before every package build

---

## Environment Architecture

```
~/.claude/plugins/wordpress-dev/     ← This plugin (symlinked from repo)
~/wordpress/                         ← Shared toolbox
│   ├── scripts/                     ← Bash automation
│   ├── templates/                   ← PRD, intake, audit document templates
│   └── CLAUDE.md                    ← Project-level Claude instructions
└── clients/
    ├── .port-registry.json          ← Port assignments (8082, 8084, 8086...)
    └── <client-slug>/               ← Isolated git repo per site project
        ├── docker-compose.yml       ← Own WordPress + MySQL + phpMyAdmin
        ├── SESSION_STATE.json       ← Build progress (enables /wp-resume)
        ├── PRD.md                   ← Product Requirements Document
        ├── CONTEXT.md               ← Design system quick reference
        ├── wp-content/themes/       ← Custom theme or Elementor child theme
        ├── elementor-templates/     ← Elementor page JSON exports (if Elementor build)
        ├── reports/                 ← Site audit + competitor analysis
        ├── docs/                    ← Changelog + site owner guide
        ├── database/seed.sql        ← DB snapshot (gitignored)
        └── deployment/              ← Production archive + .htaccess hardening
```

**Port scheme:** WordPress on 8082, 8084, 8086... (phpMyAdmin at port+1). Port 8080 is the shared sandbox.

---

## Plugin File Structure

```
wordpress-dev/
├── README.md                              ← This file
├── commands/                              ← Slash command skill files
│   ├── wp-start.md                        ← Full build flow (custom PHP + Elementor)
│   ├── wp-resume.md                       ← State reconstruction + recovery
│   ├── wp-review.md                       ← Audit-only mode
│   ├── wp-refine.md                       ← UI/UX/CX/Security audit + auto-fix
│   ├── wp-save.md                         ← DB export + git commit
│   ├── wp-package.md                      ← Deployment packaging + hardening
│   ├── wp-demo.md                         ← Cloudflare Tunnel preview
│   └── wp-status.md                       ← Project health overview
└── skills/wordpress-dev/
    ├── SKILL.md                           ← Auto-trigger skill definition + core expertise
    └── references/
        ├── coding-standards.md            ← PHP/CSS/JS security, quality, and reusability rules
        ├── theme-scaffold.md              ← Classic PHP theme boilerplate (secure by default)
        ├── design-tokens.md              ← CSS custom property system specification
        ├── session-management.md          ← SESSION_STATE.json schema + resume protocol
        └── elementor-vs-custom.md         ← Decision guide: trade-offs comparison
```

---

## WP-CLI — Correct Command Format

**Always use this format inside site projects. Never use `--profile cli`.**

```bash
cd ~/clients/<slug>
docker compose run --rm wpcli <command>

# Examples
docker compose run --rm wpcli option update blogname "My Site"
docker compose run --rm wpcli theme activate my-theme
docker compose run --rm wpcli rewrite flush

# DB export — wpcli container lacks /backups mount, always use db service
docker compose exec db mysqldump -u wpuser -pwppass123 wordpress > database/seed.sql
```

---

## Quality Standards (Non-Negotiable on Every Build)

| Standard | Minimum bar |
|----------|-------------|
| **Security** | All output escaped, all forms nonce-protected, WordPress hardening in every `functions.php` |
| **Accessibility** | WCAG 2.1 AA — contrast ≥4.5:1, keyboard nav, focus styles, skip link, one H1 per page |
| **Code quality** | No magic numbers, no hardcoded values, BEM CSS, guard clauses, DRY functions |
| **Reusability** | All values from CSS custom properties, repeated PHP patterns extracted into prefixed functions |
| **Performance** | `loading="lazy"` on images, `font-display: swap`, JS deferred, no render-blocking resources |

---

## Keeping This Plugin Updated

This repo is symlinked from `~/.claude/plugins/wordpress-dev/` — edits to plugin files go directly into this git repo. A PostToolUse hook auto-stages changes after each edit session.

To push updates to GitHub:

```bash
cd ~/MyCodes/Claude-Plugins/wordpress-dev
git log --oneline -5   # review what changed
git push
```

---

## Licence

MIT — use freely, adapt for your own Claude Code workflows.
