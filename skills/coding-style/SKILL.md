---
name: coding-style
description: Enforces formatting, whitespace, control flow, import order, naming and TypeScript conventions across the monorepo (Fastify API and worker, React dashboard in apps/web, Next.js site, packages/*). Always active when writing or editing code.
---

# Coding Style

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
const summarise = (tasks: Task[]): string | null => {
  if (tasks.length === 0) {
    return null;
  }

  const text = tasks.map(task => task.title).join(', ');

  if (text.length < 20) {
    return text;
  }

  return `${text.slice(0, 200)}…`;
};
```

## Braces

- **Every** `if`, `else if`, `else`, `for` and `while` body takes braces, even a single statement, guards included: `if (!project) { return null; }` written over three lines, never `if (!project) return null;`. ESLint's `curly: 'all'` enforces it, and `eslint --fix` adds them.
- `{` on the same line as the keyword; `else` on the same line as the closing `}`.

## Declaration order

A function or component sits above everything that calls it, so a file reads top to bottom: helpers and child components first, the exported component or factory last. Inside a factory, the functions it defines follow the same order. ESLint enforces it (`@typescript-eslint/no-use-before-define` in `packages/config/eslint.js`); types are exempt.

Two values that need each other, such as a server and the job queue whose error handler logs through it, get a one-line `// eslint-disable-next-line @typescript-eslint/no-use-before-define -- <why>`. They are the only exception.

## Comments

**No comments by default**: clean code explains itself. Before writing one, make the code say it: a clearer name, a named constant, a small function, a guard clause. A comment is allowed only when that's impossible: a workaround for someone else's bug, or an outside limit or trap a reader would otherwise get wrong. One line, never two or more. When in doubt, leave it out.

- Never restate a name, a type or what the next line plainly does.
- Constants, types, interface fields, options, env schema entries and function summaries get no comment: a field named `timeoutSeconds` explains itself. A field comment only for a trap, such as a limit that is best effort, and it starts `Trap:`.
- No dates, decisions or task numbers in code. Those go in commits and PRs.

```ts
// Yes: a trap the code can't show
// createQueue leaves an existing queue's settings alone, and a stored schedule outlives the code that made it.

// No: says what the name already says
// Stops taking jobs and waits for the running ones to finish.
const stop = async () => { … };
```

## Control flow

Bail out with a **guard clause**. Bad input, nothing to do, or a state with no answer returns on the spot, so the real body stays at one indent level and is never wrapped in `else`.

**Never `await` inside an `if` condition.** Await into a named `const`, then test the name, so the condition reads as a plain value (ESLint's `no-restricted-syntax` enforces it). An `await` in the `if`'s body is fine.

```ts
// Yes
const removed = await tasks.remove(accountId, taskId);

if (!removed) {
  return reply.status(404).send({ error: MESSAGE.notFound });
}

// No
if (!(await tasks.remove(accountId, taskId))) {
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
const statusLabel = (status: TaskStatus): string => {
  switch (status) {
    case 'todo':
      return 'To do';
    case 'in_progress':
      return 'In progress';
    default:
      return 'Done';
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
2. **Workspace packages**: `@repo/shared`, `@repo/db`, `@repo/ui`
3. **`@/` alias** inside the current app
4. **Relative** siblings in the same module folder
5. **Assets and CSS**

`import type` follows the group of its source.

```ts
import { z } from 'zod';

import { taskStatusSchema } from '@repo/shared';

import { cn } from '@/lib/utils';

import { StatusBadge } from './StatusBadge';

import './styles.css';
```

## Boundaries

Enforced by a lint rule. Breaking one is a lint error, not a style note.

| Code | May import |
|---|---|
| `apps/site`, `apps/web` | `@repo/shared`, `@repo/ui` |
| `apps/api`, `apps/worker` | `@repo/db`, `@repo/shared` |
| `packages/shared` | `zod` only (it ships to browsers) |
| any app | never another app |

## Naming

| Kind | Convention | Example |
|---|---|---|
| React component | `PascalCase/` folder with `index.tsx` | `components/ProjectCard/index.tsx` |
| Hook | `camelCase`, `use` prefix | `hooks/useProjects.ts` |
| Fastify route module | `kebab-case.ts` | `routes/project-members.ts` |
| Worker job | `kebab-case.ts`, named after the job | `jobs/send-digest.ts` |
| Library file | `kebab-case.ts` | `lib/format-cents.ts` |
| Zod schema file | `camelCase.ts` | `schemas/createTask.ts` |
| Test file | `<name>.test.ts(x)` in `__tests__/` next to the code | `lib/__tests__/format-cents.test.ts` |
| Constant value | `UPPER_SNAKE_CASE` | `MAX_TASKS_PER_PROJECT` |
| Database enum value | the project vocabulary, `snake_case` | `in_progress`, `done` |

Use the vocabulary table in your docs verbatim for statuses, kinds and roles. Never invent a synonym (`completed` for `done`, `active` for `in_progress`).

## TypeScript

- Strict mode, ESM (`"type": "module"`) in every package; no `any` outside interop.
- `interface` for object shapes; `type` for unions, intersections, mapped types.
- String-literal unions derived from Zod (`z.enum([...])` + `z.infer`) for vocabulary values, not TypeScript `enum`.
- Types that cross a package boundary live in `@repo/shared`: request and response shapes, job data, vocabulary. A package's own interface is exported with the functions that use it.
- Every value from outside the process is parsed with Zod at the edge: request bodies, webhooks, env vars, third-party responses.

## React (dashboard and site)

- **Components are const arrow functions, never `function` declarations**: `export const ProjectCard = ({ title }: ProjectCardProps) => { … };`. Next.js pages too: `const HomePage = () => { … };` then `export default HomePage;`. Generics stay as written: `export const SegmentedControl = <Value extends string>(…) => …`. ESLint (`no-restricted-syntax` in `packages/config/eslint.js`) fails a capitalised function declaration. Plain helpers and hooks may be either.
- One folder per component, entry `index.tsx`, named export for components, default export only where the framework requires it (Next.js pages).
- Props `interface` at the top; accept `className` and merge with `cn()` from `@/lib/utils`.
- Tailwind with design-token classes (`bg-background`, `text-foreground`, `bg-primary`, `text-muted-foreground`, `border-border`); no raw hex values in class names.

## Services (Fastify)

- **A factory that returns an object of functions defines each one first**, as a named `const` typed from its interface, then returns them by name: `return { create, list, remove };`. Never write the function bodies inside the returned object literal.
- One route module per resource, registered as a plugin; request and response schemas from `@repo/shared`.
- No business logic in handlers beyond parsing and calling a service.
- Every tenant query runs inside `withAccount(accountId, …)` from `@repo/db`.
- Logs go through the request logger (Pino) with request id and account id; never log emails, phone numbers or text users typed.

## Error handling

- `try/catch` around every awaited external call (third-party APIs, email, payments); log with context and return a defined fallback or a typed error.
- Fire-and-forget work (emails, notifications) uses `promise.catch(err => log.error(…))` or a pg-boss job; never an unhandled promise.
- User-facing errors say what happened and what to do next, in plain English.

## Barrels

- Workspace packages expose one `src/index.ts` with `export * from './…'`.
- Inside apps, no barrels: import from the file.

## Checklist

- [ ] Blank line before `if`, `switch`, `return` except the listed cases; no double blank lines
- [ ] Guard clauses instead of `else`; no nested ternaries; no `await` inside an `if` condition (checks print nothing)
- [ ] React components declared as `const Name = (…) => …`, not `function`
- [ ] Every function and component declared above what calls it
- [ ] Comments only for a non-obvious why, one line each
- [ ] Imports in the five groups, in order
- [ ] No import that breaks the boundary table
- [ ] Vocabulary values exactly as in the project's vocabulary table
- [ ] External input parsed with Zod; tenant queries inside `withAccount`
- [ ] No personal data in logs
