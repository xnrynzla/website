# Performance pass — 2026-09-23

## Overall result

The portfolio remains a static HTML page with inline CSS and native JavaScript.
The existing GitHub Pages workflow deploys the repository on pushes to `main`;
no build step, packages, server, or hosting changes are required. These changes
have been verified locally, not deployed or benchmarked on production.

## Largest bottlenecks fixed

- **P2: oversized avatar.** The 75 × 75 CSS-pixel avatar fetched a 1024 × 1024,
  1,733,024-byte PNG. `logo-avatar.webp` is 225 × 225 (3× display density),
  16,258 bytes, generated with Lanczos resizing and WebP quality 90. The original
  PNG remains the social-preview image and source artwork.
- **P2: refresh-rate-dependent canvas work.** The background drew on every
  animation callback, including high-refresh displays. Drawing is capped at
  60 fps, with time-based motion retaining the original 60 Hz speed. Catch-up
  is bounded after stalls. The size-dependent gradient is created on resize
  instead of on every draw.
- Canvas scheduling and the text rotator explicitly pause while the document
  is hidden, and resume without accumulating timers or animation loops.

## Measured before/after

Baseline was the working copy at task start, including the owner's existing
uncommitted changes, not Git HEAD. Headless Microsoft Edge served the files
over local HTTP with fresh browser contexts. Each profile was observed for
three seconds after navigation, before screenshots and resize checks.

| Metric | Before | After |
| --- | ---: | ---: |
| Avatar response body | 1,733,024 bytes | 16,258 bytes |
| HTML + avatar, uncompressed file bytes | 1,757,217 | 41,412 |
| Desktop canvas draws | 656 | 153 |
| Desktop gradient creations | 656 | 1 |
| Desktop animation callback JS time | 173.2 ms | 42.7 ms |
| Mobile canvas draws | 716 | 166 |
| Mobile gradient creations | 716 | 1 |
| Mobile animation callback JS time | 490.9 ms | 122.0 ms |
| Reduced-motion canvas draws | 1 | 1 |

Desktop: 1440 × 900, DPR 1. Mobile: 390 × 844, DPR 3, with the existing
canvas DPR cap of 2. Avatar bytes fell 99.1%; HTML plus avatar fell 97.6%.
Animation callback time fell approximately 75% in these samples. Callbacks
were instrumented with `performance.now()`; these figures exclude asynchronous
GPU work and are not total browser CPU time or field Core Web Vitals. RAF
callbacks still arrive at display cadence, but excess callbacks skip drawing.
Single-run timing is machine-dependent; no production latency, LCP, or battery
life improvement is claimed.

## Correctness checks

- Desktop, mobile, and reduced-motion browser runs: no JavaScript errors,
  all eight links present, correct avatar loaded, and no horizontal overflow.
- Additional 320-pixel viewport resize checks passed for all profiles.
- Before/after mobile screenshots inspected: layout and avatar appearance
  preserved; randomized particle positions naturally differ.
- Deterministic lifecycle checks passed for visibility pause/resume, one active
  animation loop/timer, resize invalidation of the gradient, the 60 fps draw
  limit, and static reduced-motion rendering.
- Git whitespace check passed with Windows CRLF recognized.

## Backend, database, caching, and external requests

There is no PHP runtime, MySQL database, API, session handling, search endpoint,
pagination, or request-time filesystem work in this repository. Backend/query/
index optimizations are therefore not applicable. The homepage makes no
third-party resource requests; external profile/project URLs are navigation
links. No new cache or service worker was added. HTTP caching remains controlled
by GitHub Pages.

## Inferred benefits and deliberately unchanged areas

The smaller image should reduce decode memory and network waiting, but those
effects were not timed separately. Hidden-tab pause/resume behavior was tested;
its battery savings were not measured, and browsers already throttle background
work. Gradients, card blur, particle count, inline styles, content, metadata,
existing responsive/accessibility edits, robots.txt, sitemap.xml, and deployment
configuration remain intact. No framework or minification pipeline was added.

## Remaining bottlenecks and next action

No P0/P1 problem was discovered in the inspected static homepage. **P2 candidate,
unconfirmed:** full-viewport canvas compositing and backdrop blur can still cost
GPU time on low-end devices. Profile those devices before altering the visual
design or particle count. **P3:** social crawlers still fetch the original PNG;
it is intentionally retained for preview quality and compatibility and is no
longer downloaded by ordinary page loads. Production network timing and field
Core Web Vitals remain unmeasured; collect them after the normal Pages deploy.
