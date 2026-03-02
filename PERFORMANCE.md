# Performance Optimization Guide

Reusable best-practice checklist for static HTML + Vite + Tailwind CSS v4 landing pages, based on PageSpeed Insights findings. Always consult this guide before making performance-related changes.

> **Current stack:** Static HTML + vanilla JS. Sections marked *(React reference)* document patterns from the previous React build — kept for reference if the stack changes or for other projects.

---

## 1. CSS Loading Strategy

**Current approach:** CSS is inlined into HTML at build time via Vite plugin (`inlineCssPlugin` in `vite.config.ts`). This eliminates render-blocking stylesheet requests entirely.

**If CSS is NOT inlined**, keep it as a normal render-blocking `<link rel="stylesheet">`:
```html
<!-- GOOD: small enough to block without hurting LCP -->
<link rel="stylesheet" crossorigin href="/assets/index-*.css">
```

**Anti-patterns — DO NOT:**
```html
<!-- BAD: causes FOUC — page renders unstyled, then CSS applies with a flash -->
<link rel="preload" as="style" href="/assets/index-*.css" onload="this.rel='stylesheet'">
```

## 2. Optimize Google Fonts Loading

**Problem:** `@import url()` in CSS for Google Fonts is render-blocking.

**Fix:**
1. Remove `@import url("https://fonts.googleapis.com/...")` from CSS.
2. Add to `<head>` in `index.html`:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="preload" as="style" fetchpriority="high"
      href="https://fonts.googleapis.com/css2?family=..."
      onload="this.rel='stylesheet'">
<noscript>
  <link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=...">
</noscript>
```

## 3. Defer Non-Critical JS

**Current approach:** `animations.js` is loaded with `defer` attribute — it downloads in parallel with HTML parsing and executes after DOM is ready. No critical chain.

**If using `type="module"` entry scripts** *(React reference)*: use a Vite plugin to defer via `requestAnimationFrame`:

```ts
// vite.config.ts
function deferEntryScriptPlugin(): Plugin {
  return {
    name: 'defer-entry-script',
    apply: 'build',
    enforce: 'post',
    transformIndexHtml: {
      order: 'post',
      handler(html) {
        return html.replace(
          /<script\s+type="module"[^>]*\ssrc="([^"]+)"[^>]*><\/script>/,
          (_match, src) =>
            `<script>requestAnimationFrame(function(){` +
            `var s=document.createElement('script');` +
            `s.type='module';s.src='${src}';` +
            `document.body.appendChild(s);})</script>`
        );
      },
    },
  };
}
```

## 4. Minimize JavaScript

**Current approach:** No framework JS. Single vanilla JS file (`animations.js`, ~1.5 KiB gzip) handles all interactions. React was removed entirely — saving ~120 KiB gzip.

**Principle:** Audit dependencies regularly. For static landing pages, most npm libraries are unnecessary. Vanilla JS + CSS can handle scroll animations, text swaps, and UI interactions.

## 5. Defer Analytics (GTM / gtag)

**Problem:** GTM and gtag scripts (~246 KiB) load synchronously in `<head>`, blocking page render.

**Fix:** Defer until user interaction or browser idle. Use `requestIdleCallback` (with fallback) instead of a fixed `setTimeout` — this lets the browser finish rendering before loading analytics.

```html
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  function loadAnalytics(){
    if(window._analyticsLoaded)return;window._analyticsLoaded=true;
    dataLayer.push({'gtm.start':new Date().getTime(),event:'gtm.js'});
    var g=document.createElement('script');g.async=true;
    g.src='https://www.googletagmanager.com/gtm.js?id=GTM-XXXXXXX';
    document.head.appendChild(g);
    var a=document.createElement('script');a.async=true;
    a.src='https://www.googletagmanager.com/gtag/js?id=AW-XXXXXXXXXXX';
    document.head.appendChild(a);
    gtag('js',new Date());gtag('config','AW-XXXXXXXXXXX');
  }
  if('requestIdleCallback' in window){requestIdleCallback(loadAnalytics,{timeout:5000})}
  else{setTimeout(loadAnalytics,5000)}
  ['scroll','click','touchstart','keydown'].forEach(function(e){
    document.addEventListener(e,loadAnalytics,{once:true,passive:true});
  });
