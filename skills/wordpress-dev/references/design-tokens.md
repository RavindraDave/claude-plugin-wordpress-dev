# Design Tokens Reference — CSS Custom Properties

All design values live in `assets/css/_variables.css`. Nothing is hardcoded anywhere else.

## _variables.css Structure

```css
/* ═══════════════════════════════════════════════
   TYPOGRAPHY
   ═══════════════════════════════════════════════ */
@import url('[GOOGLE_FONTS_URL]');

:root {
  --font-display : '[DISPLAY_FONT]', Georgia, serif;
  --font-body    : '[BODY_FONT]', system-ui, sans-serif;

  /* Fluid type scale — clamp(mobile-min, preferred, desktop-max) */
  --text-xs   : clamp(0.75rem,  1.5vw, 0.875rem);
  --text-sm   : clamp(0.875rem, 1.8vw, 1rem);
  --text-base : clamp(1rem,     2vw,   1.125rem);
  --text-lg   : clamp(1.125rem, 2.5vw, 1.25rem);
  --text-xl   : clamp(1.25rem,  3vw,   1.5rem);
  --text-2xl  : clamp(1.5rem,   3.5vw, 2rem);
  --text-3xl  : clamp(2rem,     4vw,   2.5rem);
  --text-4xl  : clamp(2.5rem,   5vw,   3.5rem);
  --text-5xl  : clamp(3rem,     6vw,   4.5rem);
  --text-6xl  : clamp(3.5rem,   7vw,   6rem);

  --leading-tight   : 1.1;
  --leading-snug    : 1.25;
  --leading-normal  : 1.5;
  --leading-relaxed : 1.7;

  --tracking-tight  : -0.025em;
  --tracking-normal :  0;
  --tracking-wide   :  0.05em;
  --tracking-wider  :  0.1em;
  --tracking-widest :  0.2em;

  /* ═══════════════════════════════════════════════
     COLORS — from PRD / CONTEXT.md
     ═══════════════════════════════════════════════ */
  --color-primary      : #hex;   /* headings, brand identity */
  --color-accent       : #hex;   /* CTAs, highlights, hover states */
  --color-accent-hover : #hex;   /* darker accent for button hover */
  --color-bg           : #hex;   /* page background */
  --color-surface      : #hex;   /* card bg, alternate section bg */
  --color-text         : #hex;   /* body copy */
  --color-muted        : #hex;   /* captions, secondary text */
  --color-border       : #hex;   /* dividers, card borders */
  --color-dark         : #hex;   /* footer bg, dark sections */

  /* ═══════════════════════════════════════════════
     SPACING — 4px base unit
     ═══════════════════════════════════════════════ */
  --space-1  : 0.25rem;   --space-2  : 0.5rem;    --space-3  : 0.75rem;
  --space-4  : 1rem;      --space-5  : 1.25rem;   --space-6  : 1.5rem;
  --space-8  : 2rem;      --space-10 : 2.5rem;    --space-12 : 3rem;
  --space-16 : 4rem;      --space-20 : 5rem;      --space-24 : 6rem;
  --space-32 : 8rem;

  /* ═══════════════════════════════════════════════
     LAYOUT
     ═══════════════════════════════════════════════ */
  --container-max     : 1200px;
  --container-wide    : 1400px;
  --container-narrow  :  760px;
  --container-padding : clamp(var(--space-4), 5vw, var(--space-12));
  --section-padding   : clamp(var(--space-12), 8vw, var(--space-24));
  --grid-gap          : clamp(var(--space-4), 3vw, var(--space-8));

  /* ═══════════════════════════════════════════════
     EFFECTS
     ═══════════════════════════════════════════════ */
  --shadow-sm : 0 1px 3px rgba(0,0,0,0.08), 0 1px 2px rgba(0,0,0,0.05);
  --shadow-md : 0 4px 12px rgba(0,0,0,0.10), 0 2px 6px rgba(0,0,0,0.06);
  --shadow-lg : 0 10px 40px rgba(0,0,0,0.12), 0 4px 16px rgba(0,0,0,0.08);

  --radius-sm   : 4px;    --radius-md   : 8px;
  --radius-lg   : 16px;   --radius-xl   : 24px;
  --radius-full : 9999px;

  --ease-fast : 150ms cubic-bezier(0.4, 0, 0.2, 1);
  --ease-base : 300ms cubic-bezier(0.4, 0, 0.2, 1);
  --ease-slow : 500ms cubic-bezier(0.4, 0, 0.2, 1);

  --z-base    :   1;   --z-raised  :  10;   --z-nav     : 100;
  --z-overlay : 200;   --z-modal   : 300;   --z-toast   : 400;
}
```

