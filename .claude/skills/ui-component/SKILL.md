---
name: ui-component
description: Build or change UI in portfolio-ai with Next.js App Router, shadcn/ui, Tailwind design tokens and accessibility rules. Use when creating or editing components, pages, the chat widget, theming or dark mode, or when implementing a screen from specs/design/.
---

# Building UI components

Design source of truth: `specs/DESIGN.md` (tokens, chat widget behaviour) and the reference screens in `specs/design/`. Rebuild from tokens; never paste exported design code as final.

## Rules

1. **Tokens only.** Use semantic classes (`bg-background`, `text-foreground`, `bg-primary`, `text-primary-foreground`, `bg-muted`, `text-muted-foreground`, `border-border`, `ring-ring`). No hex/rgb, no raw palette classes like `bg-blue-500`, no inline `style` colours. New tokens go in `app/globals.css` and `specs/DESIGN.md` together.
2. **Dark mode via tokens.** The `.dark` class swaps token values; components don't need `dark:` colour overrides.
3. **Server Components by default.** Add `'use client'` only to the smallest interactive piece (chat widget, theme toggle, input). Hooks (`useState`, `useEffect`) need `'use client'`.
4. **Don't pass functions from Server to Client Components.** Define handlers inside the client component, or call the API from it.
5. **shadcn/ui first.** Add primitives with `npx shadcn@latest add <name>` into `components/ui/`; compose them in `app/components/`. Don't edit `components/ui/` for one-off styling; pass `className`.
6. **Variants with `cva`**, merged with `cn()`; no long conditional class strings.
7. **Layering.** UI calls `/api/chat` or imports types from `lib/entities`; it never imports `lib/adapters` or use cases (see `specs/ARCHITECTURE.md`).

## Accessibility (required)

- Every input has a label (visible or `sr-only`); icon-only buttons have `aria-label`.
- Keyboard: everything reachable with Tab; visible focus with `focus-visible:ring-2 focus-visible:ring-ring`.
- Chat widget: opening moves focus to the input; `Esc` closes it and returns focus to the trigger (use shadcn `Sheet`/`Dialog`, which handle this).
- Streaming answers sit in an `aria-live="polite"` region with `aria-busy` while loading.
- Errors are announced (`role="alert"`) and say what to do next ("Too many questions, try again in 30s").
- Respect `prefers-reduced-motion` (`motion-safe:` / `motion-reduce:` variants).
- Colour is never the only signal; citations are real links with readable text.

## Responsive

Mobile-first: base classes for 390px, then `md:` and `lg:`. On mobile the chat widget is full-screen; on desktop a panel (exact behaviour in `specs/DESIGN.md`). Check both widths with the Playwright screenshot projects.

## Example

```tsx
// app/components/chat/chat-input.tsx
'use client';

import { useState } from 'react';
import { Button } from '@/components/ui/button';
import { Textarea } from '@/components/ui/textarea';
import { MAX_QUESTION_LENGTH } from '@/lib/entities/models';

export function ChatInput({ onSend, disabled }: { onSend: (q: string) => void; disabled: boolean }) {
  const [value, setValue] = useState('');

  return (
    <form
      className="flex items-end gap-2 border-t border-border bg-background p-3"
      onSubmit={(e) => {
        e.preventDefault();
        if (value.trim()) onSend(value.trim());
        setValue('');
      }}
    >
      <label htmlFor="chat-question" className="sr-only">Ask a question</label>
      <Textarea
        id="chat-question"
        value={value}
        maxLength={MAX_QUESTION_LENGTH}
        onChange={(e) => setValue(e.target.value)}
        className="min-h-10 resize-none"
        placeholder="Ask about my experience…"
      />
      <Button type="submit" disabled={disabled || !value.trim()} aria-label="Send question">
        Send
      </Button>
    </form>
  );
}
```

## Done when

- Lint, typecheck and Playwright screenshots at both widths are green.
- No hard-coded colours (`grep -rE '#[0-9a-fA-F]{3,6}|rgb\(' app components` finds nothing new outside `globals.css`).
