---
name: shared-validation
description: Adds or updates Zod validation schemas and validation copy in @repo/shared, apps/api and apps/web. Use when introducing a new request body, new form, or new validation message that crosses (or might cross) the API ↔ web boundary.
---

# Shared Validation

A Zod schema can live in three places and validation copy in three constants. Pick the right one before you start writing.

## Where schemas live

| Location | When to use | Examples |
|----------|-------------|----------|
| `packages/shared/src/schemas/` | Consumed by **both** the API (Fastify body validation) and the web app (react-hook-form `zodResolver`). | `signInSchema`, `signUpSchema`, `createProjectSchema`, `createTaskSchema` |
| `apps/api/src/schemas/index.ts` | Consumed **only** by Fastify; no form mirrors it 1:1. | `updateTaskStatusSchema`, `inviteMemberSchema` |
| `apps/web/src/schemas/<formName>.ts` | Form-only, or the form needs `.refine(...)` cross-field rules the API doesn't apply. | `projectSettings`, `confirmDelete` |

Prefer the shared tier whenever the same shape exists on both sides. One schema stops the API and the form from drifting.

## Where validation copy lives

| Constant | Location | Use for |
|----------|----------|---------|
| `VALIDATION_MESSAGE` | `packages/shared/src/constants/validationMessage.ts` | Form-friendly copy (`"Email is required"`, `"Title must be 200 characters or fewer"`). Used by every shared schema and most form-only schemas. |
| `FORM_VALIDATION` | `apps/web/src/constants/message.ts` | Web-only copy that doesn't belong in the API. |
| `API_VALIDATION` | `apps/api/src/constants/apiValidation.ts` | API-only copy for 400 responses on fields with no form analogue (`"taskId is required"`). |

**Never inline validation strings in a schema.** Add the message to the right constant, import it, then reference it.

## Shared schema pattern

```typescript
// packages/shared/src/schemas/task.ts
import { z } from 'zod';

import { VALIDATION_MESSAGE } from '../constants/validationMessage';

export const TASK_STATUS = ['todo', 'in_progress', 'done'] as const;

export const createTaskSchema = z.object({
  title: z
    .string()
    .min(1, VALIDATION_MESSAGE.TITLE_REQUIRED)
    .max(200, VALIDATION_MESSAGE.TITLE_MAX_LENGTH),
  status: z.enum(TASK_STATUS).default('todo'),
});
```

Re-export it from `packages/shared/src/index.ts` (`export * from './schemas/task';`) and use it on both sides:

```typescript
// API (apps/api/src/routes/tasks.ts)
import { API_ROUTE, createTaskSchema } from '@repo/shared';

zod.post(API_ROUTE.TASKS, { schema: { body: createTaskSchema } }, handler);

// Web (apps/web/src/pages/Tasks/NewTask.tsx)
import { createTaskSchema } from '@repo/shared';

const form = useForm({ resolver: zodResolver(createTaskSchema), defaultValues });
```

## API-only schema pattern

```typescript
// apps/api/src/schemas/index.ts
import { z } from 'zod';

import { TASK_STATUS, VALIDATION_MESSAGE } from '@repo/shared';

import { API_VALIDATION } from '../constants/apiValidation';

export const updateTaskStatusSchema = z.object({
  taskId: z.string().min(1, API_VALIDATION.TASK_ID_REQUIRED),
  status: z.enum(TASK_STATUS),
});

export const inviteMemberSchema = z.object({
  email: z
    .string()
    .min(1, VALIDATION_MESSAGE.EMAIL_REQUIRED)
    .email(VALIDATION_MESSAGE.EMAIL_INVALID)
    .max(255, VALIDATION_MESSAGE.EMAIL_MAX_LENGTH),
  role: z.enum(['ADMIN', 'MEMBER']).default('MEMBER'),
  expiresAt: z.string().datetime({ offset: true }).nullable().optional(),
});
```

API schemas can mix `API_VALIDATION` (API-only errors) and `VALIDATION_MESSAGE` (when the form already has the same copy).

## Web-only schema pattern

See [[creating-forms]]. Web-only schemas live in `apps/web/src/schemas/<formName>.ts` and export `formSchema`, `XxxFormValues` and `defaultValues`. They use `VALIDATION_MESSAGE` (preferred) or `FORM_VALIDATION`.

## Workflow

1. Pick the tier (shared, API-only, web-only) from the tables above.
2. Add any new copy to the right constant (`VALIDATION_MESSAGE`, `FORM_VALIDATION` or `API_VALIDATION`).
3. Write the schema, importing the copy. No inline strings.
4. If shared, barrel-export it from `packages/shared/src/index.ts`.
5. Wire it into the API ([[creating-api-routes]]) and/or the form ([[creating-forms]]).

## Checklist

- [ ] Schema is in the lowest tier both consumers can reach
- [ ] Every `min`, `max`, `email`, `regex` message references a constant
- [ ] Shared schemas re-exported from `packages/shared/src/index.ts`
- [ ] API handler uses `zod.<verb>(path, { schema: { body: schema } }, …)` (typed via `withTypeProvider<ZodTypeProvider>()`)
- [ ] Form uses `useForm({ resolver: zodResolver(schema), defaultValues })`
