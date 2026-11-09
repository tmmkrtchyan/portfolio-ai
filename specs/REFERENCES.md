# Reference repos review

The full guidelines and code examples are in [ARCHITECTURE.md](ARCHITECTURE.md).

What to take from two reference projects for portfolio-ai (Next.js on Vercel, RAG chatbot, CI-only verification).

## 1. Clean Architecture example for Next.js

A todo app with auth that shows Clean Architecture in Next.js: entities → use cases → controllers → app, with interfaces for every external service, mocks for tests, and an ESLint rule that enforces the layering.

### Adopt

| Idea | Why it matters here |
|---|---|
| **Dependency rule enforced by `eslint-plugin-boundaries`** | Most valuable piece. Agents (Ralph runs) will write most of the code; a lint rule that fails CI when `app/` imports an adapter directly keeps the structure from drifting. |
| **Ports (interfaces) for every external service** | `LlmProvider`, `EmbeddingProvider`, `VectorStore`, `SemanticCache`, `RateLimiter`, `Tracer`. This is exactly what the v3 model router needs (Gemini → Groq/Workers AI on 429), and it lets us swap Upstash ↔ Supabase without touching core logic. |
| **In-memory/mock adapters for unit tests** | You don't run the app locally and CI has no API keys for PRs from forks, so core logic (guardrails, retrieval ranking, refusal, router fallback) must be testable with fakes. |
| **Zod validation at the entry point** | `/api/chat` validates body, enforces the input length cap and returns typed errors before anything costs quota. |
| **Typed domain errors** | `RateLimitedError`, `InputTooLongError`, `OffTopicError`, `NoRelevantContextError`, `ProviderQuotaError` → mapped to HTTP status + UI state in one place. |
| **Factory functions instead of classes** | `createChatUseCase(deps) => (input) => …` — simple, testable, no framework. |
| **Vitest + coverage in CI**, tests mirroring source tree | Matches "every PR has tests". |

### Layout

See [ARCHITECTURE.md §2](ARCHITECTURE.md#2-folder-layout) for the folder layout and §4 for the boundary rules.

## 2. Portable AI-assistant config folder

A portable folder of GitHub Copilot / VS Code config: ~20 skills, prompts and instructions, written for other projects and stacks (Neon/Prisma).

### What we take

| Source skill | What we take |
|---|---|
| `clean-architecture-frontend` → "Pragmatic: when NOT to use full Clean Architecture" | Keep UI components simple; layering only for server-side logic (chat, RAG, guardrails). |
| `testing-automation` | Vitest for units, Playwright for e2e/screenshots at mobile + desktop, run in CI. |
| `component-development` | shadcn/ui composition, a11y checklist (labels, focus, keyboard, `aria-live` for streaming answers), dark mode via tokens. |
| `performance-optimization` | Optional later: Lighthouse CI budget on the home page. |
| `code-review` | A short review checklist in CLAUDE.md (boundaries, tests, no secrets, tokens not colours). |

These become a few short Claude skills under `.claude/skills/` (e.g. `ui-component`, `add-adapter`, `write-tests`) once the specs are approved.

## 3. Fit with Ralph (PR #1)

- Ralph's root `CLAUDE.md` is its loop prompt, but the kickoff wants `CLAUDE.md` to hold durable project rules. **Move Ralph's prompt to `ralph/CLAUDE.md`** (and `ralph.sh`, `prompt.md`, skills into `ralph/`) and point the script at it.
- Ralph uses one branch per PRD; our rule is one issue = one branch = one PR. So: **one PRD per issue**, with `branchName: feat/<issue>-<slug>`, and the build run opens a PR with `Closes #N` instead of pushing a long-lived branch.
- Ralph's "commit only if checks pass" must mean **CI checks** (you don't run locally): the build run pushes, waits for the workflow, and fixes until green.

## Summary of what goes into the specs PR

1. `eslint-plugin-boundaries` rules for the layered `lib/` layout.
2. Port interfaces + in-memory fakes as the testing strategy.
3. Zod at `/api/chat` and a typed error → HTTP/UI mapping.
4. Vitest + Playwright in the CI workflow from day one.
5. A short review checklist and stack rules in `CLAUDE.md`; Ralph moved under `ralph/`.
