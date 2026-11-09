# Architecture Guide

Rules and code examples that every change must follow. It is based on two reference projects: a Clean Architecture example for Next.js, and an AI-assistant config suite with architecture, UI and testing skills. The examples are adapted to this project's domain (RAG chatbot).

## 1. Layers and the dependency rule

Layers depend **only on layers below them**, never above. Outer layers are replaceable; the core knows nothing about Next.js, Gemini, Upstash or Langfuse.

| Layer | Folder | Contains | May import |
|---|---|---|---|
| Frameworks & Drivers | `app/` | pages, components, route handlers (`/api/chat`), server actions | `lib/controllers`, `lib/entities`, `lib/container` |
| Interface Adapters | `lib/controllers/` | controllers + presenters | `lib/use-cases`, `lib/ports`, `lib/entities` |
| Application | `lib/use-cases/`, `lib/ports/` | use cases; interfaces for services | `lib/ports`, `lib/entities` |
| Entities | `lib/entities/` | models, domain errors, business rules | `lib/entities` only |
| Infrastructure | `lib/adapters/` | Gemini, Groq, Upstash Vector, rate limiter, cache, Langfuse, in-memory fakes | `lib/ports`, `lib/entities` |
| Composition | `lib/container.ts` | wires ports to adapters | everything in `lib/` |

Nothing imports from `app/`. Nothing except `lib/container.ts` imports from `lib/adapters/`.

### What each layer does

- **Frameworks & Drivers (`app/`)** – everything Next.js. Uses only *controllers*, *models* and *errors*; **must not** call use cases, ports or adapters directly. Catches domain errors and turns them into HTTP responses / UI states.
- **Controllers (`lib/controllers/`)** – the entry point into the system.
  - Do **input validation** (Zod) and abuse checks (rate limit, length cap) before calling use cases.
  - **Orchestrate** use cases; they contain no business logic themselves.
  - Let errors from deeper layers bubble up; `app/` handles them.
  - Use a **presenter** to shape output for the UI: send only what the client needs, never internal fields (raw prompts, scores, API metadata).
- **Use cases (`lib/use-cases/`)** – one operation each: `answerQuestion`, `retrieveContext`, `checkGuardrails`, `ingestDocuments`.
  - Receive pre-validated input.
  - Talk to the outside world only through **ports**.
  - **A use case must not call another use case.** That means it does too much: split it, and let the controller orchestrate.
- **Ports (`lib/ports/`)** – TypeScript interfaces for external services: `LlmProvider`, `EmbeddingProvider`, `VectorStore`, `SemanticCache`, `RateLimiter`, `Tracer`. They live in the application layer so the core can depend on them without depending on any SDK.
- **Entities (`lib/entities/`)**
  - **Models**: plain TS types for domain data (`KbChunk`, `Citation`, `ChatAnswer`), with the rules that rarely change (e.g. max question length).
  - **Errors**: our own error classes. Library errors (Gemini 429, Upstash timeouts) are **caught in adapters and converted** into these, so the core never sees SDK-specific errors.
- **Adapters (`lib/adapters/`)** – implement ports with real SDKs. One method = one external operation. No validation or business rules; just call the service and return results or throw a domain error.

## 2. Folder layout

```
app/
  api/chat/route.ts
  (site)/page.tsx
  components/            # app-specific components
components/ui/           # shadcn/ui primitives
lib/
  entities/
    models.ts
    errors.ts
  ports/
    llm-provider.ts
    vector-store.ts
    rate-limiter.ts
  use-cases/
    answer-question.ts
  controllers/
    chat.controller.ts
  adapters/
    gemini.llm.ts
    upstash.vector-store.ts
    upstash.rate-limiter.ts
    fakes/               # in-memory implementations for tests
  container.ts
tests/
  unit/                  # mirrors lib/ (tests/unit/use-cases/answer-question.test.ts)
  e2e/                   # Playwright
```

## 3. Code examples

### 3.1 Entities: models and errors

```ts
// lib/entities/models.ts
export const MAX_QUESTION_LENGTH = 500;

export type KbChunk = {
  id: string;
  source: string;      // e.g. "kb/projects/portfolio-ai.md"
  heading: string;
  text: string;
};

export type ScoredChunk = KbChunk & { score: number };

export type Citation = { source: string; heading: string };

export type ChatAnswer =
  | { kind: 'answer'; text: string; citations: Citation[] }
  | { kind: 'refusal'; reason: 'no-context' | 'off-topic' };
```

