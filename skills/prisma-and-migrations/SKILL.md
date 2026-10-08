---
name: prisma-and-migrations
description: Add or change tables in packages/db (Prisma schema, migrations, row-level security, withAccount, withAssistant). Use whenever editing packages/db/prisma/schema.prisma, writing a migration, adding a tenant table, or querying the database from a service.
---

# Prisma and migrations

_Rewritten 2026-09-28 for Goodo (it described debrief-demo before). Source of truth: `packages/db`._

`packages/db` (`@goodo/db`) holds the schema, the migrations and the only way services reach the database. Prisma 7 with the `prisma-client` generator and the `@prisma/adapter-pg` driver, as in debrief-demo. Tenancy rules come from [[project-context]]: every tenant row has `account_id`, every tenant query runs inside `withAccount()`, and row-level security is the net.

## Files

| File | What it is |
|---|---|
| `packages/db/prisma/schema.prisma` | Models and enums |
| `packages/db/prisma/migrations/<timestamp>_<name>/migration.sql` | One folder per migration; Prisma's SQL plus our row-level security |
| `packages/db/prisma.config.ts` | CLI config; reads `packages/db/.env` (copy `.env.example`) |
| `packages/db/src/index.ts` | `createDb()`, `withAccount()`, `withAssistant()`, exported model types and enums |
| `packages/db/src/account-db.ts` | `AccountDb` (the transaction client without raw SQL), `AssistantScope` |
| `packages/db/src/crawls.ts` | `createCrawls()`: queue, start, update, finish and list an assistant's scans |
| `packages/db/src/generated/prisma/` | Prisma's client. Git-ignored; Turborepo's `generate` task writes it before build, dev, lint, typecheck and test |
| `packages/db/src/__tests__/tenant.test.ts` | Tenant isolation suite |
| `packages/db/src/__tests__/assistant-scope.test.ts` | Isolation between two assistants of one account |

## Querying from a service

```ts
import { createDb } from '@goodo/db';

// Once at start-up, with DATABASE_URL from the service's env schema.
const db = createDb({ url: env.DATABASE_URL });

const assistants = await db.withAccount(accountId, tx => tx.assistant.findMany());
```

