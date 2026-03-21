# Theme Scaffold Reference — Classic PHP Theme

## Directory Structure

```
<slug>-theme/
├── style.css
├── functions.php
├── index.php
├── front-page.php
├── page.php
├── single.php
├── archive.php
├── 404.php
├── search.php
├── header.php
├── footer.php
├── template-parts/
│   ├── hero.php
│   ├── services.php
│   ├── about.php
│   ├── testimonials.php
│   ├── cta.php
│   ├── contact.php
│   └── (additional per PRD)
└── assets/
    ├── css/
    │   ├── _variables.css
    │   ├── _reset.css
    │   ├── _grid.css
    │   ├── _utilities.css
    │   ├── _animations.css
    │   ├── _nav.css
    │   ├── _hero.css
    │   ├── _sections.css
    │   └── _footer.css
    └── js/
        ├── nav.js
        └── main.js
```

## Boilerplate Files

### style.css
```css
/*
Theme Name:  [THEME_NAME]
Theme URI:   [URL]
Description: Custom theme for [CLIENT_NAME]. No page builder dependency.
Version:     1.0.0
Text Domain: [SLUG]
*/
```

### functions.php
```php
<?php
/**
 * [THEME_NAME] functions and definitions.
 *
 * Security standards applied:
 *   - All output escaped at render time
 *   - XMLRPC disabled, version disclosure removed
 *   - Security headers sent on every response
 *   - User enumeration blocked
 *   - Nonces required on all form/AJAX actions (see coding-standards.md)
 */

// ── Theme Setup ────────────────────────────────────────────
function [SLUG]_setup() {
    add_theme_support( 'title-tag' );
    add_theme_support( 'post-thumbnails' );
    add_theme_support( 'html5', [ 'search-form', 'comment-form', 'comment-list', 'gallery', 'caption' ] );
    add_theme_support( 'custom-logo', [
        'height'      => 80,
        'width'       => 250,
        'flex-height' => true,
        'flex-width'  => true,
    ] );
    register_nav_menus( [
        'primary' => __( 'Primary Menu', '[SLUG]' ),
        'footer'  => __( 'Footer Menu', '[SLUG]' ),
    ] );
}
add_action( 'after_setup_theme', '[SLUG]_setup' );

// ── Enqueue Styles & Scripts ───────────────────────────────
function [SLUG]_enqueue() {
    $v   = wp_get_theme()->get( 'Version' );
    $uri = get_template_directory_uri();

    // Google Fonts
    wp_enqueue_style( '[SLUG]-fonts', '[GOOGLE_FONTS_URL]', [], null );

    // CSS partials — dependency chain ensures correct load order
    wp_enqueue_style( '[SLUG]-variables',  $uri . '/assets/css/_variables.css',  [ '[SLUG]-fonts' ], $v );
    wp_enqueue_style( '[SLUG]-reset',      $uri . '/assets/css/_reset.css',      [ '[SLUG]-variables' ], $v );
    wp_enqueue_style( '[SLUG]-grid',       $uri . '/assets/css/_grid.css',       [ '[SLUG]-reset' ], $v );
    wp_enqueue_style( '[SLUG]-utilities',  $uri . '/assets/css/_utilities.css',  [ '[SLUG]-grid' ], $v );
    wp_enqueue_style( '[SLUG]-animations', $uri . '/assets/css/_animations.css', [ '[SLUG]-utilities' ], $v );
    wp_enqueue_style( '[SLUG]-nav',        $uri . '/assets/css/_nav.css',        [ '[SLUG]-animations' ], $v );
    wp_enqueue_style( '[SLUG]-hero',       $uri . '/assets/css/_hero.css',       [ '[SLUG]-nav' ], $v );
    wp_enqueue_style( '[SLUG]-sections',   $uri . '/assets/css/_sections.css',   [ '[SLUG]-hero' ], $v );
    wp_enqueue_style( '[SLUG]-footer-css', $uri . '/assets/css/_footer.css',     [ '[SLUG]-sections' ], $v );
    wp_enqueue_style( '[SLUG]-style',      get_stylesheet_uri(),                 [ '[SLUG]-footer-css' ], $v );

    // JS — loaded in footer, nonce localised for AJAX if needed
    wp_enqueue_script( '[SLUG]-nav-js',  $uri . '/assets/js/nav.js',  [], $v, true );
    wp_enqueue_script( '[SLUG]-main-js', $uri . '/assets/js/main.js', [ '[SLUG]-nav-js' ], $v, true );

    // Localise nonce for any AJAX actions (remove if no AJAX used)
    wp_localize_script( '[SLUG]-main-js', '[SLUG]Ajax', [
        'ajaxurl' => admin_url( 'admin-ajax.php' ),
        'nonce'   => wp_create_nonce( '[SLUG]_ajax_nonce' ),
    ] );
}
add_action( 'wp_enqueue_scripts', '[SLUG]_enqueue' );

// ── Security Hardening ─────────────────────────────────────

// Remove WordPress version number (no fingerprinting)
remove_action( 'wp_head', 'wp_generator' );
add_filter( 'the_generator', '__return_empty_string' );
add_filter( 'style_loader_src',  '[SLUG]_remove_version_query', 10, 1 );
add_filter( 'script_loader_src', '[SLUG]_remove_version_query', 10, 1 );
function [SLUG]_remove_version_query( $src ) {
    return $src ? remove_query_arg( 'ver', $src ) : $src;
}

// Disable XML-RPC (brute force and DDoS amplification vector)
add_filter( 'xmlrpc_enabled', '__return_false' );
remove_action( 'wp_head', 'rsd_link' );
remove_action( 'wp_head', 'wlwmanifest_link' );

// Remove unnecessary head noise
remove_action( 'wp_head', 'wp_shortlink_wp_head' );
remove_action( 'wp_head', 'adjacent_posts_rel_link_wp_head' );

// Block user enumeration via ?author= query string
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

// Security headers on every response
add_action( 'send_headers', function () {
    header( 'X-Frame-Options: SAMEORIGIN' );
    header( 'X-Content-Type-Options: nosniff' );
    header( 'Referrer-Policy: strict-origin-when-cross-origin' );
    header( 'Permissions-Policy: camera=(), microphone=(), geolocation=()' );
} );

// Obscure login errors (don't reveal whether email or password was wrong)
add_filter( 'login_errors', function () {
    return __( 'Incorrect credentials.', '[SLUG]' );
} );
```