```ts
// lib/entities/errors.ts
export class InputParseError extends Error {
  constructor(message: string, options?: ErrorOptions) {
    super(message, options);
    this.name = 'InputParseError';
  }
}

export class RateLimitedError extends Error {
  constructor(public readonly retryAfterSeconds: number, options?: ErrorOptions) {
    super('Too many requests', options);
    this.name = 'RateLimitedError';
  }
}

export class ProviderUnavailableError extends Error {
  constructor(public readonly provider: string, options?: ErrorOptions) {
    super(`LLM provider ${provider} is unavailable`, options);
    this.name = 'ProviderUnavailableError';
  }
}
```

### 3.2 Ports

```ts
// lib/ports/llm-provider.ts
export type LlmMessage = { role: 'system' | 'user' | 'assistant'; content: string };

export interface LlmProvider {
  readonly name: string;
  /** Streams answer tokens. Throws ProviderUnavailableError on quota/5xx. */
  stream(messages: LlmMessage[]): AsyncIterable<string>;
}
```

```ts
// lib/ports/vector-store.ts
import type { KbChunk, ScoredChunk } from '@/lib/entities/models';

export interface VectorStore {
  upsert(chunks: (KbChunk & { embedding: number[] })[]): Promise<void>;
  query(embedding: number[], topK: number): Promise<ScoredChunk[]>;
}
```

### 3.3 Use case (factory function, dependencies injected)

```ts
// lib/use-cases/answer-question.ts
import type { LlmProvider } from '@/lib/ports/llm-provider';
import type { EmbeddingProvider } from '@/lib/ports/embedding-provider';
import type { VectorStore } from '@/lib/ports/vector-store';
import type { Citation } from '@/lib/entities/models';

export const MIN_RELEVANCE = 0.75;

export type AnswerQuestionUseCase = ReturnType<typeof answerQuestionUseCase>;

export const answerQuestionUseCase =
  (deps: { llm: LlmProvider; embeddings: EmbeddingProvider; store: VectorStore }) =>
  async (question: string) => {
    const vector = await deps.embeddings.embed(question);
    const chunks = (await deps.store.query(vector, 5)).filter((c) => c.score >= MIN_RELEVANCE);

    if (chunks.length === 0) {
      return { kind: 'refusal', reason: 'no-context' } as const;
    }

    const context = chunks.map((c, i) => `[${i + 1}] ${c.source} – ${c.heading}\n${c.text}`).join('\n\n');
    const citations: Citation[] = chunks.map(({ source, heading }) => ({ source, heading }));

    return {
      kind: 'stream',
      tokens: deps.llm.stream([
        { role: 'system', content: `Answer only from the context. Cite sources as [n].\n\n${context}` },
        { role: 'user', content: question },
      ]),
      citations,
    } as const;
  };
```

### 3.4 Controller: validation, orchestration, presenter

```ts
// lib/controllers/chat.controller.ts
import { z } from 'zod';
import { InputParseError, RateLimitedError } from '@/lib/entities/errors';
import { MAX_QUESTION_LENGTH } from '@/lib/entities/models';
import type { RateLimiter } from '@/lib/ports/rate-limiter';
import type { CheckGuardrailsUseCase } from '@/lib/use-cases/check-guardrails';
import type { AnswerQuestionUseCase } from '@/lib/use-cases/answer-question';

const inputSchema = z.object({
  question: z.string().trim().min(1).max(MAX_QUESTION_LENGTH),
});

export type ChatController = ReturnType<typeof chatController>;

export const chatController =
  (deps: {
    rateLimiter: RateLimiter;
    checkGuardrails: CheckGuardrailsUseCase;
    answerQuestion: AnswerQuestionUseCase;
  }) =>
  async (body: unknown, clientId: string) => {
    const limit = await deps.rateLimiter.consume(clientId);
    if (!limit.allowed) throw new RateLimitedError(limit.retryAfterSeconds);

    const parsed = inputSchema.safeParse(body);
    if (!parsed.success) throw new InputParseError('Invalid question', { cause: parsed.error });

    const verdict = await deps.checkGuardrails(parsed.data.question);
    if (!verdict.ok) return { kind: 'refusal', reason: verdict.reason } as const;

    return deps.answerQuestion(parsed.data.question);
  };
```

### 3.5 Adapter: wraps the SDK and converts its errors

