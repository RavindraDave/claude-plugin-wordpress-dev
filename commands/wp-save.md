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

**IMPORTANT: The wpcli container lacks a `/backups` mount and its MariaDB client can't
authenticate to MySQL 8. Always dump via the db service.**

If a `/wp-demo` tunnel is active, the live DB holds the tunnel URL everywhere (siteurl,
Elementor data, Yoast). A plain dump would bake in a URL that dies with the tunnel, so
the snapshot is taken from a scratch copy with URLs switched back to localhost. The live
demo is not touched.

```bash
cd ~/clients/<slug>
docker compose up -d db

WP_PORT=$(grep "^WP_PORT" .env | cut -d= -f2)
LOCAL_URL="http://localhost:$WP_PORT"
TUNNEL_URL=$(python3 -c "import json;d=json.load(open('SESSION_STATE.json'));print(d.get('tunnel_url','') if d.get('tunnel_active') else '')")
db_sh() { docker compose exec -T db sh -c "export MYSQL_PWD=\"\$MYSQL_ROOT_PASSWORD\"; $1"; }

if [ -z "$TUNNEL_URL" ]; then
  SRC_DB='"$MYSQL_DATABASE"'
else
  SRC_DB=wp_save_tmp
  db_sh "mysql -uroot -e \"DROP DATABASE IF EXISTS $SRC_DB; CREATE DATABASE $SRC_DB; GRANT ALL ON $SRC_DB.* TO '\$MYSQL_USER'@'%';\""
  db_sh "mysqldump -uroot --single-transaction --no-tablespaces \"\$MYSQL_DATABASE\" | mysql -uroot $SRC_DB"
  # Plain + JSON-escaped (Elementor) forms; GUIDs untouched.
  for PAIR in "$TUNNEL_URL|$LOCAL_URL" "${TUNNEL_URL//\//\\/}|${LOCAL_URL//\//\\/}"; do
    docker compose run --rm -T -e WORDPRESS_DB_NAME=$SRC_DB wpcli search-replace \
      "${PAIR%%|*}" "${PAIR#*|}" --all-tables --precise --skip-columns=guid --format=count
  done
fi

# Temp file first: a failed dump never truncates the last good snapshot.
db_sh "mysqldump -uroot --single-transaction --no-tablespaces $SRC_DB" > database/seed.sql.tmp \
  && mv database/seed.sql.tmp database/seed.sql
[ -n "$TUNNEL_URL" ] && db_sh "mysql -uroot -e 'DROP DATABASE IF EXISTS $SRC_DB'"

echo "DB exported: $(wc -c < database/seed.sql | awk '{printf "%.0f KB", $1/1024}')"
grep -o "'siteurl','[^']*'" database/seed.sql   # must show $LOCAL_URL
```

If the dump fails, stop and report — do not commit.

## Step 3 — Update SESSION_STATE.json

Do this **before** committing, so state and code land in one commit. Don't store the
commit hash in the file: a commit can't contain its own hash (amending to add it creates a
new hash). `git log -1` is the source of truth.

```bash
cd ~/clients/<slug>
python3 -c "
import json, datetime
with open('SESSION_STATE.json') as f: d = json.load(f)
d['last_updated'] = datetime.date.today().isoformat()
d.setdefault('timestamps', {})['last_session'] = datetime.datetime.now().isoformat(timespec='seconds')
d.get('git', {}).pop('last_commit', None)
with open('SESSION_STATE.json', 'w') as f: json.dump(d, f, indent=2)
print('SESSION_STATE.json updated')
"
```

## Step 4 — Git commit

Determine commit message:
- If the user provided one → use it
- If not → inspect recent changes and derive a descriptive message:

```bash
cd ~/clients/<slug>
git status --short
git diff --stat
```

Commit:
```bash
cd ~/clients/<slug>
git add -A
git commit -m "<message>"
git log --oneline -1   # hash for the report
```

Example auto-generated messages:
- `feat: hero and services sections complete`
- `fix: QA pass — responsive, a11y, contrast`
- `refine: UI/UX pass — score 7.2 → 8.6/10`
- `docs: client handoff documents`

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
