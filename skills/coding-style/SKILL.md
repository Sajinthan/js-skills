---
name: coding-style
description: Enforces formatting, whitespace, control flow, import order, naming and TypeScript conventions for the Goodo monorepo (Fastify services, React dashboard, Next.js site, Preact widget). Always active when writing or editing code.
---

# Coding Style

_Adapted 2026-09-23 from rn-boilerplate `coding-style`, merged with Debrief's `blank-line-spacing`, `curly-braces` and `import-ordering`. Written before the code exists: paths are the planned ones in `docs/TASKS.md`. Run [[refine-skill]] once task 1.0 lands._

## Formatting (Prettier)

- `printWidth: 120`, `singleQuote: true`, `tabWidth: 2`, no tabs, semicolons
- `arrowParens: 'avoid'` → `value => value * 2`
- `bracketSameLine: false`
- `prettier-plugin-tailwindcss` sorts classes in `apps/site` and `apps/web`
- ESLint presets come from `packages/config`; never add a per-app override without a comment saying why

## Whitespace

Insert a **blank line** before `if`, `switch` and `return`, unless:

- the statement is the **first line** of a block (right after `{`);
- the `return` directly follows an `if` block (a blank line there is optional);
- a blank line is already there.

Never two blank lines in a row.

```ts
const summarise = (turns: Turn[]): string | null => {
  if (turns.length === 0) {
    return null;
  }

  const text = turns.map(turn => turn.text).join(' ');

  if (text.length < 20) {
    return text;
  }

  return `${text.slice(0, 200)}…`;
};
```

## Braces

- **Every** `if`, `else if`, `else`, `for` and `while` body takes braces, even a single statement, guards included: `if (!assistant) { return null; }` written over three lines, never `if (!assistant) return null;` (decided 2026-10-02). ESLint's `curly: 'all'` enforces it, and `eslint --fix` adds them.
- `{` on the same line as the keyword; `else` on the same line as the closing `}`.

## Declaration order

A function or component sits above everything that calls it, so a file reads top to bottom: helpers and child components first, the exported component or factory last. Inside a factory, the functions it defines follow the same order. ESLint enforces it (`@typescript-eslint/no-use-before-define` in `packages/config/eslint.js`); types are exempt.

Two values that need each other, such as a server and the job queue whose error handler logs through it, get a one-line `// eslint-disable-next-line @typescript-eslint/no-use-before-define -- <why>`. They are the only exception.

## Comments