### header.php
```php
<!DOCTYPE html>
<html <?php language_attributes(); ?>>
<head>
    <meta charset="<?php bloginfo( 'charset' ); ?>">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <?php wp_head(); ?>
</head>
<body <?php body_class(); ?>>
<?php wp_body_open(); ?>

<a class="skip-link screen-reader-text" href="#main"><?php _e( 'Skip to content', '[SLUG]' ); ?></a>

<header class="site-header" id="site-header" role="banner">
    <div class="container site-header__inner">
        <a href="<?php echo esc_url( home_url( '/' ) ); ?>" class="site-header__logo" rel="home"
           aria-label="<?php bloginfo( 'name' ); ?> — Home">
            <?php if ( has_custom_logo() ) : the_custom_logo();
            else : ?><span class="site-header__logo-text"><?php bloginfo( 'name' ); ?></span>
            <?php endif; ?>
        </a>

        <nav class="site-nav" id="site-nav" role="navigation"
             aria-label="<?php _e( 'Primary menu', '[SLUG]' ); ?>">
            <?php wp_nav_menu( [
                'theme_location' => 'primary',
                'container'      => false,
                'menu_class'     => 'site-nav__list',
                'fallback_cb'    => false,
            ] ); ?>
            <a href="<?php echo esc_url( get_permalink( get_page_by_path( 'contact' ) ) ); ?>"
               class="btn btn--primary site-nav__cta">
                <?php _e( 'Get in Touch', '[SLUG]' ); ?>
            </a>
        </nav>

        <button class="hamburger" id="hamburger" aria-controls="site-nav"
                aria-expanded="false" aria-label="<?php _e( 'Open menu', '[SLUG]' ); ?>">
            <span class="hamburger__bar"></span>
            <span class="hamburger__bar"></span>
            <span class="hamburger__bar"></span>
        </button>
    </div>
</header>

<!-- Mobile overlay -->
<div class="mobile-nav-overlay" id="mobile-nav" aria-hidden="true"
     role="dialog" aria-label="<?php _e( 'Mobile menu', '[SLUG]' ); ?>">
    <?php wp_nav_menu( [
        'theme_location' => 'primary',
        'container'      => false,
        'menu_class'     => 'site-nav__list',
        'fallback_cb'    => false,
    ] ); ?>
    <a href="<?php echo esc_url( get_permalink( get_page_by_path( 'contact' ) ) ); ?>"
       class="btn btn--primary site-nav__cta"><?php _e( 'Get in Touch', '[SLUG]' ); ?></a>
</div>
```