</script>
```

## 6. Fix Forced Reflows

**Problem:** Reading layout properties (`scrollY`, `innerWidth`, `getBoundingClientRect`) in scroll handlers causes synchronous layout thrashing.

**Fixes:**

### Scroll handlers — batch with `requestAnimationFrame`
```js
let ticking = false;
function handleScroll() {
  if (ticking) return;
  ticking = true;
  requestAnimationFrame(() => {
    // Read layout properties here
    ticking = false;
  });
}
window.addEventListener('scroll', handleScroll, { passive: true });
```

### Media queries — use `matchMedia` instead of `innerWidth`
```js
const mql = window.matchMedia('(max-width: 767px)');
mql.addEventListener('change', (e) => {
  // e.matches — no layout read needed
});
```

## 7. Image Optimization

**Problem:** Images without `width`/`height` cause layout shift (CLS). Images loaded eagerly below the fold waste bandwidth. Oversized source images waste bandwidth.

### 7a. Width/Height and Lazy Loading

- Add explicit `width` and `height` attributes to **every** `<img>` tag — including icons, avatars, and staff photos. Match the CSS display size (e.g., `w-5 h-5` = `width="20" height="20"`, `w-12 h-12` = `width="48" height="48"`).
- Add `loading="lazy"` to all images **except** the above-the-fold logo.
- Avatar images (`<AvatarImage>`) also accept `width`, `height`, and `loading="lazy"`.

```html
<!-- Header logo — eager (above fold, LCP candidate) -->
<img src="/logo.svg" alt="Logo" width="138" height="28" />

<!-- Icons (w-5 h-5 = 20px, w-6 h-6 = 24px) -->
<img src="/zalo-icon.svg" alt="Zalo" width="20" height="20" loading="lazy" class="w-5 h-5" />
<img src="/zalo-icon.svg" alt="Zalo" width="24" height="24" loading="lazy" class="w-6 h-6" />

<!-- Staff/team photos (w-12 h-12 = 48px) -->
<img src="/staff-1.webp" alt="Team" width="48" height="48" loading="lazy" class="w-12 h-12 rounded-full object-cover" />

<!-- Avatars (h-10 w-10 = 40px) -->
<img src="/avatar.webp" alt="Name" width="40" height="40" loading="lazy" />
```

### 7b. Resize Images to Actual Display Size

**Problem:** Source images much larger than their display size (e.g., 1244x1244 PNG for a 40x40 avatar) waste bandwidth dramatically.

**Rules:**
- **Avatars (40x40 display):** Resize to 80x80 (2x retina). Convert PNG → JPEG. Target: <10KB each.
- **Content images (half-width sections):** Resize to max 800px wide. Target: <100KB.
- **Hero/full-width images:** Resize to max 1200px wide. Target: <150KB.
- Use **WebP** for all photos (superior compression). Keep SVG for logos/icons.

**Convert to WebP (cwebp):**
```bash
# Install: brew install webp

# Convert avatar (quality 80)
cwebp -q 80 -resize 80 80 input.png -o avatar.webp

# Convert content image (quality 80, max 800px wide)
cwebp -q 80 -resize 800 0 input.jpeg -o content.webp
```

**macOS alternative (sips + cwebp):**
```bash
# Resize first, then convert
sips -Z 80 input.png --out temp.png
cwebp -q 80 temp.png -o avatar.webp
```

**Real examples from this project:**
- 5 avatars: .jpg → .webp (40 KiB → 10 KiB, **-75%**)
- `security.jpeg` → `.webp` (103 KiB → 38 KiB, **-64%**)
- 3 staff photos: .jpg/.png → .webp (83 KiB → 11 KiB, **-87%**)

**After converting:** Update all `src` references in `index.html` from `.jpg`/`.jpeg`/`.png` to `.webp`, and delete the old files.

### 7c. Self-Host External Images

**Problem:** External image URLs (e.g., Unsplash) add DNS lookup, connection, and download latency. They also break if the external service changes URLs.

**Fix:** Download, resize, and serve from `public/` instead. Delete any unused URL constant files (e.g. `src/assets/images.ts`).

```html
<!-- Before (external request) -->
<img src="https://images.unsplash.com/photo-..." />

