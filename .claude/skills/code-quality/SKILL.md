---
name: code-quality
description: Pre-PR quality pass for portfolio-ai — TypeScript and Next.js lint rules, error handling standards, type safety, and the review checklist. Use before opening or updating any PR, when lint or typecheck fails in CI, or when asked to review code.
---

# Code quality pass

Run this before every PR update. CI must be green: `npm run lint`, `npm run typecheck`, `npm run test`.

## 1. Check only what changed

```bash
git diff --name-only origin/main...HEAD
```

Fix every lint/type error in changed files. Don't reformat or refactor unrelated files in the same PR.

## 2. Severity

| Severity | Examples | Action |
|---|---|---|
| Blocker | boundary violation, secret in code, unhandled promise, `any` in `lib/` | Fix now |
| Major | missing test for a branch, swallowed error, duplicated logic | Fix in this PR |
| Minor | naming, small duplication in tests | Fix if cheap, else note in PR |

## 3. Common fixes

- **Unused imports/variables** → remove; prefix intentionally unused params with `_`.
- **Floating promises** → `await` them, or `void promise` with a comment when fire-and-forget is intended (e.g. trace flush).
- **Nested ternaries** → early returns or a small lookup object.
- **Long functions (> ~40 lines) / deep nesting** → extract named helpers.
- **Magic numbers** → named constants in `lib/entities` (`MAX_QUESTION_LENGTH`, `MIN_RELEVANCE`).
- **Non-null assertions (`!`)** → narrow with a check or use the `env()` helper.

## 4. Error handling standards

- Adapters catch SDK errors and rethrow **domain errors** with `{ cause }`. Never let SDK errors reach use cases or `app/`.
- Preserve domain error types when rethrowing; don't wrap a `RateLimitedError` in a generic `Error`.
- Never swallow errors (`catch {}`); either handle them meaningfully or rethrow.
- `app/` maps domain errors to HTTP status and user-facing messages in one place; unknown errors become a generic 500 without leaking details.

```ts
// ✅ adapter
try {
  return await client.query(params);
} catch (err) {
  throw new VectorStoreError('query failed', { cause: err });
}
```

## 5. Type safety

- Production code: no `any`, no `@ts-ignore`. Use `unknown` and narrow (Zod at boundaries).
- Exported functions have explicit return types or come from `ReturnType<typeof factory>`.
- Tests: no `any` either; use port types, `Partial<T>`, or `// @ts-expect-error <reason>` for deliberate bad input.

## 6. Review checklist

- [ ] Follows `specs/ARCHITECTURE.md` (layers, ports/adapters, no use case calling another).
- [ ] Inputs validated with Zod; abuse limits (rate limit, length cap) intact.
- [ ] Tests cover new behaviour and error branches (see `write-tests`).
- [ ] UI uses tokens only and meets the accessibility rules (see `ui-component`).
- [ ] No secrets; new env vars documented (see `env-secrets`).
- [ ] Performance budget respected (see `perf-budget`).
- [ ] One commit on `feat/<issue>-<slug>`, PR description has `Closes #N`.
