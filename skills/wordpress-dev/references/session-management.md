# Session Management Reference

## SESSION_STATE.json

Every client project has this file at its root. It enables `/wp-resume` to pick up
exactly where work stopped — even across different Claude Code sessions.

### Full Schema

```json
{
  "project": "Client Name",
  "slug": "client-slug",
  "theme": "client-slug-theme",
  "scenario": 1,
  "port": 8082,
  "phase": "build",
  "current_step": "hero",
  "completed_steps": [
    "intake", "site-audit", "competitor-analysis", "prd",
    "scaffold", "design-tokens", "grid-system", "navigation"
  ],
  "pending_steps": [
    "hero", "services", "about", "testimonials", "cta", "footer",
    "page-templates", "wp-config", "responsive-audit", "accessibility-audit",
    "seo", "final-qa", "handoff-docs", "package"
  ],
  "blockers": [],
  "notes_for_next_session": "",
  "design": {
    "fonts": {
      "display": "Font Name",
      "body": "Font Name",
      "google_url": "https://fonts.googleapis.com/css2?..."
    },
    "colors": {
      "primary": "#hex",
      "accent": "#hex",
      "bg": "#hex",
      "surface": "#hex",
      "text": "#hex",
      "muted": "#hex",
      "border": "#hex",
      "dark": "#hex"
    },
    "hero_treatment": "A",
    "animation_tone": "subtle",
    "layout_philosophy": "description"
  },
  "git": {
    "current_branch": "dev",
    "remote": "pending",
    "staging_deployed": null,
    "production_deployed": null,
    "live_version": null
  },
  "deployment": {
    "provider": "cPanel",
    "prod_url": "https://domain.com",
    "local_url": "http://localhost:8082"
  },
  "urls": {
    "existing_site": "https://old-site.com",
    "competitors": ["https://comp1.com", "https://comp2.com"]
  },
  "timestamps": {
    "created": "2026-03-20T10:00:00Z",
    "last_session": "2026-03-20T15:30:00Z"
  }
}
```

### Update Rules

1. **After every completed step:**
   - Move step name from `pending_steps` to `completed_steps`
   - Update `current_step` to the next pending step
   - Update `phase` if crossing a phase boundary
   - Do not store the commit hash (a commit cannot contain its own hash) — read it with `git log -1`
   - Set `timestamps.last_session` to now

2. **On blockers:**
   - Add description to `blockers[]`
   - Set `notes_for_next_session` with context needed to resolve

3. **On design decisions:**
   - Record in `design` object (font choices, color palette, hero treatment, etc.)
   - These are permanent — don't change mid-project

4. **Commit SESSION_STATE.json after every update:**
   ```bash
   git add SESSION_STATE.json && git commit -m "chore: update session state — [step] complete"
   ```

---

## CONTEXT.md

A condensed design reference card that never changes mid-project.
Read this at the start of every session instead of re-reading the full PRD.

### Template

```markdown
# Context — [CLIENT_NAME]

## Typography
Display: [font] | Body: [font]
Google Fonts: [url]

## Colors
--color-primary  : #hex
--color-accent   : #hex
--color-bg       : #hex
--color-surface  : #hex
--color-text     : #hex
--color-muted    : #hex
--color-border   : #hex
--color-dark     : #hex

## Spacing
Base: 4px | Scale: 4, 8, 12, 16, 20, 24, 32, 48, 64, 80, 96, 128

## Breakpoints
Mobile: 375px | Tablet: 768px | Desktop: 1024px | Wide: 1440px

## Layout
[philosophy: asymmetric vs symmetric, density, whitespace approach]

## Animation
[tone + key effects]

## Brand Personality
[one line]

## Pages (from PRD)
[ordered list with key sections]

## Plugins
[list with purpose]
```

---

## Resume Protocol (`/wp-resume`)

1. Read `SESSION_STATE.json` → report: project, phase, completed, current, next
2. Read `CONTEXT.md` → confirm design system values
3. `git status` → check branch and uncommitted files
4. `git log --oneline -5` → show recent commits
5. Check Docker: `docker compose ps` → restart if needed
6. Report status and wait for go-ahead
7. If `notes_for_next_session` has content, address it first
8. Continue from `current_step`

### Crash Recovery

If `current_step` is not in `completed_steps` (interrupted mid-step):
- Check `git log` for last commit
- Assess what was partially done
- Either complete the step or clean up and redo it
