---
name: perf-budget
description: Keep portfolio-ai fast — Core Web Vitals targets, Next.js rendering and bundle rules, and the Lighthouse CI budget. Use when adding pages, images, fonts, client components or dependencies, when Lighthouse CI fails, or when asked about performance.
---

# Performance budget

A portfolio is judged in the first second. Targets for the home page on mobile:

| Metric | Budget |
|---|---|
| LCP | ≤ 2.5 s |
| CLS | ≤ 0.1 |
| INP | ≤ 200 ms |
| FCP | ≤ 1.8 s |
| Lighthouse performance / accessibility / best practices / SEO | ≥ 0.9 each |

## Rules

- **Static by default.** Pages built from `kb/` render at build time (Server Components, no `fetch` at request time). Only `/api/chat` is dynamic.
- **Small client bundle.** `'use client'` only on interactive leaves. Load the chat widget lazily (`next/dynamic`) when the visitor opens it; the home page must not ship the chat code up front.
- **Images:** `next/image` with explicit `width`/`height` (or `fill` + sized parent), `priority` only on the LCP image, modern formats.
- **Fonts:** `next/font` (self-hosted, `display: 'swap'`); at most two families.
- **No layout shift:** reserve space for the chat button, images and streamed text containers.
- **Dependencies:** check size before adding a client-side package; prefer what shadcn/Radix already provide. No heavy markdown/highlighting libraries on the client.
- **Streaming:** `/api/chat` streams tokens so the first words appear fast; show a skeleton immediately.

## Lighthouse CI

Runs in GitHub Actions against the production build (`next build && next start`):

```js
// lighthouserc.cjs
module.exports = {
  ci: {
    collect: { url: ['http://localhost:3000/'], startServerCommand: 'npm run start', numberOfRuns: 3 },
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 0.9 }],
        'categories:accessibility': ['error', { minScore: 0.9 }],
        'categories:best-practices': ['error', { minScore: 0.9 }],
        'categories:seo': ['error', { minScore: 0.9 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
      },
    },
    upload: { target: 'temporary-public-storage' },
  },
};
```

## When the budget fails

1. Open the Lighthouse report link from the CI log; find the failing audit.
2. Typical fixes: lazy-load a client component, add image dimensions, drop an unused dependency, move work to build time.
3. Never lower a threshold in the same PR that breaks it; if a threshold must change, explain why in the PR description.
