---
description: "Generate a production-ready deployment package for a WordPress client"
---

# /wp-package — Deployment Package

You are a DevOps lead preparing a WordPress site for production handoff. This means security hardening, environment substitution, validation, and a clean deployable archive — not just a zip of files.

## Step 1 — Identify the client

If not specified, check `ls ~/clients/`. Required inputs:
- Client slug
- Production URL (e.g. `https://acmecorp.com`)
- Hosting platform (cPanel / VPS / WP Engine / Kinsta / Cloudways / other)

## Step 2 — Pre-flight checks

```bash
cd ~/clients/<slug>

# Check git status
git status --short
git log --oneline -3
```

If there are uncommitted changes, save first:
```bash
# DB export
docker compose exec db mysqldump -u wpuser -pwppass123 wordpress > database/seed.sql

# Commit
git add -A && git commit -m "chore: pre-deployment save"
```

Check Docker and WordPress are healthy:
```bash
docker compose ps
```

Run a quick sanity check on the theme:
```bash
docker compose run --rm wpcli theme status <slug>-theme
docker compose run --rm wpcli plugin status
```

Flag any plugins that are inactive (may need activation on production) or have known update conflicts.

## Step 3 — Security hardening audit

Before packaging, run the full security checklist. Fix any issues found — do not package a site with open vulnerabilities.

**Theme code scan:**
```bash
THEME_DIR=~/clients/<slug>/wp-content/themes/<slug>-theme

# Unescaped output — must return nothing
echo "=== Unescaped output ==="
grep -rn 'echo \$' "$THEME_DIR" --include="*.php" | grep -v 'esc_\|wp_kses\|absint\|intval\|wp_json_encode'

# Raw superglobal access — must return nothing
echo "=== Raw superglobal access ==="
grep -rn '\$_GET\|\$_POST\|\$_REQUEST' "$THEME_DIR" --include="*.php" | grep -v 'wp_verify_nonce\|sanitize\|absint'

# console.log — must return nothing
echo "=== JS debug statements ==="
grep -rn 'console\.log\|console\.warn\|console\.error' "$THEME_DIR" --include="*.js"

# innerHTML with dynamic data — flag for review
echo "=== innerHTML usage ==="
grep -rn 'innerHTML' "$THEME_DIR" --include="*.js"
```

**WordPress hardening checks:**
```bash
FUNCS=~/clients/<slug>/wp-content/themes/<slug>-theme/functions.php

grep -q 'xmlrpc_enabled' "$FUNCS" && echo "✓ XMLRPC disabled" || echo "✗ XMLRPC not disabled — ADD hardening block"
grep -q 'X-Frame-Options' "$FUNCS" && echo "✓ Security headers present" || echo "✗ Security headers missing — ADD hardening block"
grep -q 'wp_generator' "$FUNCS" && echo "✓ Version disclosure removed" || echo "✗ Version disclosure present — ADD hardening block"
grep -q 'author.*redirect\|rest_endpoints' "$FUNCS" && echo "✓ User enumeration blocked" || echo "✗ User enumeration open — ADD hardening block"
```

If any check fails, add the full hardening block from `coding-standards.md → WordPress Hardening` to `functions.php` before packaging.

**WordPress config checklist:**
- [ ] `WP_DEBUG` disabled for production (in wp-config template)
- [ ] `DISALLOW_FILE_EDIT true` in wp-config template
- [ ] `WP_AUTO_UPDATE_CORE true` in wp-config template
- [ ] Database credentials are placeholders in the package (not real credentials)
- [ ] No `.env` file included in the archive

## Step 4 — Database preparation

Export with production URL already substituted:

```bash
cd ~/clients/<slug>

# Export clean DB
docker compose exec db mysqldump -u wpuser -pwppass123 wordpress > database/seed.sql

# Substitute local URL → production URL
PROD_URL="<production-url>"  # e.g. https://acmecorp.com
WP_PORT=$(grep "^WP_PORT" .env | cut -d= -f2)
LOCAL_URL="http://localhost:${WP_PORT}"

sed -i "s|${LOCAL_URL}|${PROD_URL}|g" database/seed.sql

echo "URL substitution: ${LOCAL_URL} → ${PROD_URL}"
grep -c "${PROD_URL}" database/seed.sql
```

## Step 5 — Generate wp-config template

Create a production-ready wp-config template (credentials as placeholders):

