---
name: project
description: Goodo monorepo structure, tech stack, package boundaries, commands and scaffolding map. Use when creating files, choosing where code lives, adding a package or app, or understanding how the apps and packages relate.
---

# Project

_Adapted 2026-09-23 from rn-boilerplate `project`, replacing Debrief's `project-structure`. Written before the code exists: the layout is the plan in `docs/PRD.md` section 7 and `docs/TASKS.md`. Run [[refine-skill]] once task 1.0 lands._

pnpm monorepo with Turborepo. Eight deployable apps, three shared packages, one vertical pack. Product decisions live in [[project-context]].

## Tech stack

| Layer | Tool |
|---|---|
| Language | TypeScript (strict, ESM), Node 24 |
| Services | Fastify 5, Zod type provider, Pino |
| Jobs | pg-boss on the same Postgres |
| Database | Postgres 16 + pgvector, Prisma, row-level security |
| Dashboard | React 19, Vite, React Router, Tailwind, shadcn/ui, react-hook-form + Zod |
| Marketing site | Next.js, static export, Tailwind |
| Widget | Preact in a Shadow DOM, Server-Sent Events, Vite library build |
| Model | Claude via the Anthropic SDK; Bedrock (Australia) per assistant |
| Embeddings | Amazon Titan Text Embeddings V2 on Bedrock Sydney, 1024 dimensions |
| Crawler | Crawlee `PlaywrightCrawler`: headless Chromium for every page |
| Voice | ElevenLabs Agents with native Twilio import; our `/brain` as the custom LLM |
| SMS and numbers | Twilio |
| Auth | Cognito, HTTP-only cookie set by the API |
| Email | SES |
| Billing | Stripe |
| Tests | Vitest, Testing Library, Playwright; `evals/` for model behaviour |
| Infra | Terraform, AWS `ap-southeast-2`, ECS Fargate; site on Vercel |

## Layout

```
Goodo/
├─ apps/
│  ├─ site/        @goodo/site     Next.js marketing site → Vercel
│  ├─ web/         @goodo/web      React dashboard → S3 + CloudFront
│  ├─ widget/      @goodo/widget   Preact embed → S3 + CloudFront (versioned)
│  ├─ api/         @goodo/api      Fastify: dashboard API, signup, Stripe, leads
│  ├─ chat/        @goodo/chat     Fastify: widget chat (SSE), widget settings, handoff email (from task 6.6)
│  ├─ voice/       @goodo/voice    Fastify: /brain, ElevenLabs + Twilio webhooks, SMS
│  ├─ worker/      @goodo/worker   pg-boss jobs: embed, recrawl, profile draft, reminders
│  └─ crawler/     @goodo/crawler  pg-boss crawl jobs in headless Chromium, always on
├─ packages/
│  ├─ engine/      @goodo/engine   prompt assembly, retrieval, tools, handoff, hours (no HTTP)
│  ├─ db/          @goodo/db       Prisma schema, migrations, withAccount(), RLS helpers
│  ├─ jobs/        @goodo/jobs     pg-boss job runner (createWorker, Job) and the services' logger
│  ├─ shared/      @goodo/shared   Zod schemas, vocabulary enums, typed API client
│  ├─ ui/          @goodo/ui       shared React components (only once site and web both need one)
│  └─ config/      @goodo/config   tsconfig, ESLint, Prettier presets
├─ packs/
│  └─ restaurant/  @goodo/packs-restaurant  profile schema, rules, Marlow's demo profile
├─ evals/          chat/ and voice/ scenario files
├─ infra/          bootstrap/, modules/, envs/dev, envs/staging, envs/prod, envs/site (Vercel)
├─ scripts/        seed, crawl-one-site, import-number, create-agent
├─ docs/           PRD.md, TASKS.md, spikes/, runbook.md
├─ .claude/skills/ Claude Code skills
├─ docker-compose.yml
├─ pnpm-workspace.yaml, turbo.json, tsconfig.base.json
└─ CLAUDE.md
```

## Package relationships

```
@goodo/shared  ← everything (browser-safe, zod only)
@goodo/db      ← api, chat, voice, worker, crawler
@goodo/engine  ← api, chat, voice, worker   (data access via interfaces, never imports Prisma directly)
packs/*        ← api, chat, voice, worker
@goodo/jobs    ← worker, crawler   (runs pg-boss jobs; other services only send them)
frontends      → talk to the API over HTTP only
```

- `api`, `chat`, `voice`, `worker` and `crawler` are separate services (chat split from api 2026-09-26, PRD section 7). They never call each other; they share the database and the engine library.
- Frontends never import `engine` or `db`. Business logic never lives in `apps/site` (no route handlers, no Server Actions).
- Never hard-code API paths in a frontend: use the typed client from `@goodo/shared`.

## Where things go

| Need | Where | Skill |
|---|---|---|
| Dashboard page | `apps/web/src/pages/<Name>/index.tsx` | `creating-pages` |
| Dashboard component | `apps/web/src/components/<Name>/index.tsx` | `adding-components` |
| Form | schema in `@goodo/shared`, form in the page | `creating-forms`, `shared-validation` |
| Hook | `apps/web/src/hooks/use<Name>.ts` + `__tests__/` | `creating-hooks` |
| API route | `apps/api/src/routes/<resource>.ts` | `creating-api-routes` |
| Widget chat route | `apps/chat/src/routes/<resource>.ts` | `creating-api-routes` |
| Service logic | `apps/<service>/src/services/<name>.ts` | `creating-services` |
| Model, prompt or tool logic | `packages/engine/src/` | [[project-context]] |
| Background job | `apps/worker/src/jobs/<job-name>.ts` | — |
| Crawl job or crawler code | `apps/crawler/src/jobs/`, `apps/crawler/src/crawl/` | — |
| Phone endpoint or tool | `apps/voice/src/routes/`, `apps/voice/src/tools/` | — |
| Table or migration | `packages/db/prisma/` | `prisma-and-migrations` |
| Shared schema or vocabulary | `packages/shared/src/schemas/` | `shared-validation` |
| Helper | `src/lib/` of the app, or the package that owns it | `utilities` |
| Test | `__tests__/` next to the code | [[testing]] |
| Model behaviour check | `evals/chat/` or `evals/voice/` | [[testing]] |

## Commands

```bash
pnpm db:up                          # start local Postgres (Docker)
pnpm dev                            # all apps via Turborepo
pnpm --filter @goodo/api dev        # one app
pnpm test                           # Vitest in every package
pnpm --filter @goodo/api test       # one package
pnpm e2e                            # Playwright (site, dashboard)
pnpm evals                          # model evals — real model, costs money, before releases
pnpm lint && pnpm typecheck
```

## Environment

- Each app validates its env with a Zod schema at start-up (`packages/shared` `env.ts` helper) and exits if a value is missing.
- `.env.example` in each app lists every variable; real values never committed.
- Local voice development needs a Cloudflare tunnel so ElevenLabs and Twilio reach `apps/voice`.

## Checklist

- [ ] New code sits in the folder the table above names
- [ ] No import across the boundaries above
- [ ] New env var added to the app's Zod env schema and `.env.example`
- [ ] New package named `@goodo/<name>` and added to `pnpm-workspace.yaml` globs
