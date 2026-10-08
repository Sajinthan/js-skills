---
name: creating-forms
description: Builds dashboard forms in apps/web with react-hook-form, zodResolver and the Form primitives in components/ui/form.tsx. Use whenever adding or editing a form in the dashboard (sign-in, sign-up, settings, dialogs that collect input) so schemas, validation copy and form wiring follow one pattern.
---

# Creating Forms

_Rewritten 2026-09-27 for Goodo from Debrief's version, when `components/ui/form.tsx` landed. The pattern is Debrief's (debrief-demo `apps/web/src/components/ui/form.tsx`); the paths are Goodo's. Screens built before it (the assistant modals, settings, web chat) still use `useState` per field: move them over when they are next touched._

Every dashboard form is **react-hook-form** + **Zod** (`zodResolver`) + the `Form` primitives. No `useState` per field, no hand-built label and error wrappers.

## Where things live

| Piece | Location |
|---|---|
| Schema the API also validates | `packages/shared/src/schemas/<name>.ts` ([[shared-validation]]) |
| Schema only the form needs (extends a shared one, cross-field `.refine`) | `apps/web/src/schemas/<formName>.ts` |
| Validation copy | `VALIDATION_MESSAGE` in `packages/shared/src/constants/validationMessage.ts`; never a string inside a schema |
| Form primitives | `apps/web/src/components/ui/form.tsx`: `Form`, `FormField`, `FormItem`, `FormLabel`, `FormControl`, `FormDescription`, `FormMessage` |
| Inputs | `apps/web/src/components/ui/input.tsx` (`Input`); any other control works inside `FormControl` |

Most Goodo forms use the shared schema directly: `zodResolver(signUpSchema)`. Add a web schema file only when the form checks something the API doesn't.

## Pattern

```tsx
import { zodResolver } from '@hookform/resolvers/zod';
import { useForm } from 'react-hook-form';

import { signUpSchema, type SignUp } from '@goodo/shared';

import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from '@/components/ui/form';
import { Input } from '@/components/ui/input';

export const SignUpForm = ({ onCreated }: SignUpFormProps) => {
  const form = useForm<SignUp>({ resolver: zodResolver(signUpSchema), defaultValues: { email: '', password: '' } });

  const onSubmit = async (values: SignUp) => {
    // call the API; field problems → form.setError, anything else → a submit error above the button
  };

  return (
    <Form {...form}>
      <form noValidate onSubmit={form.handleSubmit(onSubmit)} className="flex flex-col gap-4">
        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl>
                <Input type="email" autoComplete="email" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
      </form>
    </Form>
  );
};
```

## Rules

- Type `useForm<Values>` with the schema's output type; never let it infer `any`.
- `noValidate` on every `<form>`, so the browser doesn't pre-empt Zod.
- `FormMessage` shows only the field's schema error. An API error about one field (an email already taken) goes in with `form.setError('email', { message })`; any other API error shows above the submit button.
- Disable the submit button with `form.formState.isSubmitting`, not a separate `sending` state.
- A form that is one step of a screen (sign-up's form, then its code step) is its own component, so each stays small ([fallow](../../../CLAUDE.md) flags complexity over 15).
- Components are `const` arrow functions ([[coding-style]]).

## Checklist

- [ ] Schema from `@goodo/shared`, or `apps/web/src/schemas/` only for form-only rules
- [ ] Copy in `VALIDATION_MESSAGE`, none inline
- [ ] `useForm` typed, `zodResolver`, `noValidate`, every field a `FormField`
- [ ] API errors: field ones through `setError`, the rest above the button
- [ ] Tests type into the labelled fields and assert the schema's messages
