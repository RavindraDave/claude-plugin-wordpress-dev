---
description: "Resume an interrupted WordPress project from where it stopped"
---

# /wp-resume — Resume an Interrupted Project

## Step 1 — Identify the project

If the user specifies a client name, use it.
Otherwise, check for active projects:

```bash
ls ~/clients/
```

If only one client exists, auto-select it.
If multiple, read each `SESSION_STATE.json` and show a compact summary:

```
ACTIVE PROJECTS
================
Client          Phase       Last Updated   Docker
────────────────────────────────────────────────────
acme-corp       build       2026-03-20     unknown
beta-client     scaffold    2026-03-18     unknown
```

Ask which to resume.

## Step 2 — Read full state

```bash
cd ~/clients/<slug>

# Check Docker
docker compose ps 2>/dev/null

# Check git
git log --oneline -5
git status --short
git branch

# Read project docs
cat SESSION_STATE.json
cat CONTEXT.md 2>/dev/null | head -80
```

## Step 3 — Reconstruct understanding

From `SESSION_STATE.json`, determine:
- Which steps are `complete`
- Which step is `in_progress` (crashed mid-step)
- Which steps are `pending`

Cross-check against git log — the last commit message reveals what was actually finished. If `SESSION_STATE.json` says a step is `in_progress` but there's no commit for it, the work was lost and must be redone.

Check for partial work:
```bash
# Look for files changed since last commit
git diff --name-only HEAD
git status --short
```

If partial work exists — evaluate whether it's salvageable or should be discarded.

## Step 4 — Recover environment

**If Docker is stopped:**
```bash
cd ~/clients/<slug> && docker compose up -d
echo "Waiting for WordPress to be ready..."
sleep 5
docker compose ps
```

**If theme is not activated:**
```bash
docker compose run --rm wpcli theme status <slug>-theme
docker compose run --rm wpcli theme activate <slug>-theme
```

**If WordPress URL is wrong (e.g. from a stopped tunnel):**
```bash
WP_PORT=$(grep "^WP_PORT" .env | cut -d= -f2)
docker compose run --rm wpcli option update siteurl "http://localhost:$WP_PORT"
docker compose run --rm wpcli option update home "http://localhost:$WP_PORT"
docker compose run --rm wpcli rewrite flush
```

## Step 5 — Report status clearly

```
PROJECT STATUS — [CLIENT_NAME]
================================
Phase:       [phase]
Branch:      [branch] ([clean / X uncommitted changes])
Last commit: [hash] — [message] ([date])
Docker:      [running / stopped — now started]

COMPLETED STEPS
  ✓ intake
  ✓ scaffold
  ✓ design_tokens
  ✓ navigation
  ...

CURRENT / INTERRUPTED
  → [step_name] — [assessment: complete but not committed / partially done / needs redo]

NEXT PENDING
  • [next_step]
  • [step_after]
  ...

NOTES FROM LAST SESSION
  [notes content from SESSION_STATE.json]
```

If a step was interrupted mid-way, be specific:
> "The hero section was started but not committed. I can see `template-parts/hero.php` exists but `_hero.css` is empty. I'll complete the hero section from where it stopped."

## Step 6 — Confirm before continuing

Do NOT start building automatically. Present the status and wait for the user to confirm:
> "Ready to continue from [next_step]. Say 'go' to proceed, or give me updated instructions."

Then continue from the next pending step following the same build sequence as `/wp-start` Phase 4+.

**IMPORTANT: Always use this WP-CLI format. NEVER use `--profile cli`:**
```bash
docker compose run --rm wpcli <command>
```

**DB export (wpcli container lacks /backups mount — use db service):**
```bash
docker compose exec db mysqldump -u wpuser -pwppass123 wordpress > database/seed.sql
```
