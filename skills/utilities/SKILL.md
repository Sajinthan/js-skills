---
name: utilities
description: Writes small pure helpers (formatters, validators, data transformers) and places them in the right app's src/lib/ or the package that owns them, such as @repo/shared. Use when creating formatters, validators, data transformers, or any pure function.
---

# Utilities

Write small pure functions and put them in the package that owns them.

## Where utilities live

| Kind | Location | Examples |
|------|----------|----------|
| Browser-only helpers | `apps/web/src/lib/` | `utils.ts` (`cn`), `fetch-api.ts`, `format-date.ts`, `task-sort.ts` |
| API-only helpers and business logic | `apps/api/src/lib/` | `slugify.ts`, `paginate.ts`, `task-status.ts` |
| Helpers both sides need | `packages/shared/src/lib/` | `format-cents.ts`, `is-overdue.ts` |
| External clients | `apps/api/src/services/` | see [[creating-services]] |
| Cross-package types | `packages/shared/src/types.ts` | `Project`, `Task`, `TaskStatus` |
| Cross-package constants | `packages/shared/src/constants/<name>.ts` | `route.ts` (`API_ROUTE`), `validationMessage.ts` (`VALIDATION_MESSAGE`) |
| Cross-package Zod schemas | `packages/shared/src/schemas/` | see [[shared-validation]] |

New helper files are `kebab-case.ts`. Everything in `@repo/shared` is barrel-exported from `packages/shared/src/index.ts`; keep it browser-safe (zod only).

## Existing helpers

### `cn`: class name merger

In `apps/web/src/lib/utils.ts`. Merges class names with Tailwind conflict resolution:

```typescript
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export const cn = (...inputs: ClassValue[]) => twMerge(clsx(inputs));
```

### `fetchApi`: authenticated JSON fetch

In `apps/web/src/lib/fetch-api.ts`. Always use it from the dashboard: it sets `credentials: 'include'`, parses JSON and throws on non-2xx.

## Example

```typescript
// packages/shared/src/lib/format-cents.ts
export const formatCents = (cents: number): string => {
  const sign = cents < 0 ? '-' : '';

  return `${sign}$${(Math.abs(cents) / 100).toFixed(2)}`;
};
```

## Conventions

- **Pure functions**: no side effects. Anything with I/O is a service.
- **TypeScript strict**: no `any`. Type every parameter and return value.
- **Named exports** only.
- **Shared types first**: if a type crosses the API ↔ web boundary, define it in `@repo/shared`.
- **Use existing constants**: `API_ROUTE`, `VALIDATION_MESSAGE` and the API-side `API_VALIDATION` before inventing new strings.

## Testing

Tests live in `__tests__/` next to the helper: `src/lib/__tests__/<name>.test.ts`. Cover edge cases with `it.each`. See [[testing]].

## Checklist

- [ ] Function is pure (or it's a service in `services/`)
- [ ] In the right place: `web/src/lib/`, `api/src/lib/`, `api/src/services/` or `packages/shared/src/`
- [ ] File name is `kebab-case.ts`, named export
- [ ] Cross-package types defined in `@repo/shared` first
- [ ] No hard-coded `/api/...` paths; uses `API_ROUTE`
- [ ] No hard-coded validation copy; uses `VALIDATION_MESSAGE` / `API_VALIDATION`
- [ ] `.js` extension on relative imports inside `apps/api`
- [ ] Test in `__tests__/` next to the helper
