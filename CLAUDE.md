# CLAUDE.md

## Project Overview

Landing page for companyCong ty Co phan Code Tot (Code Tot JSC). Built as a single vanilla HTML page with Tailwind CSS v4, no framework JS.

## Tech Stack

- **Build:** Vite 5 (HTML entry, no framework)
- **Styling:** Tailwind CSS v4 via `@tailwindcss/vite`
- **Animations:** Vanilla JS (~3 KiB) with IntersectionObserver
- **Icons:** Inline SVGs
- **Deployment:** Cloudflare Pages via Wrangler
- **Analytics:** Google Tag Manager (GTM-xxx), deferred until user interaction or 3.5s idle

## Key Files

- `index.html` — Full landing page: all sections, meta tags, Open Graph, GTM, canonical, structured data (JSON-LD)
- `src/index.css` — Design tokens (oklch color system), typography, base styles, animation classes
- `public/scripts/animations.js` — Vanilla JS: scroll animations, count-up, checklist loop, mobile plan selector, floating CTA
- `public/` — Static assets (logos, icons, QR code, staff photos, OG image)
- `public/_routes.json` — Cloudflare Pages routing config

## Commands

- `npm run dev` — Start dev server
- `npm run build` — Production build
- `npm run pages:deploy` — Build and deploy to Cloudflare Pages (production)
- `npm run pages:deploy:staging` — Build and deploy to staging

## Workflow

- **Commit and push code changes after every feature or bugfix.** Do not wait for the user to ask — commit proactively when changes are complete.
- Do not batch unrelated changes into a single commit.
- Use conventional-style commit messages in Vietnamese or English, focusing on what changed and why.
- All content is in Vietnamese. Keep translations accurate and natural.

## Design Conventions

- Primary color: `oklch(0.45 0.16 155)` (Deep Trust Green — emerald)
- Font: Plus Jakarta Sans (body), JetBrains Mono (metrics/code)
- Use oklch color space for all CSS custom properties
- Cards use subtle borders and micro-shadows for depth
- Conversion buttons use `--primary` with high contrast foreground (WCAG AA)
- User text selection is disabled (`select-none` on body)

## Section Order (index.html)

You should update this section based on order.

## SEO & Meta

- Canonical: `https://codetot.vn/toi-uu-website/`
- OG image: `/public/og-image.jpeg` (absolute URL in meta tag)
- Robots: `index, follow`
- JSON-LD: Organization, ProfessionalService, WebPage, Service, FAQPage, Person
- GTM deferred in `index.html` before `</body>` — loaded on interaction or 3.5s idle

## Performance

- Lighthouse mobile: Performance 100, Accessibility 100, Best Practices 100, SEO 100
- **See [`PERFORMANCE.md`](./PERFORMANCE.md)** for the full optimization guide, checklist, and anti-patterns.
- Always consult `PERFORMANCE.md` before making performance-related changes.