```bash
cat > ~/clients/<slug>/deployment/wp-config-template.php << 'WPCONFIG'
<?php
// Production WordPress Configuration
// Replace all PLACEHOLDER values before deploying

define('DB_NAME',     'PLACEHOLDER_DB_NAME');
define('DB_USER',     'PLACEHOLDER_DB_USER');
define('DB_PASSWORD', 'PLACEHOLDER_DB_PASS');
define('DB_HOST',     'localhost');
define('DB_CHARSET',  'utf8mb4');
define('DB_COLLATE',  '');

// Generate fresh salts at: https://api.wordpress.org/secret-key/1.1/salt/
define('AUTH_KEY',         'PLACEHOLDER_SALT_1');
define('SECURE_AUTH_KEY',  'PLACEHOLDER_SALT_2');
define('LOGGED_IN_KEY',    'PLACEHOLDER_SALT_3');
define('NONCE_KEY',        'PLACEHOLDER_SALT_4');
define('AUTH_SALT',        'PLACEHOLDER_SALT_5');
define('SECURE_AUTH_SALT', 'PLACEHOLDER_SALT_6');
define('LOGGED_IN_SALT',   'PLACEHOLDER_SALT_7');
define('NONCE_SALT',       'PLACEHOLDER_SALT_8');

$table_prefix = 'wp_';

// Disable debug on production
define('WP_DEBUG', false);
define('WP_DEBUG_LOG', false);
define('WP_DEBUG_DISPLAY', false);

// Security hardening
define('DISALLOW_FILE_EDIT', true);   // Disable theme/plugin editor in wp-admin
define('WP_AUTO_UPDATE_CORE', true);  // Auto-patch core minor versions

if (!defined('ABSPATH')) {
    define('ABSPATH', __DIR__ . '/');
}
require_once ABSPATH . 'wp-settings.php';
WPCONFIG
```

## Step 6 — Plugin licence inventory

Check for any premium/licenced plugins that require re-activation on the new domain:

```bash
docker compose run --rm wpcli plugin list --fields=name,status,version
```

Document any that need licence keys — include in the deployment checklist.

## Step 6b — Generate .htaccess hardening file

Create `~/clients/<slug>/deployment/.htaccess-hardening` for Apache hosts. The client merges this into their root `.htaccess` after deployment:

```bash
cat > ~/clients/<slug>/deployment/.htaccess-hardening << 'HTACCESS'
# === WordPress Security Hardening ===
# Merge this into your root .htaccess (after the WordPress block)

# Disable directory browsing
Options -Indexes

# Protect wp-config.php
<Files wp-config.php>
    Order Allow,Deny
    Deny from all
</Files>

# Block XML-RPC at server level (belt and suspenders with PHP filter)
<Files xmlrpc.php>
    Order Allow,Deny
    Deny from all
</Files>

# Protect .htaccess itself
<Files .htaccess>
    Order Allow,Deny
    Deny from all
</Files>

# Block access to sensitive files
<FilesMatch "\.(log|sql|bak|swp|env)$">
    Order Allow,Deny
    Deny from all
</FilesMatch>

# Block author scan (?author=N)
RewriteEngine On
RewriteCond %{QUERY_STRING} ^author=\d
RewriteRule .* - [F]

# Security headers (add to what PHP sends — only if not set already)
<IfModule mod_headers.c>
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-Content-Type-Options "nosniff"
    Header always set Referrer-Policy "strict-origin-when-cross-origin"
    Header always set Permissions-Policy "camera=(), microphone=(), geolocation=()"
</IfModule>
HTACCESS
echo ".htaccess hardening file written"
```

## Step 7 — Build the archive

```bash
cd ~/clients/<slug>

# Determine version tag from git
VERSION=$(git describe --tags 2>/dev/null || echo "v1.0.0")
ARCHIVE_NAME="<slug>-${VERSION}.tar.gz"

mkdir -p deployment

# Build archive (exclude dev-only files)
tar -czf "deployment/${ARCHIVE_NAME}" \
  --exclude='wp-content/uploads' \
  --exclude='*.log' \
  --exclude='.git' \
  --exclude='.env' \
  --exclude='node_modules' \
  wp-content/themes/<slug>-theme \
  wp-content/plugins \
  database/seed.sql \
  deployment/wp-config-template.php

echo "Archive: deployment/${ARCHIVE_NAME} ($(du -sh deployment/${ARCHIVE_NAME} | cut -f1))"
```

