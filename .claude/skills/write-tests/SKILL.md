---
name: write-tests
description: Write or fix tests for portfolio-ai — Vitest unit tests with fakes for lib/, and Playwright e2e and screenshot tests at mobile and desktop widths. Use whenever a change adds or modifies behaviour, when CI tests fail, or when setting up test config or the CI test job.
---

# Writing tests

All verification runs in GitHub Actions; nobody runs the app locally. Tests must pass in CI with **no API keys**.

## What to test

| Change | Test |
|---|---|
| Use case / controller in `lib/` | Vitest unit test with fakes: happy path + every error/refusal branch |
| Adapter in `lib/adapters/` | Unit test that SDK errors become domain errors (mock the SDK call, not the network) |
| New page or UI state | Playwright test + screenshot at 390px and 1280px |
| `/api/chat` behaviour | Playwright API test or route-handler unit test with the container swapped for fakes |

## Unit tests (Vitest)

- Files live in `tests/unit/`, mirroring `lib/` (`lib/use-cases/x.ts` → `tests/unit/use-cases/x.test.ts`).
- Build the unit under test directly with fakes from `lib/adapters/fakes/`; never import `lib/container.ts`.
- Name tests by behaviour: `it('refuses when no chunk is relevant enough')`, grouped with `describe('<use case>')`.
- No `any`. Use port types or `Partial<T>`; if a test deliberately passes bad input, use `// @ts-expect-error <reason>`.
- One behaviour per `it`. Prefer `await expect(promise).rejects.toBeInstanceOf(RateLimitedError)` over try/catch.

```ts
import { describe, expect, it } from 'vitest';
import { chatController } from '@/lib/controllers/chat.controller';
import { RateLimitedError } from '@/lib/entities/errors';
import { createFakeRateLimiter } from '@/lib/adapters/fakes/fake.rate-limiter';

describe('chatController', () => {
  it('throws RateLimitedError when the client is over the limit', async () => {
    const controller = chatController({
      rateLimiter: createFakeRateLimiter({ allowed: false, retryAfterSeconds: 30 }),
      checkGuardrails: async () => ({ ok: true }),
      answerQuestion: async () => ({ kind: 'refusal', reason: 'no-context' }),
    });

    await expect(controller({ question: 'hi' }, 'ip-1')).rejects.toBeInstanceOf(RateLimitedError);
  });
});
```

## E2E and screenshots (Playwright)

- Files live in `tests/e2e/`. Use role-based locators (`getByRole`, `getByLabel`); `data-testid` only when no accessible name exists.
- Mock every external call with `page.route('**/api/chat', ...)` so CI needs no keys and results are deterministic.
- Two projects in `playwright.config.ts`: `mobile` (390×844) and `desktop` (1280×800).
- Screenshots are coarse checks: `await expect(page).toHaveScreenshot('home.png', { maxDiffPixelRatio: 0.02 })`. Disable animations (`animations: 'disabled'`) and wait for fonts.
- Cover the design states: home, chat open, answer with citations, loading, error.

```ts
import { test, expect } from '@playwright/test';

test('shows citations under an answer', async ({ page }) => {
  await page.route('**/api/chat', (route) =>
    route.fulfill({ json: { kind: 'answer', text: 'Five years.', citations: [{ source: 'kb/cv.md', heading: 'Experience' }] } }),
  );
  await page.goto('/');
  await page.getByRole('button', { name: /chat/i }).click();
  await page.getByLabel(/ask a question/i).fill('How much TypeScript experience?');
  await page.keyboard.press('Enter');
  await expect(page.getByRole('link', { name: /experience/i })).toBeVisible();
  await expect(page).toHaveScreenshot('answer-with-citations.png');
});
```

## Updating screenshot baselines

Baselines are generated in CI (Linux fonts differ from macOS). When a UI change is intended, run the workflow with `--update-snapshots`, commit the new PNGs into the same single PR commit, and say so in the PR description.

## CI

The test job runs `npm run lint`, `npm run typecheck`, `npm run test -- --coverage` and `npx playwright test`. Keep coverage of `lib/` at 80%+. A red CI is never "fixed" by skipping or deleting a test without saying why in the PR.
