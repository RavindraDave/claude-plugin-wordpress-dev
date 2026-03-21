# Coding Standards Reference

Security, code quality, and reusability are non-negotiable at every layer.
Apply these rules during build (wp-start), refine (wp-refine), and packaging (wp-package).

---

## PHP — Security First

### 1. Output Escaping (XSS Prevention)

Every value echoed to HTML must be escaped. No exceptions.

```php
esc_html( $text )           // Plain text — strips all HTML
esc_attr( $value )          // HTML attributes (class, id, data-*, aria-*)
esc_url( $url )             // href, src, action attributes
esc_js( $value )            // Values inside inline <script> (avoid inline JS)
wp_kses_post( $html )       // Rich HTML (post content, allow safe tags only)
absint( $id )               // Integer IDs — cast and sanitize in one step
intval( $n )                // Integer values
number_format_i18n( $n )    // Formatted numbers for display

// Translated + escaped (use instead of _e() / __())
esc_html_e( 'String', 'theme-slug' );          // echo translated + escaped
esc_attr_e( 'String', 'theme-slug' );          // echo for attribute context
$str = esc_html__( 'String', 'theme-slug' );   // return translated + escaped
```

**Never do this:**
```php
echo $user_data;                    // Raw echo of any variable
echo $_GET['value'];                // Direct superglobal output
echo get_post_meta( $id, $key, true ); // Unescaped meta output
```

---

### 2. Input Sanitization (Second-Order Injection Prevention)

Sanitize on the way IN. Escape on the way OUT. Both are required.

```php
// Text inputs
sanitize_text_field( $_POST['name'] )      // Single-line text — strips tags, extra whitespace
sanitize_textarea_field( $_POST['msg'] )   // Multiline text
sanitize_email( $_POST['email'] )          // Email addresses
sanitize_url( $_POST['website'] )          // URLs from user input
sanitize_key( $_POST['type'] )             // Keys, slugs — lowercase, alphanumeric + hyphens
sanitize_html_class( $_POST['class'] )     // CSS class names
absint( $_POST['id'] )                     // Integer IDs

// Arrays of text
array_map( 'sanitize_text_field', $_POST['items'] )

// File names (for any upload handling)
sanitize_file_name( $filename )
```

**Pattern — sanitize then store, escape then output:**
```php
// Saving to DB
$name  = sanitize_text_field( $_POST['name'] ?? '' );
$email = sanitize_email( $_POST['email'] ?? '' );
$msg   = sanitize_textarea_field( $_POST['message'] ?? '' );
update_post_meta( $post_id, '_contact_name', $name );

// Reading from DB and outputting
$name = get_post_meta( $post_id, '_contact_name', true );
echo '<p>' . esc_html( $name ) . '</p>';
```

---

### 3. Nonce Verification (CSRF Protection)

Any form or AJAX action that changes data must use nonces. No exceptions.

```php
// ── In the form (PHP template) ─────────────────────────────
<form method="post" action="<?php echo esc_url( admin_url( 'admin-post.php' ) ); ?>">
    <?php wp_nonce_field( 'mytheme_contact_action', 'mytheme_contact_nonce' ); ?>
    <input type="hidden" name="action" value="mytheme_contact">
    <!-- form fields -->
</form>

// ── In the handler (functions.php) ─────────────────────────
function mytheme_handle_contact() {
    // 1. Verify nonce first — die() if invalid
    if ( ! isset( $_POST['mytheme_contact_nonce'] ) ||
         ! wp_verify_nonce( $_POST['mytheme_contact_nonce'], 'mytheme_contact_action' ) ) {
        wp_die( __( 'Security check failed.', 'theme-slug' ), 403 );
    }

    // 2. Check capability if this is an admin action
    // if ( ! current_user_can( 'manage_options' ) ) { wp_die( 'Unauthorized', 403 ); }

    // 3. Sanitize inputs
    $name  = sanitize_text_field( $_POST['name'] ?? '' );
    $email = sanitize_email( $_POST['email'] ?? '' );

    // 4. Process...

    // 5. Redirect (PRG pattern — prevents double-submit)
    wp_safe_redirect( add_query_arg( 'sent', '1', wp_get_referer() ) );
    exit;
}
add_action( 'admin_post_nopriv_mytheme_contact', 'mytheme_handle_contact' );
add_action( 'admin_post_mytheme_contact', 'mytheme_handle_contact' );

// ── AJAX nonce pattern ──────────────────────────────────────
// Localise nonce to JS (in functions.php enqueue):
wp_localize_script( '[SLUG]-main-js', 'mythemeAjax', [
    'ajaxurl' => admin_url( 'admin-ajax.php' ),
    'nonce'   => wp_create_nonce( 'mytheme_ajax_nonce' ),
] );

// In JS:
// fetch(mythemeAjax.ajaxurl, {
//   method: 'POST',
//   body: new URLSearchParams({ action: 'my_action', nonce: mythemeAjax.nonce, data: value })
// })

// In PHP handler:
function mytheme_ajax_handler() {
    check_ajax_referer( 'mytheme_ajax_nonce', 'nonce' ); // dies if invalid
    // ... process
    wp_send_json_success( $data );
}
add_action( 'wp_ajax_my_action', 'mytheme_ajax_handler' );
add_action( 'wp_ajax_nopriv_my_action', 'mytheme_ajax_handler' ); // if public
```