<!-- After (local, optimized) -->
<img src="/security.jpeg" />  <!-- 103KB, 800x533 -->
```

## 8. Fix Animation Jank (LCP Impact)

**Current approach:** Hero renders instantly as static HTML. Animations use vanilla JS + CSS transitions with `IntersectionObserver`. No framework animation library.

**Problem (general):** Fade-in animations on hero elements cause:
- Content invisible on first paint (bad LCP)
- `scale` transforms trigger expensive layout recalculations

**Fixes:**
- **Hero section:** Render hero content instantly — no fade-in or entrance animation.
- **Below-fold animations:** Use `opacity` + `translateY` only (GPU-composited, no layout). Never use `scale` transforms.
- **Text swap animations:** Delay start so LCP measures the static first word.

```html
<!-- GOOD: hero content renders instantly -->
<section id="hero">
  <h1>Static heading text</h1>
</section>
```

```css
/* GOOD: GPU-composited animation (no layout thrashing) */
.fade-in { opacity: 0; transform: translateY(20px); transition: opacity 0.5s, transform 0.5s; }
.fade-in.visible { opacity: 1; transform: none; }
```

## 9. Accessibility (PageSpeed flags)

### Missing `<label>` on `<select>`
```html
<label for="plan-selector" class="sr-only">Chọn gói</label>
<select id="plan-selector" ...>
```

### Missing `title` on icon-only links
```html
<a href="https://zalo.me/..." title="Chat Zalo">
  <img src="/zalo-icon.svg" alt="Zalo" loading="lazy" />
</a>
```

## 10. Mobile Spacing

**Problem:** Desktop section padding (`py-24` = 96px) doubles up between sections on mobile, creating excessive whitespace.

**Fix:** Use responsive padding — smaller on mobile, full on desktop.

```html
<!-- Before -->
<section class="py-24 bg-background">

<!-- After -->
<section class="py-12 md:py-24 bg-background">
```

Same for section headers:
```html
<div class="text-center mb-8 md:mb-16">
```

## 11. Source Maps

**Problem:** PageSpeed reports "Missing source maps for large first-party JavaScript."

### 11a. Vite-processed files

```ts
// vite.config.ts
build: {
  sourcemap: true,
}
```

### 11b. Static JS files in `public/`

Vite copies `public/` files as-is without source maps. The `sourcemap: true` setting only applies to files processed by Rollup (entry points, imports). Static JS files like `public/scripts/animations.js` need a custom Vite plugin to generate identity source maps at build time.

**Fix:** Use `publicJsSourcemapPlugin()` in `vite.config.ts` — it runs in `closeBundle`, finds JS files in `dist/` that originated from `public/`, and generates identity source maps with `sourcesContent` embedded.

```ts
// vite.config.ts — already implemented
plugins: [
  // ...
  publicJsSourcemapPlugin(),  // generates .map for public/ JS files
]
```

**Output:** `dist/scripts/animations.js.map` with `//# sourceMappingURL=animations.js.map` appended to the JS file.

## 12. Code Splitting with React.lazy() *(React reference)*

**Problem:** A single monolithic JS bundle (349 KiB) loads everything upfront — including below-fold sections the user hasn't scrolled to yet.

**Fix:** Split the page into critical (hero) and deferred (below-fold) chunks using `React.lazy()` + `Suspense`.

