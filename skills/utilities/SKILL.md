---
name: utilities
description: Generate utility functions and helpers for Debrief. Use when creating formatters, validators, data transformers, de-identification helpers, or any pure functions.
---

# Utilities

Generate small pure functions / helpers and place them in the correct package.

## Where Utilities Live

| Kind | Location | Examples |
|------|----------|----------|
| Browser-only helpers | `apps/web/src/lib/` (camelCase) | `fetchApi.ts`, `audioSession.ts`, `consultSlots.ts`, `refLabel.ts`, `pendingRedirect.ts`, `utils.ts` (`cn`) |
| API-only helpers / business logic | `apps/api/src/lib/` (camelCase) | `reflectHelpers.ts`, `sessionContext.ts`, `resolveDoctor.ts`, `textSanitise.ts`, `firstSentence.ts`, `sanitizeUrls.ts` |
| AWS / external clients | `apps/api/src/services/` | see [creating-services](../creating-services/SKILL.md) |
| Cross-package types | `apps/shared/src/types.ts` (re-exported from `@debrief/shared`) | `Doctor`, `Consultation`, `ReflectRequest`, … |
| Cross-package route + validation constants | `apps/shared/src/constants/` | `route.ts` (`API_ROUTE`), `validationMessage.ts` (`VALIDATION_MESSAGE`) |
| Cross-package Zod schemas | `apps/shared/src/schemas/` | `auth.ts`, `public.ts` |

There is **no `apps/shared/src/constants.ts`** file — constants live in `apps/shared/src/constants/<name>.ts` and are barrel-exported from `apps/shared/src/index.ts`.

## Key Existing Helpers

### `cn` — class name merger

Lives in [apps/web/src/lib/utils.ts](file:///Users/sajinthanjanahiram/Projects/debrief-demo/apps/web/src/lib/utils.ts). Merges class names with Tailwind conflict resolution:

```typescript
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

### `fetchApi` — authenticated JSON fetch wrapper

Lives in [apps/web/src/lib/fetchApi.ts](file:///Users/sajinthanjanahiram/Projects/debrief-demo/apps/web/src/lib/fetchApi.ts). Always use this from the SPA — it sets `credentials: 'include'`, parses JSON, and throws on non-2xx.

### De-identification (critical)

All free text destined for the AI **must** pass through `apps/api/src/services/deidentify.ts` (Amazon Comprehend Medical `DetectPHI`). Two helpers:

```typescript
import { deidentifyText, deidentifyConsultation } from '../services/deidentify';
```

## Conventions

- **Pure functions** — no side effects unless explicitly a "service".
- **TypeScript strict** — no `any`. Type every parameter and return value.
- **ESM relative imports** in `apps/api` use the `.js` extension (e.g., `from './lib/prisma.js'`); web does not.
- **Named exports** for helpers; default exports only for hooks and React components.
- **Shared types first** — if a type crosses the API/web boundary, define it in `apps/shared/src/types.ts` and import via `@debrief/shared`.
- **Use existing constants** — `API_ROUTE`, `VALIDATION_MESSAGE`, and the API-side `API_VALIDATION` (see [creating-api-routes](../creating-api-routes/SKILL.md)) before inventing new strings.

## Testing

Tests live in `apps/<pkg>/src/__tests__/<name>.test.ts` — **not** co-located beside source files. See [testing](../testing/SKILL.md).

## Checklist

- [ ] Function is pure (or clearly marked as a service with side effects)
- [ ] Placed in the correct package (`web/src/lib/`, `api/src/lib/`, `api/src/services/`, or `shared/src/`)
- [ ] Cross-package types defined in `@debrief/shared` first
- [ ] No hard-coded `/api/...` paths — uses `API_ROUTE`
- [ ] No hard-coded validation copy — uses `VALIDATION_MESSAGE` / `API_VALIDATION`
- [ ] De-identification applied before any AI/Bedrock call
- [ ] `.js` extension on relative imports inside `apps/api`