### footer.php
```php
<footer class="site-footer" id="site-footer" role="contentinfo">
    <div class="site-footer__main">
        <div class="container site-footer__grid">
            <div class="site-footer__brand">
                <a href="<?php echo esc_url( home_url( '/' ) ); ?>" class="site-footer__logo">
                    <?php if ( has_custom_logo() ) : the_custom_logo();
                    else : ?><span class="site-footer__logo-text"><?php bloginfo( 'name' ); ?></span>
                    <?php endif; ?>
                </a>
                <p class="site-footer__tagline"><?php bloginfo( 'description' ); ?></p>
                <div class="site-footer__social" role="list"
                     aria-label="<?php _e( 'Social media', '[SLUG]' ); ?>">
                    <!-- Inline SVG social icons — no Font Awesome -->
                </div>
            </div>
            <div class="site-footer__nav">
                <h3 class="site-footer__heading"><?php _e( 'Navigation', '[SLUG]' ); ?></h3>
                <?php wp_nav_menu( [ 'theme_location' => 'footer', 'container' => false,
                    'menu_class' => 'site-footer__nav-list', 'fallback_cb' => false ] ); ?>
            </div>
            <div class="site-footer__services">
                <h3 class="site-footer__heading"><?php _e( 'Services', '[SLUG]' ); ?></h3>
                <!-- Service links per PRD -->
            </div>
            <div class="site-footer__contact">
                <h3 class="site-footer__heading"><?php _e( 'Contact', '[SLUG]' ); ?></h3>
                <address class="site-footer__address">
                    <!-- Contact info per PRD -->
                </address>
            </div>
        </div>
    </div>
    <div class="site-footer__bottom">
        <div class="container site-footer__bottom-inner">
            <p>&copy; <?php echo esc_html( date( 'Y' ) ); ?>
                <a href="<?php echo esc_url( home_url( '/' ) ); ?>"><?php bloginfo( 'name' ); ?></a>.
                <?php _e( 'All rights reserved.', '[SLUG]' ); ?></p>
            <nav aria-label="<?php _e( 'Legal', '[SLUG]' ); ?>">
                <a href="#"><?php _e( 'Privacy Policy', '[SLUG]' ); ?></a>
                <a href="#"><?php _e( 'Terms of Service', '[SLUG]' ); ?></a>
            </nav>
        </div>
    </div>
</footer>
<?php wp_footer(); ?>
</body>
</html>
```

### front-page.php
```php
<?php get_header(); ?>
<main id="main" class="site-main">
    <?php get_template_part( 'template-parts/hero' ); ?>
    <?php get_template_part( 'template-parts/services' ); ?>
    <?php get_template_part( 'template-parts/about' ); ?>
    <?php get_template_part( 'template-parts/testimonials' ); ?>
    <?php get_template_part( 'template-parts/cta' ); ?>
</main>
<?php get_footer(); ?>
```