```ts
// lib/adapters/gemini.llm.ts
import { GoogleGenerativeAI } from '@google/generative-ai';
import { ProviderUnavailableError } from '@/lib/entities/errors';
import type { LlmMessage, LlmProvider } from '@/lib/ports/llm-provider';

export function createGeminiLlm(apiKey: string, model = 'gemini-2.0-flash'): LlmProvider {
  const client = new GoogleGenerativeAI(apiKey).getGenerativeModel({ model });

  return {
    name: 'gemini',
    async *stream(messages: LlmMessage[]) {
      try {
        const system = messages.find((m) => m.role === 'system')?.content;
        const result = await client.generateContentStream({
          systemInstruction: system,
          contents: messages
            .filter((m) => m.role !== 'system')
            .map((m) => ({ role: m.role === 'assistant' ? 'model' : 'user', parts: [{ text: m.content }] })),
        });
        for await (const chunk of result.stream) yield chunk.text();
      } catch (err) {
        throw new ProviderUnavailableError('gemini', { cause: err });
      }
    },
  };
}
```

### 3.6 In-memory fake for tests

```ts
// lib/adapters/fakes/in-memory.vector-store.ts
import type { VectorStore } from '@/lib/ports/vector-store';
import type { KbChunk, ScoredChunk } from '@/lib/entities/models';

export function createInMemoryVectorStore(seed: ScoredChunk[] = []): VectorStore & { chunks: ScoredChunk[] } {
  const chunks = [...seed];
  return {
    chunks,
    async upsert(items: (KbChunk & { embedding: number[] })[]) {
      for (const { embedding: _e, ...chunk } of items) chunks.push({ ...chunk, score: 1 });
    },
    async query(_embedding, topK) {
      return [...chunks].sort((a, b) => b.score - a.score).slice(0, topK);
    },
  };
}
```

### 3.7 Composition root (no DI library needed)

```ts
// lib/container.ts
import 'server-only';
import { createGeminiLlm } from '@/lib/adapters/gemini.llm';
import { createUpstashVectorStore } from '@/lib/adapters/upstash.vector-store';
import { createUpstashRateLimiter } from '@/lib/adapters/upstash.rate-limiter';
import { createGeminiEmbeddings } from '@/lib/adapters/gemini.embeddings';
import { answerQuestionUseCase } from '@/lib/use-cases/answer-question';
import { checkGuardrailsUseCase } from '@/lib/use-cases/check-guardrails';
import { chatController } from '@/lib/controllers/chat.controller';

function env(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing env var ${name}`);
  return value;
}

let cached: ReturnType<typeof build> | undefined;

function build() {
  const llm = createGeminiLlm(env('GEMINI_API_KEY'));
  const embeddings = createGeminiEmbeddings(env('GEMINI_API_KEY'));
  const store = createUpstashVectorStore(env('UPSTASH_VECTOR_REST_URL'), env('UPSTASH_VECTOR_REST_TOKEN'));
  const rateLimiter = createUpstashRateLimiter(env('UPSTASH_REDIS_REST_URL'), env('UPSTASH_REDIS_REST_TOKEN'));

  return {
    chatController: chatController({
      rateLimiter,
      checkGuardrails: checkGuardrailsUseCase({ llm }),
      answerQuestion: answerQuestionUseCase({ llm, embeddings, store }),
    }),
  };
}

export function getContainer() {
  return (cached ??= build());
}
```

Tests never touch `container.ts`: they build use cases and controllers directly with fakes.

### 3.8 Route handler: maps domain errors to HTTP

```ts
// app/api/chat/route.ts
import { getContainer } from '@/lib/container';
import { InputParseError, ProviderUnavailableError, RateLimitedError } from '@/lib/entities/errors';

export async function POST(req: Request) {
  const clientId = req.headers.get('x-forwarded-for')?.split(',')[0]?.trim() ?? 'anonymous';
  try {
    const result = await getContainer().chatController(await req.json(), clientId);
    if (result.kind === 'refusal') return Response.json(result);
    return streamAnswer(result.tokens, result.citations); // presenter: tokens + citations only
  } catch (err) {
    if (err instanceof RateLimitedError)
      return Response.json({ error: 'rate_limited' }, { status: 429, headers: { 'Retry-After': String(err.retryAfterSeconds) } });
    if (err instanceof InputParseError) return Response.json({ error: 'invalid_input' }, { status: 400 });
    if (err instanceof ProviderUnavailableError) return Response.json({ error: 'unavailable' }, { status: 503 });
    throw err;
  }
}
```

### 3.9 Unit test with fakes

```ts
// tests/unit/use-cases/answer-question.test.ts
import { describe, expect, it } from 'vitest';
import { answerQuestionUseCase } from '@/lib/use-cases/answer-question';
import { createInMemoryVectorStore } from '@/lib/adapters/fakes/in-memory.vector-store';
import { createFakeLlm } from '@/lib/adapters/fakes/fake.llm';
import { createFakeEmbeddings } from '@/lib/adapters/fakes/fake.embeddings';

