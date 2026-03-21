# Elementor vs Custom PHP Theme — Client Decision Guide

Use this document when scoping a new project. Share it with the client, walk through
the trade-offs together, and record their decision in the PRD before build starts.

---

## The core trade-off in one sentence

**Elementor** gives the client control and independence after launch.
**Custom PHP** gives a faster, more secure, better-performing site that costs less
to maintain — but requires a developer for layout changes.

---

## Side-by-side comparison

| | Elementor | Custom PHP Theme |
|---|---|---|
| **Who controls the layout** | Client (drag-and-drop) | Developer |
| **Content editing** | Visual editor — live canvas | WordPress block editor (text/images) |
| **Build time** | Faster | Slower (more craft) |
| **Initial cost** | Lower | Higher |
| **Page load speed** | Slower — 300–500KB CSS/JS added regardless of page content | Faster — only what the page needs |
| **Core Web Vitals** | Harder to achieve green scores | Achievable with a proper build |
| **Security surface** | Larger — Elementor is a high-profile CVE target | Smaller — minimal plugin footprint |
| **Design precision** | Limited by widget constraints | Pixel-perfect, no constraints |
| **Mobile control** | 3 fixed breakpoints in Elementor settings | Full CSS control at any breakpoint |
| **Plugin risk** | Site breaks if Elementor abandoned or major version changes | No dependency — self-contained |
| **Ongoing licence cost** | Elementor Pro ~$59–99/yr | None |
| **Developer handoff** | Any Elementor-familiar developer | Requires reading the codebase |

---

## When Elementor is the right choice

- **Frequent layout changes** — adding sections, rearranging pages, landing pages — without developer involvement
- **Marketing team independence** — campaigns, A/B pages, seasonal updates in-house
- **Constrained budget** — faster build time reduces initial cost
- **Tight timeline** — need something live quickly
- **Mostly text and images** — no complex interactive components needed

---

## When Custom PHP is the right choice

- **Performance is a priority** — e-commerce conversion, SEO rankings, Core Web Vitals
- **Site is a revenue tool** — not a brochure, but active lead generation
- **Distinctive design** — not achievable within Elementor's widget system
- **Security-sensitive industry** — finance, healthcare, legal, data-handling
- **Stable site post-launch** — content changes, not layout changes
- **Long-term total cost matters** — no licence fees, no plugin update risk, no vendor lock-in

---

## Honest risks of each

### Elementor risks

- **Performance debt** — Elementor loads its full CSS/JS bundle on every page regardless
  of what's used. Core Web Vitals scores are harder to achieve. This directly affects
  Google search rankings.

- **Security CVEs** — Elementor and its add-on ecosystem are among the most exploited
  WordPress attack vectors. Keeping it updated is non-optional, ongoing maintenance work.

- **Version lock-in** — Major Elementor version upgrades have historically broken layouts.
  Every upgrade requires testing every page.

- **"You can edit it" rarely happens** — Most clients who choose Elementor for self-editing
  end up not using it. The editor is powerful but not simple for non-technical users.

- **Plugin ecosystem fragility** — Many Elementor sites depend on third-party add-on packs
  (Essential Addons, ElementsKit, etc.) — each is an additional attack surface and
  independent failure point.

### Custom PHP risks

- **Developer dependency** — Layout changes require a developer. If the relationship ends,
  a new developer needs time to read someone else's codebase.

- **Higher initial cost** — More hours to build properly.

- **No visual preview while editing** — Content is edited in the WordPress block editor,
  not a live canvas.

---

## Decision framework

```
Does the client have in-house staff who will change layouts regularly?
  YES → Elementor (communicate performance and security caveats)
  NO  → Custom PHP

Is Core Web Vitals / SEO a stated business priority?
  YES → Custom PHP
  NO  → Either

Is the industry security-sensitive (finance, health, legal)?
  YES → Custom PHP
  NO  → Either

Is the build budget constrained?
  YES → Elementor (faster build)
  NO  → Custom PHP (better long-term value)

Is the deadline under 2 weeks?
  YES → Elementor
  NO  → Either
```

---

## What stays the same regardless of choice

Whichever path is chosen, these do not change:

- Expert design system derivation (colour psychology, WCAG validation, typography)
- Security hardening (functions.php, XMLRPC, security headers, user enumeration)
- SEO setup (Yoast, structured data, meta descriptions per page)
- Mobile responsiveness
- Accessibility standards (WCAG 2.1 AA)
- Git version control and deployment packaging

The quality of design and content strategy is identical.
The difference is how the layout is assembled and who can change it after launch.

---

## Recording the decision

Once the client decides, record in PRD:

```
Build approach: [elementor / custom-php]
Reason: [client stated reason]
Client confirmed: [date]
Elementor licence: [free / pro — if pro, licence key provided: yes/no]
Post-launch editor: [client name/role who will use Elementor]
```

This decision locks the scaffold path and cannot be changed mid-build without
significant rework.
