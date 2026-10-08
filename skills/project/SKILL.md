---
name: project
description: Monorepo structure, tech stack, package boundaries, commands and scaffolding map for a pnpm + Turborepo TypeScript repo (apps/web, apps/site, apps/api, apps/worker, packages/*). Use when creating files, choosing where code lives, adding a package or app, or understanding how the apps and packages relate.
---

# Project

pnpm monorepo with Turborepo. Deployable apps in `apps/`, shared code in `packages/`, package scope `@repo/`. Product decisions live in [[project-context]].

## Tech stack

| Layer | Tool |
|---|---|
| Language | TypeScript (strict, ESM) |
| API | Fastify 5, `fastify-type-provider-zod`, Pino |
| Jobs | pg-boss on the same Postgres |
| Database | Postgres, Prisma, row-level security |
| Dashboard | React 19, Vite, React Router, Tailwind, shadcn/ui, react-hook-form + Zod |
| Marketing site | Next.js, Tailwind (optional) |
| Auth | The auth provider; HTTP-only cookie set by the API |
| Tests | Vitest, Testing Library, Playwright |

## Layout

```
<repo>/
├─ apps/
│  ├─ web/         @repo/web      React dashboard
│  ├─ site/        @repo/site     Next.js marketing site (optional)
│  ├─ api/         @repo/api      Fastify API
│  └─ worker/      @repo/worker   pg-boss background jobs
├─ packages/
│  ├─ shared/      @repo/shared   Zod schemas, vocabulary enums, API_ROUTE, VALIDATION_MESSAGE, typed client, parseEnv
│  ├─ db/          @repo/db       Prisma schema, migrations, withAccount(), RLS helpers
│  ├─ ui/          @repo/ui       shared React components
│  └─ config/      @repo/config   tsconfig, ESLint, Prettier presets
├─ .claude/skills/ Claude Code skills
├─ docker-compose.yml
├─ pnpm-workspace.yaml, turbo.json, tsconfig.base.json
└─ CLAUDE.md
```

## Package relationships

```
@repo/shared  ← everything (browser-safe, zod only)
@repo/db      ← api, worker
frontends     → talk to the API over HTTP only
```

- Frontends (`web`, `site`) never import `@repo/db`. Business logic never lives in `apps/site` (no route handlers, no Server Actions).
- Apps never import each other; `api` and `worker` share the database, not code paths.
- Never hard-code API paths in a frontend: use the typed client and `API_ROUTE` from `@repo/shared`.

## Where things go

| Need | Where | Skill |
|---|---|---|
| Dashboard page | `apps/web/src/pages/<Name>/index.tsx` + `src/routes.tsx` | [[creating-pages]] |
| Dashboard component | `apps/web/src/components/<Name>/index.tsx` | [[adding-components]] |
| Context provider | `apps/web/src/context/` | [[adding-context-providers]] |
| Hook | `apps/web/src/hooks/use<Name>.ts` + `__tests__/` | [[creating-hooks]] |
| Form | schema in `@repo/shared`, form in the page | [[creating-forms]], [[shared-validation]] |
| API route | `apps/api/src/routes/<resource>.ts` | [[creating-api-routes]] |
| Service logic | `apps/<app>/src/services/<name>.ts` | [[creating-services]] |
| Background job | `apps/worker/src/jobs/<job-name>.ts` | — |
| Table or migration | `packages/db/prisma/` | [[prisma-and-migrations]] |
| Shared schema or vocabulary | `packages/shared/src/schemas/` | [[shared-validation]] |
| Helper | `src/lib/` of the app, or the package that owns it | [[utilities]] |
| Test | `__tests__/` next to the code | [[testing]] |

## Commands

```bash
pnpm db:up                          # start local Postgres (Docker)
pnpm dev                            # all apps via Turborepo
pnpm --filter @repo/api dev         # one app
pnpm test                           # Vitest in every package
pnpm --filter @repo/api test        # one package
pnpm e2e                            # Playwright
pnpm lint && pnpm typecheck
```

## Environment

- Each app validates its env with a Zod schema at start-up (`parseEnv` from `@repo/shared`) and exits if a value is missing.
- `.env.example` in each app lists every variable; real values are never committed.

## Checklist

- [ ] New code sits in the folder the table above names
- [ ] No import across the boundaries above
- [ ] New env var added to the app's Zod env schema and `.env.example`
- [ ] New package named `@repo/<name>` and covered by the `pnpm-workspace.yaml` globs