describe('answerQuestion', () => {
  it('refuses when no chunk is relevant enough', async () => {
    const store = createInMemoryVectorStore([
      { id: '1', source: 'kb/bio.md', heading: 'Hobbies', text: 'Chess', score: 0.2 },
    ]);
    const answer = answerQuestionUseCase({ llm: createFakeLlm('x'), embeddings: createFakeEmbeddings(), store });

    await expect(answer('What is the capital of France?')).resolves.toEqual({ kind: 'refusal', reason: 'no-context' });
  });

  it('returns citations for the chunks it used', async () => {
    const store = createInMemoryVectorStore([
      { id: '1', source: 'kb/cv.md', heading: 'Experience', text: '5 years TypeScript', score: 0.9 },
    ]);
    const answer = answerQuestionUseCase({ llm: createFakeLlm('Five years.'), embeddings: createFakeEmbeddings(), store });

    const result = await answer('How much TypeScript experience?');
    expect(result).toMatchObject({ kind: 'stream', citations: [{ source: 'kb/cv.md', heading: 'Experience' }] });
  });
});
```

## 4. Enforcing the rule: `eslint-plugin-boundaries`

CI fails if a file imports across layers it shouldn't. This is the main guardrail against agents drifting from the architecture.

```js
// eslint.config.mjs (excerpt)
import boundaries from 'eslint-plugin-boundaries';