```tsx
// Home.tsx — only hero renders in the critical path
import { lazy, Suspense } from 'react';

const BelowFold = lazy(() => import('./BelowFold'));

export default function Home() {
  return (
    <>
      {/* Hero section — renders instantly, in critical bundle */}
      <section>...</section>

      {/* Below-fold — separate chunk, loaded after initial render */}
      <Suspense fallback={null}>
        <BelowFold />
      </Suspense>
    </>
  );
}
```

**For animations that only run after a delay** (e.g., text swap that starts 10s after load), double-defer with a timer + lazy import:

```tsx
const AnimatedSwapText = lazy(() => import('./HeroSwapText'));

function HeroSwapText() {
  const [started, setStarted] = useState(false);
  useEffect(() => {
    const delay = setTimeout(() => setStarted(true), 10000);
    return () => clearTimeout(delay);
  }, []);

  if (!started) return <span>{staticWord}</span>;

  return (
    <Suspense fallback={<span>{staticWord}</span>}>
      <AnimatedSwapText words={words} />
    </Suspense>
  );
}
```

**Result:** Critical bundle dropped from 349 KiB → 184 KiB (47% reduction). framer-motion (115 KiB) only loads after 10s.

## 13. Replace framer-motion with CSS Transitions *(React reference)*

**Problem:** `framer-motion` adds ~112 KiB to any chunk that imports it. Below-fold fade-in animations don't need a full physics engine.

**Fix:** Replace `motion.div` with CSS transitions + IntersectionObserver.

### useInView hook (replaces `whileInView`)
```tsx
function useInView(options?: { once?: boolean; margin?: string }) {
  const ref = useRef<HTMLDivElement>(null);
  const [inView, setInView] = useState(false);
  useEffect(() => {
    const el = ref.current;
    if (!el) return;
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setInView(true);
          if (options?.once !== false) observer.disconnect();
        } else if (options?.once === false) {
          setInView(false);
        }
      },
      { threshold: 0.1, rootMargin: options?.margin ?? '0px' }
    );
    observer.observe(el);
    return () => observer.disconnect();
  }, []);
  return { ref, inView };
}
```

### FadeIn component (replaces `motion.div` with `initial/whileInView`)
```tsx
function FadeIn({ children, className = '', delay = 0, direction = 'up' }: {
  children: React.ReactNode; className?: string; delay?: number;
  direction?: 'up' | 'down' | 'left' | 'right' | 'none';
}) {
  const { ref, inView } = useInView();
  const transforms = {
    up: 'translateY(20px)', down: 'translateY(-20px)',
    left: 'translateX(-20px)', right: 'translateX(20px)', none: '',
  };
  return (
    <div ref={ref} className={className}
      style={{
        opacity: inView ? 1 : 0,
        transform: inView ? 'none' : transforms[direction],
        transition: `opacity 0.5s ease ${delay}s, transform 0.5s ease ${delay}s`,
      }}>
      {children}
    </div>
  );
}
```

### Migration pattern
```tsx
// Before (framer-motion)
<motion.div
  initial={{ opacity: 0, y: 20 }}
  whileInView={{ opacity: 1, y: 0 }}
  viewport={{ once: true }}
  transition={{ delay: 0.2 }}
>
  <Card>...</Card>
</motion.div>

// After (CSS transitions)
<FadeIn delay={0.2}>
  <Card>...</Card>
</FadeIn>
```

### Progress bar animations (replaces `motion.div` animate width)
```tsx
// CSS transition on width — triggered by IntersectionObserver
<div
  style={{
    width: inView ? `${percentage}%` : '0%',
    transition: 'width 1s ease',
    transitionDelay: `${index * 0.2}s`,
  }}
/>
```

**Note:** Keep framer-motion only where CSS can't replicate it (e.g., `AnimatePresence` exit animations, spring physics for text swap). Isolate these into lazy-loaded components.

**Result:** BelowFold chunk dropped from 165 KiB → 52 KiB. framer-motion completely removed from critical path.

## 14. Preload Lazy-Loaded Chunks *(React reference)*