---

### 4. Capability Checks (Privilege Escalation Prevention)

Any action that modifies data or accesses sensitive information requires a capability check.

```php
// Admin-only actions
if ( ! current_user_can( 'manage_options' ) ) {
    wp_die( __( 'You do not have permission.', 'theme-slug' ), 403 );
}

// Editor-level actions
if ( ! current_user_can( 'edit_posts' ) ) {
    wp_die( __( 'You do not have permission.', 'theme-slug' ), 403 );
}

// Checking ownership before editing
if ( ! current_user_can( 'edit_post', $post_id ) ) {
    wp_die( __( 'You do not have permission.', 'theme-slug' ), 403 );
}
```

**Common capabilities:** `manage_options` (admin), `edit_posts` (editor+), `publish_posts`, `upload_files`, `read` (subscriber+)

---

### 5. Database Queries (SQL Injection Prevention)

Never concatenate user input into SQL. Use `$wpdb->prepare()` always.

```php
global $wpdb;

// Correct — prepared statement
$results = $wpdb->get_results(
    $wpdb->prepare(
        "SELECT * FROM {$wpdb->posts} WHERE post_author = %d AND post_status = %s",
        absint( $user_id ),
        sanitize_key( $status )
    )
);

// Also correct — WordPress query API (preferred over raw SQL)
$query = new WP_Query( [
    'post_type'      => 'page',
    'author'         => absint( $user_id ),
    'post_status'    => 'publish',
    'posts_per_page' => 10,
] );

// NEVER do this:
$results = $wpdb->get_results( "SELECT * FROM {$wpdb->posts} WHERE ID = " . $_GET['id'] );
```

---

### 6. File Upload Security

If the site accepts file uploads (beyond WordPress media library):

```php
// Allowed MIME types (whitelist approach)
$allowed_types = [ 'image/jpeg', 'image/png', 'image/webp', 'application/pdf' ];

function mytheme_handle_upload( $file ) {
    // 1. Check file type by content, not extension
    $finfo = finfo_open( FILEINFO_MIME_TYPE );
    $mime  = finfo_file( $finfo, $file['tmp_name'] );
    finfo_close( $finfo );

    if ( ! in_array( $mime, $allowed_types, true ) ) {
        return new WP_Error( 'invalid_type', __( 'File type not allowed.', 'theme-slug' ) );
    }

    // 2. Sanitize filename
    $file['name'] = sanitize_file_name( $file['name'] );

    // 3. Use WordPress upload handler
    return wp_handle_upload( $file, [ 'test_form' => false ] );
}
```

---

### 7. Theme Functions — Quality & Reusability

**Prefix everything** — all functions, hooks, globals, constants:
```php
// Good — namespaced with theme slug
function mytheme_get_services( $limit = 6 ) { }
function mytheme_render_card( $args = [] ) { }
define( 'MYTHEME_VERSION', '1.0.0' );

// Bad — collides with anything
function get_services() { }
function render_card() { }
```

**DRY — extract repeated patterns into reusable functions:**
```php
// Instead of repeating the same card markup in 4 templates:
function mytheme_render_service_card( $args = [] ) {
    $defaults = [
        'title'       => '',
        'description' => '',
        'icon'        => '',
        'link'        => '',
        'link_text'   => __( 'Learn more', 'theme-slug' ),
    ];
    $args = wp_parse_args( $args, $defaults );

    // Escape at render time
    ?>
    <article class="service-card">
        <div class="service-card__icon" aria-hidden="true"><?php echo $args['icon']; // SVG — already safe ?></div>
        <h3 class="service-card__title"><?php echo esc_html( $args['title'] ); ?></h3>
        <p class="service-card__desc"><?php echo esc_html( $args['description'] ); ?></p>
        <?php if ( $args['link'] ) : ?>
        <a href="<?php echo esc_url( $args['link'] ); ?>" class="btn btn--outline">
            <?php echo esc_html( $args['link_text'] ); ?>
        </a>
        <?php endif; ?>
    </article>
    <?php
}

// Then call with data:
mytheme_render_service_card( [
    'title'       => __( 'Data Centre Fit-Out', 'theme-slug' ),
    'description' => __( 'End-to-end solutions for Tier 1–4 facilities.', 'theme-slug' ),
    'link'        => esc_url( get_permalink( 18 ) ),
] );
```