export default [
  // ...next/typescript configs
  {
    plugins: { boundaries },
    settings: {
      'boundaries/include': ['app/**/*', 'components/**/*', 'lib/**/*'],
      'boundaries/elements': [
        { type: 'web', pattern: ['app/**/*', 'components/**/*'], mode: 'full' },
        { type: 'controllers', pattern: ['lib/controllers/**/*'], mode: 'full' },
        { type: 'use-cases', pattern: ['lib/use-cases/**/*'], mode: 'full' },
        { type: 'ports', pattern: ['lib/ports/**/*'], mode: 'full' },
        { type: 'entities', pattern: ['lib/entities/**/*'], mode: 'full' },
        { type: 'adapters', pattern: ['lib/adapters/**/*'], mode: 'full' },
        { type: 'container', pattern: ['lib/container.ts'], mode: 'full' },
      ],
    },
    rules: {
      'boundaries/no-unknown': 'error',
      'boundaries/no-unknown-files': 'error',
      'boundaries/element-types': ['error', {
        default: 'disallow',
        rules: [
          { from: 'web', allow: ['web', 'controllers', 'entities', 'container'] },
          { from: 'controllers', allow: ['use-cases', 'ports', 'entities'] },
          { from: 'use-cases', allow: ['ports', 'entities'] },
          { from: 'ports', allow: ['entities'] },
          { from: 'entities', allow: ['entities'] },
          { from: 'adapters', allow: ['adapters', 'ports', 'entities'] },
          { from: 'container', allow: ['adapters', 'use-cases', 'controllers', 'ports', 'entities'] },
        ],
      }],
    },
  },
];
```

`web` may import controllers only for types; runtime instances always come from `getContainer()`.

## 5. Be pragmatic: when *not* to layer

Apply the full layering only where it pays off. Rule of thumb: if a use case would be a pass-through wrapper, skip it.

| Criteria | Simple pattern (direct in `app/`) | Full layering (`lib/`) |
|---|---|---|
| Operation | Rendering static/markdown content | Calls an LLM, vector store or rate limiter |
| Business rules | None | Refusal thresholds, guardrails, fallback |
| Cost/abuse risk | None | Consumes free-tier quota |
| Testability | Playwright screenshot is enough | Needs unit tests with fakes |

- **Simple:** home page, project list read from `kb/` at build time, layout, theme toggle.
- **Full:** `/api/chat`, guardrails, semantic cache, retrieval, ingestion, model router, job-description fit analyzer.

## 6. Anti-patterns

**Core depends on a framework or SDK**

```ts
// ❌ lib/use-cases/answer-question.ts
import { GoogleGenerativeAI } from '@google/generative-ai'; // SDK in the core
```
```ts
// ✅ depend on the port; the SDK lives in lib/adapters/gemini.llm.ts
import type { LlmProvider } from '@/lib/ports/llm-provider';
```

**Use case creates its own dependencies**

```ts
// ❌
const store = createUpstashVectorStore(process.env.URL!, process.env.TOKEN!);
```
```ts
// ✅ injected through the factory
export const retrieveContextUseCase = (deps: { store: VectorStore }) => async (q: string) => { /* ... */ };
```

**Use case calls another use case**

```ts
// ❌ answerQuestion calls checkGuardrails internally
```
```ts
// ✅ the controller calls checkGuardrails, then answerQuestion
```

**SDK errors leak upward**

```ts
// ❌ route handler checks err.status === 429 from the Gemini SDK
// ✅ adapter throws ProviderUnavailableError; router/handler checks instanceof
```

**Route handler holds business logic**

```ts
// ❌ app/api/chat/route.ts builds prompts and filters chunks
// ✅ it only parses the request, calls the controller and maps errors to HTTP
```

## 7. UI guidelines (shadcn/ui + Tailwind)

- Use shadcn/ui primitives from `components/ui/`; compose rather than fork them. Add with `npx shadcn@latest add <component>`.
- **Design tokens only** (`bg-background`, `text-foreground`, `bg-primary`, `ring-ring`, `text-muted-foreground`); never hex/rgb or raw Tailwind palette colours. Tokens are defined in `app/globals.css` and documented in `specs/DESIGN.md`.
- Dark mode through the same tokens (`.dark` class); no `dark:` overrides with hard-coded colours.
- Server Components by default; `'use client'` only for interactive pieces (chat widget, theme toggle).
- Variants with `cva` instead of conditional class strings.

### Accessibility checklist

- Every input has a visible or `sr-only` label; icon buttons have `aria-label`.
- All interactive elements work with keyboard; visible focus (`focus-visible:ring-2 focus-visible:ring-ring`).
- The streaming answer area uses `aria-live="polite"` and `aria-busy` while loading, so screen readers announce answers.
- Chat widget: `Esc` closes it, focus moves into it on open and back to the trigger on close.
- Respect `prefers-reduced-motion` for typing/loading animations.

```tsx
// app/components/chat/answer.tsx
'use client';

export function Answer({ text, loading }: { text: string; loading: boolean }) {
  return (
    <div aria-live="polite" aria-busy={loading} className="rounded-lg bg-muted p-3 text-foreground">
      {loading && !text ? <span className="text-muted-foreground motion-safe:animate-pulse">Thinking…</span> : text}
    </div>
  );
}
```

## 8. Testing guidelines

- **Vitest** for unit tests; files in `tests/unit/` mirror `lib/`. The `@` alias must match `tsconfig.json`:

  ```ts
  // vitest.config.ts
  import { defineConfig } from 'vitest/config';
  import { fileURLToPath } from 'node:url';

  export default defineConfig({
    test: { environment: 'node', coverage: { provider: 'v8', include: ['lib/**'] } },
    resolve: { alias: { '@': fileURLToPath(new URL('./', import.meta.url)) } },
  });
  ```
- **Playwright** for e2e and screenshot checks at mobile (390px) and desktop (1280px); external APIs are mocked with `page.route`, so CI needs no keys.
- Test names describe behaviour, not implementation: `it('refuses when no chunk is relevant enough')`, grouped with `describe` per use case.
- Use fakes from `lib/adapters/fakes/`, not ad-hoc mocks. No `any` in tests; use the port types or `Partial<T>`.
- Every use case and controller gets unit tests for the happy path **and** each error/refusal branch.
- Coverage target for `lib/`: 80%+.

## 9. Review checklist (every PR)

- [ ] No boundary violations (`npm run lint` green).
- [ ] New external service = new port + adapter + fake.
- [ ] Inputs validated with Zod in the controller; errors are domain errors.
- [ ] Tests added for new behaviour and error branches.
- [ ] No hard-coded colours; tokens only.
- [ ] No secrets, keys or `.env` values in code or tests.
- [ ] One commit, branch `feat/<issue>-<slug>`, PR says `Closes #N`.