**Problem:** `React.lazy(() => import('./BelowFold'))` creates a network waterfall: HTML → index.js → BelowFold.js. The browser can't discover BelowFold.js until index.js has fully loaded and executed, adding 300-900ms to the critical path.

**Fix:** Add `<link rel="modulepreload">` for all non-entry JS chunks via a Vite plugin. This tells the browser to start fetching lazy chunks immediately from the HTML, in parallel with the entry script.

```ts
// vite.config.ts
function modulePreloadPlugin(): Plugin {
  return {
    name: 'modulepreload-lazy-chunks',
    apply: 'build',
    enforce: 'post',
    transformIndexHtml: {
      order: 'post',
      handler(html, ctx) {
        if (!ctx.bundle) return html;
        const preloads: string[] = [];
        for (const [fileName, chunk] of Object.entries(ctx.bundle)) {
          if (chunk.type === 'chunk' && !chunk.isEntry && fileName.endsWith('.js')) {
            preloads.push(`<link rel="modulepreload" href="/${fileName}">`);
          }
        }
        if (!preloads.length) return html;
        return html.replace('</head>', preloads.join('\n') + '\n</head>');
      },
    },
  };
}
```

**Before:** HTML → index.js (320ms) → BelowFold.js (944ms) — sequential
**After:** HTML → index.js (320ms) + BelowFold.js (parallel) — no chain

**Note:** Place this plugin before `deferEntryScriptPlugin()` in the plugins array so the preload links are in the HTML before the entry script is deferred.

## 15. Dead Code and Unused File Cleanup

**Problem:** Scaffolded projects accumulate unused files that may confuse PageSpeed audits or inflate the bundle.

**Common dead code in landing pages:**
- **Image URL constants** (e.g., `src/assets/images.ts` with Unsplash URLs) — replaced by local images in `public/` but file never deleted.
- **Animation presets** (e.g., `src/lib/motion.ts`) — unused after replacing framer-motion with CSS.
- **Empty directories** (e.g., `src/assets/`) — left behind after deleting the last file.

**Fix:** Audit with `grep -r` for imports, then delete unused files and empty directories.

```bash
# Check if any file imports from a module
grep -r "from.*assets/images" src/
grep -r "from.*lib/motion" src/

# If no results, safe to delete
rm src/assets/images.ts
rm src/lib/motion.ts
rmdir src/assets/  # only if empty
```

## 16. Static HTML Shell for Instant LCP *(React reference — already achieved)*

**Problem:** The browser downloads HTML, then fetches JS, then React executes and renders the hero. Until JS runs, the user sees a blank page — this kills LCP.

**Fix:** Place a static HTML copy of the header + hero inside `<div id="root">` in `index.html`. The browser paints it immediately on HTML parse. When React mounts via `createRoot().render()`, it replaces the static shell seamlessly.

```html
<div id="root">
  <!-- Static shell: replaced by React on mount -->
  <div class="flex min-h-screen flex-col bg-background font-sans">
    <header class="fixed top-0 z-50 w-full bg-transparent py-5">
      <div class="container mx-auto flex items-center justify-between px-4">
        <a href="#"><img src="/logo.svg" alt="Logo" class="h-7" width="138" height="28"></a>
        <!-- Nav links with inline SVG icons (no JS needed) -->
      </div>
    </header>
    <main class="flex-grow">
      <section id="hero" class="relative min-h-[90vh] flex items-center justify-center pt-20 bg-background">
        <!-- Badge, h1 (static first word), description, CTA as <a>, stat cards -->
        <!-- Use inline <svg> for icons (Shield, Zap, Check, Lock) instead of Lucide React -->
      </section>
    </main>
  </div>
</div>
```

**Rules:**
- CTA button uses `<a href="#pricing">` (no JS `onClick`)
- Icons are inline `<svg>` (copied from Lucide icon set)
- Only include above-fold content (header + hero) — no footer, no below-fold
- Static text uses the first word of any animated text swap

