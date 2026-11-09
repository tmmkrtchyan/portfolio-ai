---
name: env-secrets
description: Handle environment variables and secrets for portfolio-ai across code, .env.example, Vercel and GitHub Actions without ever committing a secret. Use when adding a new external service or API key, touching lib/container.ts, editing workflows that need secrets, or debugging a missing env var.
---

# Environment variables and secrets

API keys live **only** in Vercel project env vars and GitHub Actions secrets. They never appear in code, tests, fixtures, logs, PR descriptions or chat.

## Adding a variable

1. Add it to `.env.example` with an empty value and a one-line comment (what it is, where to get the free-tier key):
   ```
   # Gemini API key (Google AI Studio, free tier)
   GEMINI_API_KEY=
   ```
2. Read it **only** in `lib/container.ts` (or `scripts/ingest.ts`) through the `env()` helper that throws on missing values. Nothing else touches `process.env`.
3. Server-only: no `NEXT_PUBLIC_` prefix for keys. `NEXT_PUBLIC_` is only for values safe to show every visitor (e.g. site URL).
4. Add it to the env var table in `CLAUDE.md`.
5. In the PR description, list the variable and where the owner must set it (Vercel: Production/Preview; GitHub: Actions secret). Agents can't set these themselves.

## In GitHub Actions

```yaml
- run: npm run ingest
  env:
    GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }}
    UPSTASH_VECTOR_REST_URL: ${{ secrets.UPSTASH_VECTOR_REST_URL }}
    UPSTASH_VECTOR_REST_TOKEN: ${{ secrets.UPSTASH_VECTOR_REST_TOKEN }}
```

- Pass secrets only to the step that needs them, not at job level.
- Lint, typecheck, unit tests and Playwright run **without** secrets (fakes and `page.route` mocks). PRs from forks don't get secrets, so those jobs must still pass.
- Steps that need real keys (ingest, evals, deploy) run on `main` or on `workflow_dispatch`, and skip cleanly with a clear message when the secret is absent.

## Never

- Commit `.env`, `.env.local` or any file with real values (`.gitignore` covers `.env*` except `.env.example`).
- Echo secrets in logs (`echo $KEY`, `console.log(process.env)`), or put them in error messages.
- Use a real key in tests "just to check".
- Hard-code a fallback key (`process.env.KEY ?? 'abc…'`).

## Free-tier hygiene

- Use separate keys for production and CI where the provider allows it, so a CI loop can't burn the production quota.
- Rate limiting and the semantic cache protect quotas; don't disable them in production config.
