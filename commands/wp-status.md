---
description: "Show current project status for one or all WordPress clients"
---

# /wp-status — Project Status

## If a client name is given

```bash
cd ~/clients/<slug>
cat SESSION_STATE.json
docker compose ps 2>/dev/null
git log --oneline -3 2>/dev/null
git status --short 2>/dev/null
```

Health checks:
```bash
# Is WordPress responding?
WP_PORT=$(grep "^WP_PORT" .env 2>/dev/null | cut -d= -f2)
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" "http://localhost:${WP_PORT}/" 2>/dev/null)
echo "HTTP status: $HTTP_STATUS"

# Active theme?
docker compose run --rm wpcli theme status 2>/dev/null | grep -E "Active|Status" | head -3
```

Present:

```
[CLIENT_NAME] — [phase]
========================
URL:        http://localhost:[port]  (HTTP [status])
WP Admin:   http://localhost:[port]/wp-admin  (admin / admin123)
phpMyAdmin: http://localhost:[port+1]

Docker:     [running / stopped]
Branch:     [branch]  ([clean / X uncommitted changes])
Last commit: [hash] — [message]

PROGRESS
  Completed:  [n] / [total] steps
  Current:    [current_step or "all complete"]
  Pending:    [list of pending steps, or "none"]

NOTES
  [notes from SESSION_STATE.json]

SCORES (if refined)
  Before: [score]/10 → After: [score]/10
```

## If no client specified (or `--all`)

```bash
ls ~/clients/
cat ~/clients/.port-registry.json 2>/dev/null
```

For each client directory:
```bash
for slug in ~/clients/*/; do
  slug=$(basename $slug)
  [ -f ~/clients/$slug/SESSION_STATE.json ] || continue
  PORT=$(python3 -c "import json; d=json.load(open('$HOME/clients/$slug/SESSION_STATE.json')); print(d.get('ports',{}).get('wp','?'))" 2>/dev/null)
  PHASE=$(python3 -c "import json; d=json.load(open('$HOME/clients/$slug/SESSION_STATE.json')); print(d.get('phase','?'))" 2>/dev/null)
  HTTP=$(curl -s -o /dev/null -w "%{http_code}" "http://localhost:${PORT}/" 2>/dev/null || echo "???")
  DOCKER=$(cd ~/clients/$slug && docker compose ps --quiet 2>/dev/null | wc -l | tr -d ' ')
  [ "$DOCKER" -gt 0 ] && DOCKER_STATUS="running" || DOCKER_STATUS="stopped"
  BRANCH=$(cd ~/clients/$slug && git branch --show-current 2>/dev/null || echo "?")
  echo "$slug | $PORT | $PHASE | HTTP $HTTP | $DOCKER_STATUS | $BRANCH"
done
```

Display as a table:

```
WORDPRESS PROJECTS
==================
Client          Port    Phase       HTTP    Docker    Branch
─────────────────────────────────────────────────────────────────
acme-corp       8082    complete    200     running   dev
beta-client     8084    build       000     stopped   dev
```

Also show:
```
PORT REGISTRY
  Next available: [next even port after highest registered]

QUICK COMMANDS
  /wp-resume <client>   — continue a build
  /wp-demo <client>     — start Cloudflare tunnel
  /wp-save <client>     — save progress
  /wp-package <client>  — package for deployment
```