**Result:** LCP measures against the static HTML paint instead of waiting for JS → React render. Cost: ~2 KiB gzip extra in the HTML.

## 17. Critical CSS Design Tokens

**Problem:** The static HTML shell may flash unstyled if CSS custom properties haven't been parsed yet.

**Fix:** Add a `<style>` block in `<head>` with **only** `:root` design tokens. Place it after `<meta charset>` and `<meta viewport>` (which must be in the first 1024 bytes).

```html
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover" />
  <!-- Critical CSS: tokens ONLY -->
  <style>
    :root{--font-sans:"Plus Jakarta Sans",system-ui,sans-serif;--background:oklch(.99 .002 245);--foreground:oklch(.18 .03 245);--primary:oklch(.40 .17 263);--primary-foreground:oklch(.99 .01 263);/* ... all tokens */}
  </style>
  <!-- Everything else (fonts, JSON-LD) follows -->
```

### Critical CSS pitfalls with Tailwind v4

Tailwind v4 uses CSS cascade layers (`@layer properties, theme, base, components, utilities`). Any CSS you add in critical `<style>` can break Tailwind if done incorrectly:

**DO NOT add unlayered resets:**
```html
<!-- BAD: unlayered * selector overrides ALL Tailwind @layer utilities -->
<style>
  *,*::before,*::after{box-sizing:border-box;margin:0;padding:0}
  body{font-family:var(--font-sans);background:var(--background);color:var(--foreground)}
  h1,h2,h3{font-weight:600}
</style>
```
Unlayered CSS always wins over `@layer`-based CSS. This breaks `mx-auto`, `px-4`, `container`, and all spacing/layout utilities.

**DO NOT add `@layer utilities` block:**
```html
<!-- BAD: declares @layer utilities before Tailwind, breaking cascade order -->
<style>
  @layer utilities{.flex{display:flex}.text-center{text-align:center}/* ... */}
</style>
```
CSS layer order is determined by first appearance. Declaring `@layer utilities` before Tailwind's layer order declaration corrupts the entire cascade.

**ONLY include `:root` custom property declarations:**
```html
<!-- GOOD: :root tokens don't interfere with any layer -->
<style>
  :root{--font-sans:...;--background:...;--primary:...;}
</style>
```

## 18. Move Analytics Script to End of Body

**Problem:** Even though the GTM/gtag loader is deferred via `requestIdleCallback`, placing the `<script>` block in `<head>` still pauses HTML parsing briefly while the browser parses the inline JavaScript.

**Fix:** Move the entire analytics `<script>` block to just before `</body>`, after the app mount script.

```html
<body>
  <div id="root">...</div>
  <script type="module" src="/src/main.tsx"></script>
  <!-- Analytics at the very end — zero impact on initial render -->
  <script>
    window.dataLayer = window.dataLayer || [];
    function gtag(){dataLayer.push(arguments);}
    function loadAnalytics(){...}
    if('requestIdleCallback' in window){requestIdleCallback(loadAnalytics,{timeout:5000})}
    else{setTimeout(loadAnalytics,5000)}
    ['scroll','click','touchstart','keydown'].forEach(function(e){
      document.addEventListener(e,loadAnalytics,{once:true,passive:true});
    });
  </script>
</body>
```

## 19. Split Vendor Chunks for Parallel Loading *(React reference)*

**Problem:** A single large entry JS file (184 KiB / 60 KiB gzip) creates a long critical path even with `modulepreload`, because it's one sequential download.

**Fix:** Use Vite's `manualChunks` to split vendor libraries into separate files. The browser fetches them all in parallel via `modulepreload`.

```ts
// vite.config.ts
build: {
  rollupOptions: {
    output: {
      manualChunks: {
        'react-vendor': ['react', 'react-dom'],
        'ui-vendor': ['lucide-react', '@radix-ui/react-slot',
                      'class-variance-authority', 'clsx', 'tailwind-merge'],
      },
    },
  },
},
```