### page.php
```php
<?php get_header(); ?>
<main id="main" class="site-main">
    <?php while ( have_posts() ) : the_post(); ?>
    <article <?php post_class( 'section' ); ?>>
        <div class="container container--narrow">
            <h1 class="text-heading mb-8"><?php the_title(); ?></h1>
            <div class="page-content text-body"><?php the_content(); ?></div>
        </div>
    </article>
    <?php endwhile; ?>
</main>
<?php get_footer(); ?>
```

### 404.php
```php
<?php get_header(); ?>
<main id="main" class="site-main">
    <section class="section">
        <div class="container container--narrow text-center">
            <h1 class="text-display mb-4"><?php _e( '404', '[SLUG]' ); ?></h1>
            <p class="text-subhead text-muted mb-8"><?php _e( 'Page not found.', '[SLUG]' ); ?></p>
            <a href="<?php echo esc_url( home_url( '/' ) ); ?>" class="btn btn--primary">
                <?php _e( 'Back to Home', '[SLUG]' ); ?>
            </a>
        </div>
    </section>
</main>
<?php get_footer(); ?>
```

### assets/js/nav.js
```js
(function () {
  'use strict';
  const header = document.getElementById('site-header');
  const hamburger = document.getElementById('hamburger');
  const overlay = document.getElementById('mobile-nav');
  if (!header || !hamburger || !overlay) return;

  const focusable = overlay.querySelectorAll('a, button, [tabindex]:not([tabindex="-1"])');

  // Scroll → sticky header
  const onScroll = () => header.classList.toggle('is-scrolled', window.scrollY > 40);
  window.addEventListener('scroll', onScroll, { passive: true });
  onScroll();

  // Menu open/close
  function openMenu() {
    overlay.classList.add('is-open');
    hamburger.classList.add('is-open');
    hamburger.setAttribute('aria-expanded', 'true');
    overlay.setAttribute('aria-hidden', 'false');
    document.body.style.overflow = 'hidden';
    if (focusable.length) focusable[0].focus();
  }
  function closeMenu() {
    overlay.classList.remove('is-open');
    hamburger.classList.remove('is-open');
    hamburger.setAttribute('aria-expanded', 'false');
    overlay.setAttribute('aria-hidden', 'true');
    document.body.style.overflow = '';
    hamburger.focus();
  }

  hamburger.addEventListener('click', () =>
    overlay.classList.contains('is-open') ? closeMenu() : openMenu());
  document.addEventListener('keydown', e => {
    if (e.key === 'Escape' && overlay.classList.contains('is-open')) closeMenu();
  });
  overlay.addEventListener('click', e => { if (e.target === overlay) closeMenu(); });
  overlay.querySelectorAll('a').forEach(a => a.addEventListener('click', closeMenu));

  // Focus trap
  overlay.addEventListener('keydown', e => {
    if (e.key !== 'Tab' || !focusable.length) return;
    const first = focusable[0], last = focusable[focusable.length - 1];
    if (e.shiftKey && document.activeElement === first) { e.preventDefault(); last.focus(); }
    else if (!e.shiftKey && document.activeElement === last) { e.preventDefault(); first.focus(); }
  });
})();
```

### assets/js/main.js
```js
(function () {
  'use strict';
  // Scroll reveal — .reveal elements become .is-visible when scrolled into view
  const els = document.querySelectorAll('.reveal');
  if (!els.length) return;
  const observer = new IntersectionObserver(
    entries => entries.forEach(entry => {
      if (entry.isIntersecting) {
        entry.target.classList.add('is-visible');
        observer.unobserve(entry.target);
      }
    }),
    { threshold: 0.15, rootMargin: '0px 0px -40px 0px' }
  );
  els.forEach(el => observer.observe(el));
})();
```