**Guard clauses — fail fast at the top:**
```php
function mytheme_render_hero() {
    $post_id = get_the_ID();
    if ( ! $post_id ) return;

    $title = get_the_title();
    if ( empty( $title ) ) return;

    // ... render
}
```

**Use WordPress APIs, not custom implementations:**
```php
// Correct
home_url( '/' )
get_template_directory_uri()
get_permalink( $page_id )
wp_get_attachment_image_url( $id, 'full' )
the_content()           // Handles shortcodes, embeds, filters
get_the_excerpt()       // Proper excerpt with length filters

// Wrong
echo 'http://mysite.com/';
echo '/wp-content/themes/my-theme/assets/';
```

---

### 8. WordPress Hardening (add to functions.php)

Include this hardening block in every theme's `functions.php`:

```php
// ── Security Hardening ──────────────────────────────────────

// Remove WordPress version number from head and RSS
remove_action( 'wp_head', 'wp_generator' );
add_filter( 'the_generator', '__return_empty_string' );

// Disable XML-RPC (attack vector for brute force and DDoS amplification)
add_filter( 'xmlrpc_enabled', '__return_false' );
remove_action( 'wp_head', 'rsd_link' );
remove_action( 'wp_head', 'wlwmanifest_link' );

// Remove unnecessary head links
remove_action( 'wp_head', 'wp_shortlink_wp_head' );
remove_action( 'wp_head', 'adjacent_posts_rel_link_wp_head' );

// Prevent user enumeration via author query string (?author=1)
add_action( 'init', function () {
    if ( ! is_admin() && isset( $_GET['author'] ) ) {
        wp_safe_redirect( home_url( '/' ), 301 );
        exit;
    }
} );

// Disable REST API user enumeration for unauthenticated requests
add_filter( 'rest_endpoints', function ( $endpoints ) {
    if ( ! is_user_logged_in() ) {
        unset( $endpoints['/wp/v2/users'] );
        unset( $endpoints['/wp/v2/users/(?P<id>[\d]+)'] );
    }
    return $endpoints;
} );

// Add security headers
add_action( 'send_headers', function () {
    header( 'X-Frame-Options: SAMEORIGIN' );
    header( 'X-Content-Type-Options: nosniff' );
    header( 'Referrer-Policy: strict-origin-when-cross-origin' );
    header( 'Permissions-Policy: camera=(), microphone=(), geolocation=()' );
    // Note: Content-Security-Policy should be added once the full asset list is known
    // to avoid blocking legitimate scripts/styles
} );

// Limit login attempts message (avoids revealing which field is wrong)
add_filter( 'login_errors', function () {
    return __( 'Incorrect credentials.', 'theme-slug' );
} );
```

---

## CSS — Quality Standards

### BEM Naming
```css
/* Block */
.service-card { }

/* Element */
.service-card__title { }
.service-card__icon { }

/* Modifier */
.service-card--featured { }
.btn--primary { }
.section--dark { }
```

### Rules
- ALL values reference CSS custom properties — no hardcoded hex, px font sizes, magic numbers
- Mobile-first: base = 375px. `min-width` queries only — never `max-width`
- `clamp()` for all typography — no media query font-size overrides
- No `!important` except single-purpose utility overrides (`.sr-only`, `.hidden`)
- No vendor prefixes manually — use what browsers support natively
- No ID selectors for styling — specificity stays low
- No nesting deeper than 3 levels
- No inline styles in PHP templates — classes only
- `min-height` over fixed `height` for flexible containers
- `aspect-ratio` for media elements instead of padding-hack

### Reusability — Design Tokens Are Law
Every value in every CSS file must reference a token from `_variables.css`:

```css
/* Correct */
.section { padding-block: var(--section-py); background: var(--color-surface); }
.card    { border-radius: var(--radius-md); box-shadow: var(--shadow); }
.heading { font-family: var(--font-display); font-size: var(--text-3xl); }

/* Wrong — hardcoded values */
.section { padding: 80px 0; background: #131C2E; }
.card    { border-radius: 12px; box-shadow: 0 8px 32px rgba(0,0,0,0.6); }
```

---

## JavaScript — Security + Quality

### Security Rules
- Never use `innerHTML` with user-supplied data — use `textContent` or `createElement`
- Never use `eval()` or `new Function()`
- Validate and sanitise on the PHP side — JS validation is UX only, not security
- No sensitive data in JS (API keys, credentials — localise only public data)
- `rel="noopener noreferrer"` on any dynamically created `target="_blank"` links

```js
// Correct — safe DOM manipulation
const el = document.createElement('p');
el.textContent = userInput;  // text, not HTML

// Wrong — XSS if userInput contains script tags
el.innerHTML = userInput;
```