**Result:**
| Chunk | Gzip | Content |
|-------|------|---------|
| `index.js` | 6 KiB | App code (Home, Layout) |
| `react-vendor.js` | 45 KiB | React + ReactDOM |
| `ui-vendor.js` | 11 KiB | Lucide, Radix, CVA, clsx |

All 3 load in parallel (modulepreloaded). Critical path limited by largest parallel chunk (45 KiB) instead of combined 60 KiB.

**Dedup note:** Update the `modulePreloadPlugin` to skip chunks Vite already preloads:
```ts
if (html.includes(`/${fileName}"`)) continue;
```

## 20. SVG Dimensions to Prevent Layout Shift

**Problem:** Inline or embedded SVGs without explicit `width`/`height` cause CLS as the browser calculates their intrinsic dimensions.

**Fix:** Add `width` and `height` attributes matching the `viewBox` to all SVGs.

```html
<!-- Before (causes CLS) -->
<svg viewBox="0 0 1440 120" class="w-full h-16 md:h-24">

<!-- After (no CLS) -->
<svg viewBox="0 0 1440 120" width="1440" height="120" class="w-full h-16 md:h-24">
```

The CSS classes override the actual display size, but the HTML attributes give the browser an aspect ratio to reserve space before rendering.

## 21. Color Contrast (WCAG AA)

**Problem:** PageSpeed flags "Background and foreground colors do not have a sufficient contrast ratio" (minimum 4.5:1 for normal text).

**Common failures:**
- **Muted text on white:** `--muted-foreground: oklch(0.45)` gives ~4.2:1 — just below AA threshold.
- **Footer text on black:** `text-zinc-500` (~4.0:1) and `text-zinc-600` (~2.6:1) both fail.
- **Badges with light tints:** `text-emerald-600` on `bg-emerald-500/10` can fail.

**Fixes:**
```css
/* Darken muted-foreground from 0.45 to 0.40 for 4.8:1 on white */
--muted-foreground: oklch(0.40 0.03 245);
```

```tsx
/* Footer: zinc-500/600 on black → zinc-400 for 4.6:1+ */
<p className="text-zinc-400 text-sm">License text</p>
<div className="text-zinc-400">Footer links</div>
<p className="text-xs text-zinc-400">Copyright</p>

/* Badge: emerald-600 → emerald-700 for better contrast on light tint */
<Badge className="text-emerald-700 bg-emerald-500/10">
```

**Quick contrast check (oklch):** For text on white (L=1.0), the text lightness must be ≤0.42 to meet 4.5:1. For text on black (L=0.0), the text lightness must be ≥0.58.

## 22. `<head>` Element Ordering (Best Practices)

**Problem:** Lighthouse flags "Charset declaration is missing or occurs too late in the HTML" if `<meta charset>` is not within the first 1024 bytes. Critical CSS tokens or large JSON-LD blocks can push it past this limit.

**Fix:** Always place `<meta charset>` and `<meta viewport>` as the first two elements in `<head>`, before any `<style>` or `<script>` blocks.

```html
<head>
  <!-- 1. Charset + viewport MUST be first (within 1024 bytes) -->
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover" />
  <!-- 2. Critical CSS tokens -->
  <style>:root{--font-sans:...;--primary:...;}</style>
  <!-- 3. Font preconnect + preload -->
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link rel="preload" as="style" href="https://fonts.googleapis.com/css2?..." onload="this.rel='stylesheet'">
  <!-- 4. Remaining meta tags, canonical, title -->
  <link rel="icon" type="image/svg+xml" href="/favicon.svg" />
  <title>...</title>
  <meta name="description" content="..." />
  <!-- 5. OG / Twitter meta -->
  <!-- 6. JSON-LD structured data (can be large, goes last in head) -->