If uploads are < 100MB, include them:
```bash
UPLOADS_SIZE=$(du -sm wp-content/uploads 2>/dev/null | cut -f1)
if [ "${UPLOADS_SIZE:-0}" -lt 100 ]; then
  echo "Uploads: ${UPLOADS_SIZE}MB — including in archive"
  # Re-build with uploads included (remove --exclude='wp-content/uploads')
else
  echo "Uploads: ${UPLOADS_SIZE}MB — too large, excluded. Transfer separately via FTP/SFTP."
fi
```

## Step 8 — Generate DEPLOYMENT.md

Write `~/clients/<slug>/deployment/DEPLOYMENT.md`:

```markdown
# Deployment Instructions — <CLIENT_NAME>
Generated: <date>
Version: <version>
Production URL: <production-url>

## Archive contents
- `wp-content/themes/<slug>-theme/` — custom theme
- `wp-content/plugins/` — all plugins
- `database/seed.sql` — database with production URLs pre-substituted
- `wp-config-template.php` — wp-config with credential placeholders

## Pre-deployment checklist
- [ ] PHP 8.0+ available on host
- [ ] MySQL/MariaDB 5.7+ available
- [ ] Create database and user with full privileges
- [ ] Generate fresh salts: https://api.wordpress.org/secret-key/1.1/salt/
- [ ] Have domain pointing to server (DNS propagated)

## Deployment steps

### 1. Upload files
Extract archive to WordPress root (public_html or equivalent).

### 2. Configure wp-config.php
Copy `wp-config-template.php` → `wp-config.php`
Replace all PLACEHOLDER values:
- DB_NAME, DB_USER, DB_PASSWORD with your database credentials
- All PLACEHOLDER_SALT_* with values from the generator above

### 3. Import database
Via phpMyAdmin: Import → Select `seed.sql`
Via CLI: `mysql -u <user> -p <dbname> < seed.sql`

### 4. Update URLs (if domain differs from <production-url>)
Via WP-CLI:
```
wp search-replace 'http://old-url.com' 'https://new-url.com' --all-tables
wp rewrite flush
```

### 5. Activate theme and plugins
WP Admin → Appearance → Themes → Activate <slug>-theme
WP Admin → Plugins → Activate All

### 6. Post-deployment
- [ ] Upload logo: Appearance → Customize → Site Identity
- [ ] Set admin email: Settings → General
- [ ] Configure CF7 email recipient: Contact → edit form → Mail tab
- [ ] Test contact form sends correctly
- [ ] Test all navigation links
- [ ] Run Google PageSpeed check
- [ ] Submit sitemap to Google Search Console

## Plugin licence keys needed
<list any premium plugins requiring re-activation>

## Known limitations / client action required
<list placeholder content, real images needed, etc.>
```

## Step 9 — Report

```
PACKAGE READY — [CLIENT_NAME]
================================
Archive:    deployment/<slug>-<version>.tar.gz ([size])
Database:   URL-substituted (localhost → <production-url>)
Uploads:    [included / excluded — transfer separately]
Version:    <version>

SECURITY CHECKS
  ✓ No unescaped output detected
  ✓ Debug mode disabled for production
  ✓ File editor disabled (DISALLOW_FILE_EDIT)
  ✓ wp-config uses credential placeholders

PLUGINS REQUIRING LICENCE RE-ACTIVATION
  [list or "none"]

NEXT STEPS
  Transfer:   SCP / FTP the archive to the production server
  Deploy:     Follow deployment/DEPLOYMENT.md step-by-step
  Preview:    /wp-demo — share tunnel URL with client before going live
```

## Hosting-specific notes

**cPanel** — Upload via File Manager, extract. Import SQL via phpMyAdmin. Set file permissions: dirs 755, files 644.

**VPS (SSH)** — `scp deployment/<slug>-*.tar.gz user@server:/var/www/html/` then `tar -xzf` and `mysql -u root -p < seed.sql`

**WP Engine** — SFTP upload or git push to WP Engine remote. Import DB via WP Engine backup tool or SSH.

**Kinsta** — SFTP files to `/public/`. Import DB via MyKinsta → Tools → Import. Use Kinsta's WP-CLI for search-replace.

**Cloudways** — SFTP via Cloudways dashboard credentials. Import DB via Cloudways DB Manager or phpMyAdmin.
