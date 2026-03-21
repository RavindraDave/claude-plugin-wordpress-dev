---
description: "Save WordPress client progress — DB export + git commit"
---

# /wp-save — Save Client Progress

## Step 1 — Identify the client

If not specified, check `ls ~/clients/`. Auto-select if only one exists.

```bash
ls ~/clients/
cat ~/clients/<slug>/SESSION_STATE.json | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('client_name',''), '|', d.get('phase',''), '|', d.get('last_updated',''))"
```

## Step 2 — DB Export

**IMPORTANT: The wpcli container lacks a `/backups` mount. Always export via the db service directly.**

```bash
cd ~/clients/<slug>

# Ensure DB container is running
docker compose up -d db

# Export
docker compose exec db mysqldump -u wpuser -pwppass123 wordpress > database/seed.sql
echo "DB exported: $(wc -c < database/seed.sql | awk '{printf "%.0f KB", $1/1024}')"
```

If `docker compose exec db` fails (db container stopped):
```bash
docker compose up -d
sleep 3
docker compose exec db mysqldump -u wpuser -pwppass123 wordpress > database/seed.sql
```

## Step 3 — Git commit

Determine commit message:
- If the user provided one → use it
- If not → inspect recent changes and derive a descriptive message:

```bash
cd ~/clients/<slug>
git status --short
git diff --cached --stat
```

Commit:
```bash
cd ~/clients/<slug>
git add -A
git commit -m "<message>"
```

Example auto-generated messages:
- `feat: hero and services sections complete`
- `fix: QA pass — responsive, a11y, contrast`
- `refine: UI/UX pass — score 7.2 → 8.6/10`
- `docs: client handoff documents`

## Step 4 — Update SESSION_STATE.json

```bash
cd ~/clients/<slug>
COMMIT_HASH=$(git rev-parse --short HEAD)
python3 -c "
import json, datetime
with open('SESSION_STATE.json') as f: d = json.load(f)
d['last_updated'] = datetime.date.today().isoformat()
d.setdefault('git', {})['last_commit'] = '$COMMIT_HASH'
d.setdefault('timestamps', {})['last_session'] = datetime.datetime.now().isoformat()
with open('SESSION_STATE.json', 'w') as f: json.dump(d, f, indent=2)
print('SESSION_STATE.json updated')
"
git add SESSION_STATE.json
git commit --amend --no-edit 2>/dev/null || git commit -m "chore: update session state"
```

## Step 5 — Report

```
SAVED — [CLIENT_NAME]
=======================
DB:      database/seed.sql ([size])
Commit:  [hash] — [message]
Branch:  [branch]

NEXT STEPS
  Preview:    /wp-demo
  Push:       cd ~/wordpress && ./scripts/push-client.sh <slug>
  Deploy:     /wp-package
```