</head>
```

**Verify:** `head -c 1024 dist/index.html | grep charset` should return a match.

## 23. Avoid Async CSS Loading with Tailwind v4

**Problem:** Async-loading Tailwind CSS via `preload + onload` pattern causes FOUC (Flash of Unstyled Content) — the page renders with broken layout before CSS applies.

**Anti-pattern (DO NOT):**
```ts
// BAD: asyncCssPlugin that converts stylesheet to preload+onload
function asyncCssPlugin(): Plugin {
  return {
    transformIndexHtml: {
      handler(html) {
        return html.replace(
          /<link\s+rel="stylesheet"[^>]*href="(\/assets\/[^"]+\.css)"[^>]*>/g,
          (_match, href) =>
            `<link rel="preload" as="style" href="${href}" onload="this.rel='stylesheet'">`
        );
      },
    },
  };
}
```

**Why it fails:**
1. Page renders before CSS loads → broken layout for all sections
2. Combined with `deferEntryScriptPlugin`, JS is also delayed → no fallback
3. Critical CSS subset in `<head>` can't cover all Tailwind utilities without breaking `@layer` cascade

**Correct approach:** Keep Tailwind CSS as a normal render-blocking `<link rel="stylesheet">`. At ~16 KiB gzip, the blocking cost is negligible compared to the FOUC damage.

---

## Quick Checklist

### HTML & Head
- [ ] `<meta charset>` + `<meta viewport>` are the first two elements in `<head>` (within 1024 bytes)
- [ ] CSS inlined into HTML at build time (via `inlineCssPlugin`) — zero render-blocking requests
- [ ] Google Fonts: preconnect + preload with `fetchpriority="high"` + `onload` fallback
- [ ] Font preload links placed BEFORE inlined CSS in `<head>`
- [ ] Static HTML renders instantly (no JS required for first paint)
- [ ] `<meta name="robots" content="index, follow">` for production

### JavaScript
- [ ] Single vanilla JS file (`animations.js`) loaded with `defer`
- [ ] No framework JS (React, Vue, etc.) — vanilla JS only
- [ ] Scroll handlers use `requestAnimationFrame` + `{ passive: true }`
- [ ] `matchMedia` used instead of `window.innerWidth`
- [ ] Unused dependencies removed from `package.json`

### CSS & Tailwind v4
- [ ] No unlayered CSS resets (`*{margin:0}`) in `<style>` — breaks Tailwind `@layer` cascade
- [ ] No `@layer utilities` block in critical CSS — corrupts Tailwind layer ordering
- [ ] Animations use CSS transitions + `IntersectionObserver` (no animation library)
- [ ] Design tokens use oklch color space

### Images
- [ ] All `<img>` tags have explicit `width` and `height` (including icons, avatars, staff photos)
- [ ] All images have `loading="lazy"` except above-fold logo
- [ ] Photos converted to **WebP** (cwebp quality 80), SVG only for logos/icons
- [ ] Images resized to actual display size (avatars: 80x80, content: 800px, hero: 1200px)
- [ ] External images (Unsplash etc.) downloaded and self-hosted in `public/`
- [ ] All SVGs have explicit `width`/`height` attributes (prevents CLS)

### Analytics
- [ ] GTM/gtag deferred until interaction or `requestIdleCallback` (3.5s timeout fallback)
- [ ] GTM/analytics `<script>` at end of `<body>` (not in `<head>`)

### Performance
- [ ] Hero section renders instantly (no fade-in animation)
- [ ] `scale` animations replaced with `opacity` + `translateY`
- [ ] Mobile section padding reduced (`py-12 md:py-24`)
- [ ] Native HTML elements: `<details>/<summary>` for FAQ, `<select>` for mobile pricing
- [ ] Build output: `index.html` (~29 KiB gzip) + `animations.js` (~1.5 KiB gzip)
- [ ] Unused files and empty directories deleted

### Accessibility
- [ ] All text meets WCAG AA contrast (4.5:1 minimum)
- [ ] `--muted-foreground` lightness ≤0.42 for 4.5:1 on white
- [ ] Footer text `text-zinc-400` or lighter on `bg-black` (not zinc-500/600)
- [ ] Icon-only links have `title` attributes
- [ ] `<select>` elements have associated `<label>`