**No comments by default** (the founder's rule: clean code explains itself). Before writing one, make the code say it: a clearer name, a named constant, a small function, a guard clause. A comment is allowed only when that's impossible: a workaround for someone else's bug, or an outside limit or trap a reader would otherwise get wrong. One line: `pnpm check:comments` fails two or more. When in doubt, leave it out.

- Never restate a name, a type or what the next line plainly does.
- Constants, types, interface fields, options, env schema entries and function summaries get no comment: a field named `timeoutSeconds` explains itself. A field comment only for a trap, such as a limit that is best effort, and it starts `Trap:` (the check fails any other).
- No dates, decisions or task numbers in code. Those go in commits, PRs and `docs/TASKS.md`.

```ts
// Yes: a trap the code can't show
// createQueue leaves an existing queue's settings alone, and a stored schedule outlives the code that made it.

// No: says what the name already says
// Stops taking jobs and waits for the running ones to finish.
const stop = async () => { … };
```

## Control flow

Bail out with a **guard clause**. Bad input, nothing to do, or a state with no answer returns on the spot, so the real body stays at one indent level and is never wrapped in `else`.

**Never `await` inside an `if` condition.** Await into a named `const`, then test the name, so the condition reads as a plain value (decided 2026-10-02; ESLint's `no-restricted-syntax` enforces it). An `await` in the `if`'s body is fine.

```ts
// Yes
const removed = await assistants.remove(accountId, assistantId);

if (!removed) {
  return reply.status(404).send({ error: MESSAGE.notFound });
}

// No
if (!(await assistants.remove(accountId, assistantId))) {
  return reply.status(404).send({ error: MESSAGE.notFound });
}
```

```ts
// Yes
export function formatCents(cents: number, { sign = false } = {}): string {
  const value = (Math.abs(cents) / 100).toFixed(2);

  if (cents < 0) {
    return `-$${value}`;
  }

  if (sign && cents > 0) {
    return `+$${value}`;
  }

  return `$${value}`;
}

// No — the ordinary case is buried and `else` carries no information
export function formatCents(cents: number, { sign = false } = {}): string {
  const value = (Math.abs(cents) / 100).toFixed(2);

  if (cents < 0) {
    return `-$${value}`;
  } else if (sign && cents > 0) {
    return `+$${value}`;
  } else {
    return `$${value}`;
  }
}
```

`else` is right only when both arms are **peers**: each does work and the function carries on afterwards.

**Never nest a ternary.** One `a ? b : c` is fine. A third arm is a mapping: write a `switch` in a named helper or a lookup table, one case per line, default last.

```ts
// Yes
const outcomeLabel = (outcome: ConversationOutcome): string => {
  switch (outcome) {
    case 'booking_request':
      return 'Booking request';
    case 'message_taken':
      return 'Message taken';
    case 'handed_off':
      return 'Handed to a person';
    default:
      return 'Answered';
  }
};
```

Checks (both should print nothing):

```bash
grep -rnE "\? .*\s+: .*[^.?]\? " apps packages --include="*.ts" --include="*.tsx" | grep -v __tests__
grep -rnE "^\s+: .*[^.?]\? " apps packages --include="*.ts" --include="*.tsx" | grep -v __tests__
```

## Import order

Groups separated by one blank line; omit empty groups:

1. **Node built-ins and third-party**: `node:path`, `fastify`, `react`, `zod`, `@prisma/client`
2. **Workspace packages**: `@goodo/shared`, `@goodo/engine`, `@goodo/db`, `@goodo/packs-restaurant`
3. **`@/` alias** inside the current app
4. **Relative** siblings in the same module folder
5. **Assets and CSS**

`import type` follows the group of its source.

```ts
import { z } from 'zod';

import { conversationOutcomeSchema } from '@goodo/shared';

import { cn } from '@/lib/utils';

import { OutcomeBadge } from './OutcomeBadge';

import './styles.css';
```

## Boundaries

Enforced by the lint rule from task 1.4. Breaking one is a lint error, not a style note.

| Code | May import |
|---|---|
| `apps/site`, `apps/web`, `apps/widget` | `@goodo/shared`, `@goodo/ui` |
| `apps/api`, `apps/chat`, `apps/voice`, `apps/worker` | `@goodo/engine`, `@goodo/db`, `@goodo/shared`, `packs/*` |
| `packages/engine` | `@goodo/shared`; data access only through interfaces passed in |
| `packages/shared` | `zod` only (it ships to browsers) |
| any app | never another app |

## Naming

| Kind | Convention | Example |
|---|---|---|
| React component | `PascalCase/` folder with `index.tsx` | `components/ConversationList/index.tsx` |
| Hook | `camelCase`, `use` prefix | `hooks/useCrawlProgress.ts` |
| Fastify route module | `kebab-case.ts` | `routes/widget-chat.ts` |
| Worker job | `kebab-case.ts`, named after the job | `jobs/crawl-source.ts` |
| Library file | `kebab-case.ts` | `lib/phone-number.ts` |
| Zod schema file | `camelCase.ts` | `schemas/bookingRequest.ts` |
| Test file | `<name>.test.ts(x)` in `__tests__/` next to the code | `lib/__tests__/phone-number.test.ts` |
| Constant value | `UPPER_SNAKE_CASE` | `MAX_PAGES_PER_CRAWL` |
| Database enum value | the vocabulary in `docs/TASKS.md`, `snake_case` | `booking_request`, `setting_up` |

Use the vocabulary table in `docs/TASKS.md` verbatim for statuses, outcomes and kinds. Never invent a synonym (`completed` for `confirmed`, `chat` for `web`).

## TypeScript

- Strict mode, ESM (`"type": "module"`) in every package; no `any` outside interop.
- `interface` for object shapes; `type` for unions, intersections, mapped types.
- String-literal unions derived from Zod (`z.enum([...])` + `z.infer`) for vocabulary values, not TypeScript `enum`.
- Types that cross a package boundary live in `@goodo/shared`: request and response shapes, job data, vocabulary. A package's own interface, such as `@goodo/jobs`' `Job`, is exported with the functions that use it.
- Every value from outside the process is parsed with Zod at the edge: request bodies, webhooks, env vars, model tool arguments, crawled JSON.

## React (dashboard and site)

- **Components are const arrow functions, never `function` declarations**: `export const Card = ({ title }: CardProps) => { … };`. Next.js pages too: `const HomePage = () => { … };` then `export default HomePage;`. Generics stay as written: `export const SegmentedControl = <Value extends string>(…) => …`. The founder's standing rule; ESLint (`no-restricted-syntax` in `packages/config/eslint.js`) fails a capitalised function declaration. Plain helpers and hooks may be either.
- One folder per component, entry `index.tsx`, named export for components, default export only where the framework requires it (Next.js pages, route modules).
- Props `interface` at the top; accept `className` and merge with `cn()` from `@/lib/utils`.
- Tailwind with design-token classes (`bg-background`, `text-foreground`, `bg-primary`, `text-muted-foreground`, `border-border`); no raw hex values in class names.

## Services (Fastify)

- **A factory that returns an object of functions defines each one first**, as a named `const` typed from its interface, then returns them by name: `return { signUp, signIn, revoke };`. Never write the function bodies inside the returned object literal. Reference: `apps/api/src/services/cognito.ts`.
- One route module per resource, registered as a plugin; request and response schemas from `@goodo/shared`.
- No business logic in handlers beyond parsing and calling a service or the engine.
- Every tenant query runs inside `withAccount(accountId, …)` from `@goodo/db`.
- Logs go through the request logger (Pino) with request id and account id; never log phone numbers, emails or message text.

## Error handling

- `try/catch` around every awaited external call (model, Twilio, ElevenLabs, Stripe, SES); log with context and return a defined fallback or a typed error.
- Fire-and-forget work (SMS, emails) uses `promise.catch(err => log.error(…))` or a pg-boss job; never an unhandled promise.
- User-facing errors say what happened and what to do next, in plain English.

## Barrels

- Workspace packages expose one `src/index.ts` with `export * from './…'`.
- Inside apps, no barrels: import from the file.

## Checklist

- [ ] Blank line before `if`, `switch`, `return` except the listed cases; no double blank lines
- [ ] Guard clauses instead of `else`; no nested ternaries; no `await` inside an `if` condition (checks print nothing)
- [ ] React components declared as `const Name = (…) => …`, not `function`
- [ ] Every function and component declared above what calls it
- [ ] Imports in the five groups, in order
- [ ] No import that breaks the boundary table
- [ ] Vocabulary values exactly as in `docs/TASKS.md`
- [ ] External input parsed with Zod; tenant queries inside `withAccount`
- [ ] No personal data in logs