## _reset.css

```css
*, *::before, *::after { box-sizing: border-box; }
* { margin: 0; padding: 0; }

html { font-size: 100%; scroll-behavior: smooth; -webkit-text-size-adjust: 100%; }

body {
  font-family: var(--font-body);
  font-size: var(--text-base);
  line-height: var(--leading-normal);
  color: var(--color-text);
  background-color: var(--color-bg);
  -webkit-font-smoothing: antialiased;
}

img, video, svg { display: block; max-width: 100%; height: auto; }
input, button, textarea, select { font: inherit; }

h1, h2, h3, h4, h5, h6 {
  font-family: var(--font-display);
  line-height: var(--leading-tight);
  letter-spacing: var(--tracking-tight);
}

a { color: inherit; text-decoration: none; }
ul, ol { list-style: none; }

.skip-link {
  position: absolute; top: -100%; left: var(--space-4);
  background: var(--color-accent); color: #fff;
  padding: var(--space-2) var(--space-4); z-index: var(--z-toast);
  border-radius: var(--radius-sm);
}
.skip-link:focus { top: var(--space-2); }

:focus-visible {
  outline: 2px solid var(--color-accent);
  outline-offset: 3px;
  border-radius: var(--radius-sm);
}

@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

## _animations.css

```css
@keyframes fadeIn     { from { opacity: 0 } to { opacity: 1 } }
@keyframes slideUp    { from { opacity: 0; transform: translateY(24px) } to { opacity: 1; transform: translateY(0) } }
@keyframes slideLeft  { from { opacity: 0; transform: translateX(-32px) } to { opacity: 1; transform: translateX(0) } }
@keyframes slideRight { from { opacity: 0; transform: translateX(32px) } to { opacity: 1; transform: translateX(0) } }
@keyframes scaleIn    { from { opacity: 0; transform: scale(0.95) } to { opacity: 1; transform: scale(1) } }

.animate-fade     { animation: fadeIn    var(--ease-slow) both; }
.animate-up       { animation: slideUp   var(--ease-slow) both; }
.animate-left     { animation: slideLeft var(--ease-slow) both; }
.animate-right    { animation: slideRight var(--ease-slow) both; }
.animate-scale    { animation: scaleIn   var(--ease-slow) both; }

.delay-1 { animation-delay: 0.1s; } .delay-2 { animation-delay: 0.2s; }
.delay-3 { animation-delay: 0.3s; } .delay-4 { animation-delay: 0.4s; }
.delay-5 { animation-delay: 0.5s; } .delay-6 { animation-delay: 0.6s; }

/* Scroll-triggered: JS adds .is-visible */
.reveal { opacity: 0; transform: translateY(24px); transition: opacity var(--ease-slow), transform var(--ease-slow); }
.reveal.is-visible { opacity: 1; transform: translateY(0); }

@media (prefers-reduced-motion: reduce) {
  .animate-fade, .animate-up, .animate-left, .animate-right, .animate-scale { animation: none; opacity: 1; transform: none; }
  .reveal, .reveal.is-visible { opacity: 1; transform: none; transition: none; }
}
```