### Quality Rules
- No `console.log()` in committed code
- No inline `<script>` blocks in PHP templates — enqueue via `wp_enqueue_script()`
- No `document.write()`, no `eval()`
- No global variables — use IIFE or module pattern
- Guard clauses at the top — return early if elements missing
- `{ passive: true }` on scroll/touch/wheel event listeners
- `IntersectionObserver` for scroll effects — never raw `scroll` event polling
- `defer` or `in_footer: true` for all scripts

### AJAX Pattern (secure)
```js
// wp_localize_script provides ajaxurl and nonce — never hardcode either
(function () {
  'use strict';

  const form = document.getElementById('contact-form');
  if (!form) return;

  form.addEventListener('submit', async function (e) {
    e.preventDefault();

    const body = new URLSearchParams({
      action: 'mytheme_contact',
      nonce:  mythemeAjax.nonce,          // from wp_localize_script
      name:   form.querySelector('[name="name"]').value,
      email:  form.querySelector('[name="email"]').value,
    });

    try {
      const res  = await fetch(mythemeAjax.ajaxurl, { method: 'POST', body });
      const data = await res.json();
      if (data.success) {
        // Show success — use textContent, not innerHTML
        document.getElementById('form-status').textContent = data.data.message;
      }
    } catch (err) {
      document.getElementById('form-status').textContent = 'An error occurred. Please try again.';
    }
  });
})();
```

---

## Security Audit Checklist

Run this mentally (and with grep) before every commit and before packaging.

### PHP
- [ ] Every `echo` uses an escape function (`esc_html`, `esc_attr`, `esc_url`, `wp_kses_post`)
- [ ] Every form has `wp_nonce_field()` and the handler calls `wp_verify_nonce()`
- [ ] Every admin/data-mutation action has `current_user_can()` check
- [ ] No raw `$_GET`, `$_POST`, `$_REQUEST` used without sanitization
- [ ] No direct SQL without `$wpdb->prepare()`
- [ ] No `var_dump()`, `print_r()`, `die()`, `exit()` without reason in committed code
- [ ] No hardcoded credentials, API keys, or paths

### WordPress
- [ ] `wp_generator` removed from head (no version disclosure)
- [ ] XMLRPC disabled
- [ ] User enumeration via `?author=` blocked
- [ ] Security headers sent (`X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`)
- [ ] `DISALLOW_FILE_EDIT` set in wp-config for production
- [ ] `WP_DEBUG` disabled for production

### JavaScript
- [ ] No `console.log()` remaining
- [ ] No `innerHTML` with dynamic/user data
- [ ] No `eval()` or `new Function()`
- [ ] Nonce passed via `wp_localize_script()`, not hardcoded
- [ ] No sensitive data exposed in JS globals

### CSS
- [ ] No hardcoded hex values outside `_variables.css`
- [ ] No `!important` except documented utility classes
- [ ] Accent uses ≤ 20 CSS rules

---

## Code Review Quick-Reference

Before committing any PHP file, grep-check:

```bash
# Unescaped echoes (flag any echo without esc_* or wp_kses_post)
grep -n 'echo \$' <file>.php | grep -v 'esc_\|wp_kses\|absint\|intval'

# Raw superglobal use without sanitize
grep -n '\$_GET\|\$_POST\|\$_REQUEST\|\$_SERVER' <file>.php | grep -v 'sanitize\|absint\|wp_verify_nonce\|isset'

# Direct SQL
grep -n 'mysql_query\|mysqli_query\|\$wpdb->query(' <file>.php | grep -v 'prepare'

# Debug leftovers
grep -n 'var_dump\|print_r\|console\.log\|die(\|exit(' <file>.php
```

---

## Git Conventions

### Commit Messages — Conventional Commits
```
feat: add services section with animated cards
fix: responsive audit — fix overflow at 375px
fix: security — add nonce verification to contact handler
style: adjust hero spacing and CTA button sizing
refactor: extract card render into reusable function
docs: add CHANGELOG and client handoff guide
chore: update SESSION_STATE after navigation step
perf: optimize hero image loading with lazy/eager
```

### What to Commit
- Theme files (PHP, CSS, JS)
- Plugin customizations
- `SESSION_STATE.json`, `CONTEXT.md`, `PRD.md`
- `.gitignore`, `.env.example`, `docker-compose.yml`

### What NOT to Commit
- `.env` (real credentials)
- `wp-content/uploads/` (binary media)
- `database/*.sql` (DB snapshots)
- `node_modules/`, `vendor/`
- `wp-config.php` (if contains real credentials)

### Branching
- `main` — production-ready, protected
- `staging` — client review
- `dev` — active development (always work here)
- `hotfix/<name>` — urgent production fixes only
