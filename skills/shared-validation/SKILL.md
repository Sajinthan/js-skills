---
name: shared-validation
description: Add or update Zod validation schemas and validation copy. Use when introducing a new request body, new form, or new validation message that crosses (or might cross) the API ↔ web boundary.
---

# Shared Validation

Debrief has three places a Zod schema can live and two places validation copy can live. Pick the right one before you start writing.

## Where Schemas Live

| Location | When to use | Examples |
|----------|-------------|----------|
| `apps/shared/src/schemas/` | Schema is consumed by **both** the API (Fastify body validation) and the web app (react-hook-form `zodResolver`). | `signInSchema`, `signUpSchema`, `forgotPasswordSchema`, `waitlistSchema`, `contactSchema`, `feedbackSchema` |
| `apps/api/src/schemas/index.ts` | Schema is **only** consumed by Fastify (no client form mirrors it 1:1). | `createSessionSchema`, `updateDoctorSchema`, `deidentifySchema`, `createSupervisorInviteSchema` |
| `apps/web/src/schemas/<formName>.ts` | Form-only schema with no API counterpart, or the form needs extra `.refine(...)` cross-field rules the API doesn't apply. | `customCase`, `onboardingProfile`, `verifyEmail` |

The "shared" tier is preferred whenever the same shape exists on both sides — keeping the schema in one place stops the API and the form from drifting.

## Where Validation Copy Lives

| Constant | Location | Audience |
|----------|----------|----------|
| `VALIDATION_MESSAGE` | [apps/shared/src/constants/validationMessage.ts](file:///Users/sajinthanjanahiram/Projects/debrief-demo/apps/shared/src/constants/validationMessage.ts) | Form-friendly copy (`"Email is required"`, `"Password must be at least 12 characters"`). Used by every shared schema and most form-only schemas. |
| `FORM_VALIDATION` | [apps/web/src/constants/message.ts](file:///Users/sajinthanjanahiram/Projects/debrief-demo/apps/web/src/constants/message.ts) | Web-only form copy that doesn't make sense to ship to the API. |
| `API_VALIDATION` | [apps/api/src/constants/apiValidation.ts](file:///Users/sajinthanjanahiram/Projects/debrief-demo/apps/api/src/constants/apiValidation.ts) | API-only error copy returned in 400 responses for fields with no form analogue (e.g., `"consultationId is required"`). |

**Never inline validation strings inside a schema.** Add the message to the right enum first, import it, then reference it.

## Shared Schema Pattern

```typescript
// apps/shared/src/schemas/auth.ts
import { z } from 'zod';

import { VALIDATION_MESSAGE } from '../constants/validationMessage';

export const signInSchema = z.object({
  email: z
    .string()
    .min(1, VALIDATION_MESSAGE.EMAIL_REQUIRED)
    .email(VALIDATION_MESSAGE.EMAIL_INVALID),
  password: z.string().min(1, VALIDATION_MESSAGE.PASSWORD_REQUIRED),
});
```

The schema is then re-exported from `apps/shared/src/index.ts` (`export * from './schemas/auth';`) and used on both sides:

```typescript
// API (apps/api/src/routes/auth.ts)
import { signInSchema } from '@debrief/shared';

zod.post(API_ROUTE.AUTH_SIGNIN, { schema: { body: signInSchema } }, handler);

// Web (apps/web/src/pages/SignIn/index.tsx)
import { signInSchema } from '@debrief/shared';

const form = useForm({ resolver: zodResolver(signInSchema), defaultValues });
```

## API-only Schema Pattern

```typescript
// apps/api/src/schemas/index.ts
import { z } from 'zod';

import { VALIDATION_MESSAGE } from '@debrief/shared';

import { API_VALIDATION } from '../constants/apiValidation';

export const createSessionSchema = z.object({
  consultationId: z.string().min(1, API_VALIDATION.CONSULTATION_ID_REQUIRED),
  source: z.enum(['halo', 'library', 'custom']).default('halo'),
});

export const createSupervisorInviteSchema = z.object({
  supervisorEmail: z
    .string()
    .min(1, VALIDATION_MESSAGE.EMAIL_REQUIRED)
    .email(VALIDATION_MESSAGE.EMAIL_INVALID)
    .max(255, VALIDATION_MESSAGE.EMAIL_MAX_LENGTH),
  kind: z.enum(['SUPERVISOR', 'EDUCATOR']).default('SUPERVISOR'),
  expiresAt: z.string().datetime({ offset: true }).nullable().optional(),
});
```

API schemas can mix `API_VALIDATION` (API-specific errors) and `VALIDATION_MESSAGE` (when the same copy already exists for forms).

## Web-only Schema Pattern

See [creating-forms](../creating-forms/SKILL.md). Web-only schemas live in `apps/web/src/schemas/<formName>.ts` and export `formSchema`, `XxxFormValues`, and `defaultValues`. They use `VALIDATION_MESSAGE` (preferred) or `FORM_VALIDATION`.

## Workflow

1. Decide the tier (shared vs api-only vs web-only) using the table above.
2. Add any new copy to the right enum (`VALIDATION_MESSAGE`, `FORM_VALIDATION`, or `API_VALIDATION`).
3. Write the schema, importing from the message enum — never inline strings.
4. If shared, ensure it's barrel-exported from [apps/shared/src/index.ts](file:///Users/sajinthanjanahiram/Projects/debrief-demo/apps/shared/src/index.ts).
5. Wire it into the API ([creating-api-routes](../creating-api-routes/SKILL.md)) and/or the form ([creating-forms](../creating-forms/SKILL.md)).

## Checklist

- [ ] Schema is in the lowest-shared tier that both consumers can reach
- [ ] All `min`, `email`, `regex`, etc. messages reference an enum constant
- [ ] Shared schemas re-exported from `apps/shared/src/index.ts`
- [ ] API handler uses `zod.<verb>(path, { schema: { body: schema } }, …)` (typed via `withTypeProvider<ZodTypeProvider>()`)
- [ ] Form uses `useForm({ resolver: zodResolver(schema), defaultValues })`
