---
description: "Start or stop a Cloudflare Tunnel to expose a client site for preview"
---

# /wp-demo — Cloudflare Tunnel for Client Preview

## Identify action

If the user says "stop", "close", or "kill" → **Stop tunnel** section.
Otherwise → **Start tunnel** section.

If no client specified, check `ls ~/clients/` and auto-select if only one exists.

---

## Start tunnel

### Step 1 — Prepare environment

```bash
cd ~/clients/<slug>

# Get port from .env
WP_PORT=$(grep "^WP_PORT" .env | cut -d= -f2)
echo "WordPress port: $WP_PORT"

# Ensure Docker is running
docker compose up -d
docker compose ps
```

Wait a few seconds for WordPress to be fully ready if containers just started.

### Step 2 — Check for existing tunnel

```bash
# Kill any stale tunnel for this client
STALE_PID=$(cat /tmp/cf-tunnel-<slug>.pid 2>/dev/null)
if [ -n "$STALE_PID" ]; then
  kill $STALE_PID 2>/dev/null
  echo "Killed stale tunnel (PID $STALE_PID)"
fi
rm -f /tmp/cf-tunnel-<slug>.pid /tmp/cf-tunnel-<slug>.log
```

### Step 3 — Start tunnel

```bash
# Start cloudflared in background
cloudflared tunnel --url http://localhost:$WP_PORT > /tmp/cf-tunnel-<slug>.log 2>&1 &
CF_PID=$!
echo $CF_PID > /tmp/cf-tunnel-<slug>.pid
echo "Tunnel started (PID: $CF_PID)"
```

### Step 4 — Wait for URL

```bash
TUNNEL_URL=""
for i in $(seq 1 30); do
  TUNNEL_URL=$(grep -o 'https://[a-zA-Z0-9-]*\.trycloudflare\.com' /tmp/cf-tunnel-<slug>.log 2>/dev/null | head -1)
  [ -n "$TUNNEL_URL" ] && break
  sleep 1
done

if [ -z "$TUNNEL_URL" ]; then
  echo "ERROR: Tunnel URL not found after 30 seconds"
  tail -20 /tmp/cf-tunnel-<slug>.log
  exit 1
fi

echo "Tunnel URL: $TUNNEL_URL"
```

If no URL appears, print the last 20 lines of the log and report the error.

### Step 5 — Update WordPress URLs

**IMPORTANT: Use this exact format. NEVER use `--profile cli`.**

```bash
docker compose run --rm wpcli option update siteurl "$TUNNEL_URL"
docker compose run --rm wpcli option update home "$TUNNEL_URL"
echo "WordPress URLs updated to tunnel URL"
```

### Step 6 — Update SESSION_STATE.json

```bash
python3 -c "
import json
with open('SESSION_STATE.json') as f: d = json.load(f)
d['tunnel_active'] = True
d['tunnel_url'] = '$TUNNEL_URL'
d['tunnel_pid'] = $CF_PID
with open('SESSION_STATE.json', 'w') as f: json.dump(d, f, indent=2)
print('SESSION_STATE updated')
"
```

### Step 7 — Present the tunnel

Output this exact block:

```
TUNNEL READY — [CLIENT_NAME]
============================================
  <tunnel-url>
============================================

Share this link with the client for preview.
The URL works from any device, anywhere.

Tunnel PID: <pid>
Local port: http://localhost:<port>

TO STOP: /wp-demo stop
```

**Tip:** The tunnel URL changes each time you start a new tunnel. Share a fresh URL each preview session.

**Staging note:** If sending to a client for review, remind them the tunnel closes when you stop it. For persistent preview, use a real staging domain or a persistent Cloudflare Tunnel with a named tunnel.

---

## Stop tunnel

```bash
cd ~/clients/<slug>

# Kill the tunnel process
CF_PID=$(cat /tmp/cf-tunnel-<slug>.pid 2>/dev/null)
if [ -n "$CF_PID" ]; then
  kill $CF_PID 2>/dev/null && echo "Killed tunnel PID $CF_PID"
fi
pkill -f "cloudflared tunnel.*$WP_PORT" 2>/dev/null || true
rm -f /tmp/cf-tunnel-<slug>.pid /tmp/cf-tunnel-<slug>.log

# Restore WordPress URLs to localhost
WP_PORT=$(grep "^WP_PORT" .env | cut -d= -f2)
docker compose run --rm wpcli option update siteurl "http://localhost:$WP_PORT"
docker compose run --rm wpcli option update home "http://localhost:$WP_PORT"
echo "WordPress URLs restored to http://localhost:$WP_PORT"

# Update SESSION_STATE.json
python3 -c "
import json
with open('SESSION_STATE.json') as f: d = json.load(f)
d['tunnel_active'] = False
d.pop('tunnel_url', None)
d.pop('tunnel_pid', None)
with open('SESSION_STATE.json', 'w') as f: json.dump(d, f, indent=2)
"
```

Print confirmation:
```
TUNNEL CLOSED — [CLIENT_NAME]
WordPress URLs restored to http://localhost:<port>
WP Admin: http://localhost:<port>/wp-admin
```
