# Component Guidelines

> How components are built in Piyaz. Pattern references: `components/workspace/project-settings/TitleSection.tsx`, `components/auth/SignInForm.tsx`.

---

## Component Structure

```tsx
"use client";

import { useCallback, useState } from "react";
import { updateProjectSettings } from "@/lib/actions/project";

interface TitleSectionProps {
  projectId: string;
  initialTitle: string;
  onUpdated?: () => void;
}

/**
 * Click-to-edit title input that persists on blur.
 * @param props - Section props.
 * @returns Title row.
 */
export function TitleSection({ projectId, initialTitle, onUpdated }: TitleSectionProps) { ... }
```

- `"use client"` on interactive components; server components are the default in `app/`.
- One named export per file; JSDoc block on the component describing behavior (every existing component has one).
- Props destructured in the signature, defaults inline (`next = null` in `SignInForm`).

---

## Props Conventions

- Props are a local `interface XxxProps` above the component, each field with a `/** ... */` doc comment explaining intent (see `SignInFormProps`).
- Optional callbacks use the `on<Event>?: () => void` shape (`onUpdated?`).
- Nullable capability flags are explicit (`turnstileSiteKey?: string | null`, where `null` means "disabled on self-host").

---

## Forms

Two form patterns exist; pick by backend target:

1. **Server-action forms** (settings, team management): controlled inputs, call the action directly, branch on its typed result, render `result.message` inline.

   ```tsx
   const result = await updateProjectSettings(projectId, { title: trimmed });
   if (!result.ok) { setServerError(result.message); return; }
   onUpdated?.();
   ```

   Error display is a small inline `<p className="... text-danger">` under the field (`TitleSection.tsx:100`). No toast library, no form library (no react-hook-form).

2. **Auth forms** (`components/auth/*Form.tsx`): controlled `useState` fields, `handleSubmit(event: FormEvent<HTMLFormElement>)`, submit through the better-auth client (`signIn.email` from `@/lib/auth-client`), local `error` + `loading` state, optional Turnstile via `TurnstileGate` / `useTurnstile`.

Inline click-to-edit fields use `useInlineEdit` from `hooks/useInlineEdit.ts` and commit on blur, `Enter` to commit, `Escape` to revert (`TitleSection.tsx:68-84`).

---

## Styling Patterns

- Tailwind v4 utility classes with semantic design tokens from `app/globals.css`: `text-text-primary`, `text-text-muted`, `bg-surface-hover`, `border-border-strong`, `text-danger`, `focus:border-accent`. Do not hard-code hex colors.
- Repeated class strings are hoisted to a module constant (`SECTION_LABEL_CLASS` in `TitleSection.tsx`).
- No CSS modules, no styled-components, no inline `style` objects for themable values.

---

## Accessibility

- Focus is managed explicitly on edit affordances: `focus-visible:ring` classes and `autoFocus` when the editor opens.
- Keyboard paths mirror pointer paths (`Enter` commits, `Escape` cancels).
- Icon-only buttons carry a `label` prop that becomes their accessible name (`EditButton onClick={...} label="Edit title"`).

---

## Common Mistakes

- Deriving state in `useEffect` when a render-time sync works: `TitleSection` syncs `initialTitle` prop changes with a render-phase comparison (`if (initialTitle !== syncedInitialTitle && !editing)`), not an effect.
- Throwing from server actions toward the UI; actions return `{ ok: false, message }` and components must handle that branch.