- `createDb()` returns `withAccount`, `withAssistant`, the named functions below and `disconnect`, and nothing else. There is no client for queries outside an account, so a tenant query cannot skip `withAccount()` by accident (decided 2026-09-28, replacing the planned "Prisma extension that refuses queries").
- `withAccount(accountId, run)` opens a transaction, sets `app.account_id` for it with `set_config(…, true)`, and passes `run` a transaction client (`AccountDb`). Everything in `run` sees that account's rows only. Keep `run` short: it holds a connection.
- `withAssistant(accountId, assistantId, run)` sets `app.assistant_id` as well, and everything in `run` sees that one assistant's rows only. An agency's clients share one account, so use it whenever the code serves one assistant: the widget, a call, a crawl or embed job, a dashboard route for one assistant. Keep `withAccount()` for views across assistants (the assistants list, the agency's conversations). Named functions for one assistant take an `AssistantScope` (`{ accountId, assistantId }`), as `createCrawls()` does. Inside it, `accounts` and `users` are read-only, since deleting the account would cascade to every other client.
- Every connection runs as the `goodo_app` role (`-c role=goodo_app`), which is not a superuser and does not own the tables, so every policy applies to it. The Docker superuser switches to it locally; each deployed environment grants it to the service's login user (task 14.4).
- Anything that is not one account's data gets its own named function on `Db`, never a general client:
  - `findAccountForUser(cognitoSub)`: the account id, or null. It calls the SQL function `app_account_for_user()`, which is `SECURITY DEFINER` (it runs as the table owner, so row-level security doesn't hide the row) with a fixed `search_path`, and returns only the one account id. Only `goodo_app` may execute it.
  - `createAccountForUser({ cognitoSub, email })`: inserts a new `business` account on the pilot plan with its user. If Postgres refuses a duplicate `cognito_sub`, it returns the existing account, so calling it twice, or at the same moment, still makes one.
  - `recordSignUp(ip, limit)`: the per-IP sign-up limit (PRD 7, S1.10). It calls `app_record_sign_up()`, a `SECURITY DEFINER` function that counts the IP's recent rows in `sign_up_attempts` and records the attempt in one step, under an advisory lock per IP so attempts arriving at once can't both take the last place. It returns true when the attempt is allowed. `goodo_app` has no grant on the table itself. A daily worker job deletes rows older than a day through `purgeSignUpAttempts()` (`app_purge_sign_up_attempts()`, which refuses less than a day), so the window can't be longer. There is no per-domain limit (dropped 2026-09-29: none of Firebase, Auth0, Supabase or AWS WAF's account creation rules use one).
  - `savePageChunks({ accountId, assistantId }, { pageId, contentHash, chunks })` and `searchChunks(accountId, { assistantId, embedding, limit })`: the chunks' vectors, which Prisma can't read or write. They run in a scoped transaction like `withAssistant()`, through `inScope()`, which keeps raw SQL inside `src/index.ts`. `savePageChunks` locks the page and saves only for an `adding` or `added` page whose text hash still matches, then flips `adding` to `added` and sets `sources.first_added_at`; `searchChunks` returns only `added` pages' chunks. An embedding must have `EMBEDDING_DIMENSIONS` (1024) numbers, not all zero. Search is exact over the assistant's own chunks: there's no vector index, because an approximate index shared by every tenant can stop before it reaches one assistant's chunks.
  - The admin screens (S9, S11) will add their own the same way. A `SECURITY DEFINER` function returns the narrowest answer it can, and gets a test that the tables stay hidden.
- **No raw SQL inside `withAccount()`.** The client it passes has no `$queryRaw` or `$executeRaw` (left out of `AccountDb`, and blocked at runtime if cast back), because raw SQL could change `app.account_id` partway through and read another account. A query that needs SQL (pgvector search) becomes a named function in `packages/db` that takes its values as parameters and uses tagged templates, never a built string.

## Conventions

- **Tables arrive with their screen.** Add a table in the screen task that first needs it (docs/TASKS.md, "Screen by screen"), not ahead of time.
- **`@map` everything.** camelCase fields with `@map("snake_case")`, and `@@map("snake_case_plural")` on every model.
- **Keys:** `id String @id @default(uuid()) @db.Uuid`; foreign keys `@db.Uuid` too.
- **Timestamps:** `createdAt DateTime @default(now()) @map("created_at")` on every table; `updatedAt DateTime @updatedAt @map("updated_at")` on rows that change.
- **Enums are Postgres enums** named after the vocabulary table in `docs/TASKS.md` (`@@map("account_kind")`), with the values written exactly as there. The Zod enums in `@goodo/shared` (`schemas/vocabulary.ts`) must list the same values.
- **`onDelete` is deliberate:** `Cascade` for rows that belong to their parent (users to an account), and a comment saying why.
- **`account_id` on every tenant table**, indexed, `onDelete: Cascade` to `accounts` unless there is a reason not to.

## Adding a tenant table

1. Add the model to `schema.prisma` with `accountId String @map("account_id") @db.Uuid` and `@@index([accountId])`.
   A table that belongs to another tenant row (channels to an assistant) keys on the account as well: the parent gets `@@unique([id, accountId])` and the child's relation uses `fields: [parentId, accountId], references: [id, accountId]`. Then a child can only ever belong to a parent of its own account, whatever row-level security allows; `tenant.test.ts` checks it ("cannot attach a channel to B's assistant").
2. Create the migration without applying it:
   ```bash
   pnpm --filter @goodo/db migrate --create-only --name add_<table>
   ```
3. Append row-level security to the generated `migration.sql`:
   ```sql
   GRANT SELECT, INSERT, UPDATE, DELETE ON "<table>" TO goodo_app;

   ALTER TABLE "<table>" ENABLE ROW LEVEL SECURITY;
   CREATE POLICY account_isolation ON "<table>"
     USING ("account_id" = app_account_id())
     WITH CHECK ("account_id" = app_account_id());
   ```
   A table with an `assistant_id` also gets the assistant policy. It is restrictive, so Postgres ANDs it with the account one, and `withAccount()` (where `app_assistant_id()` is NULL) still sees every assistant:
   ```sql
   CREATE POLICY assistant_isolation ON "<table>" AS RESTRICTIVE
     USING (app_assistant_id() IS NULL OR "assistant_id" = app_assistant_id())
     WITH CHECK (app_assistant_id() IS NULL OR "assistant_id" = app_assistant_id());
   ```
4. Apply it: `pnpm --filter @goodo/db migrate`. The next `pnpm dev`, `pnpm test` or `pnpm typecheck` regenerates the client.
5. Add the table to `tenant.test.ts` in the same PR: A cannot read, change, delete or add B's rows, and a query with no account sees nothing. The suite also fails on any table without row-level security and a policy. A table with an `assistant_id` goes in `assistant-scope.test.ts` too, which also fails on any such table without the restrictive policy.
6. Export the model type from `src/index.ts` if a service needs it.

A table that is not tenant data (for example `leads` from the site) still needs row-level security enabled, with a policy written for how it is used, or the schema test fails. Say why in the migration. A table reached only through a `SECURITY DEFINER` function gets no grant for `goodo_app` and a policy that refuses every row (`sign_up_attempts`: `USING (false) WITH CHECK (false)`), with a test that the role can't read or write it.

## Migrations

```bash
pnpm db:up                                              # local Postgres (Docker)
pnpm --filter @goodo/db migrate                         # create and apply migrations (does not regenerate the client in Prisma 7)
pnpm --filter @goodo/db migrate --create-only --name <verb>_<scope>   # write one to edit before applying
pnpm --filter @goodo/db generate                        # regenerate the client (Turborepo also runs this before build, dev, lint, typecheck, test)
pnpm db:seed                                            # generate, then upsert the sample accounts (prisma/seed.ts)
pnpm --filter @goodo/db migrate:deploy                  # apply without prompting (CI, staging, production)
```

- Names are snake_case verb and scope: `add_accounts_and_users`, `add_assistants`.
- Hand-edit SQL for row-level security, pgvector, partial indexes and backfills. Edit before the migration is committed.
- A vector column is `Unsupported("vector(1024)")` in the schema. Prisma ignores partial indexes, but drops any other index it doesn't know about in its next migration: declare it in the schema, or check the generated SQL.
- `migrate dev` stops when Prisma has a warning to confirm and the terminal isn't interactive (Claude Code). Then write the SQL with `pnpm exec prisma migrate diff --from-config-datasource --to-schema prisma/schema.prisma --script` into a new `migrations/<timestamp>_<name>/migration.sql`, add the hand-written parts, and apply it with `migrate:deploy`.
- Once a migration is merged to `main`, never edit it: write a new one.
- Prisma 7 runs neither `generate` nor the seed after `migrate dev` or `migrate reset`. Run `pnpm db:seed` after a reset (it generates first).
- CI starts Postgres, runs `migrate:deploy`, then the tests.

## Checklist

- [ ] Table added in the screen task that needs it
- [ ] Every field `@map`ped, every model `@@map`ped
- [ ] Tenant table: `account_id`, index, grant, row-level security and policy in the migration
- [ ] Table added to `tenant.test.ts` in the same PR, and to `assistant-scope.test.ts` with the assistant policy if it has an `assistant_id`
- [ ] Enum values match the vocabulary table and `@goodo/shared`
- [ ] `onDelete` chosen on purpose
- [ ] Queries go through `withAccount()`, `withAssistant()` (for code serving one assistant) or a named function in `packages/db`
- [ ] No merged migration edited

Related: [[project]], [[testing]], [[creating-services]].
