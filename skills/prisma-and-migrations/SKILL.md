---
name: prisma-and-migrations
description: Add or change tables in packages/db (Prisma schema, migrations, row-level security, withAccount tenancy). Use whenever editing packages/db/prisma/schema.prisma, writing a migration, adding a tenant table, or querying the database from a service.
---

# Prisma and migrations

`packages/db` (`@repo/db`) holds the schema, the migrations and the only way services reach the database. Prisma 7 with the `prisma-client` generator and the `@prisma/adapter-pg` driver. Tenancy rules: every tenant row has `account_id`, every tenant query runs inside `withAccount()`, and row-level security is the net.

## Files

| File | What it is |
|---|---|
| `packages/db/prisma/schema.prisma` | Models and enums |
| `packages/db/prisma/migrations/<timestamp>_<name>/migration.sql` | One folder per migration; Prisma's SQL plus our row-level security |
| `packages/db/prisma.config.ts` | CLI config; reads `packages/db/.env` (copy `.env.example`) |
| `packages/db/src/index.ts` | `createDb()`, `withAccount()`, exported model types and enums |
| `packages/db/src/account-db.ts` | `AccountDb`, the transaction client without raw SQL |
| `packages/db/src/generated/prisma/` | Prisma's client. Git-ignored; Turborepo's `generate` task writes it before build, dev, lint, typecheck and test |
| `packages/db/src/__tests__/tenant.test.ts` | Tenant isolation suite |

## Querying from a service

```ts
import { createDb } from '@repo/db';

// Once at start-up, with DATABASE_URL from the service's env schema.
const db = createDb({ url: env.DATABASE_URL });

const projects = await db.withAccount(accountId, tx => tx.project.findMany());
```

- `createDb()` returns `withAccount`, the named functions below and `disconnect`, and nothing else. There is no client for queries outside an account, so a tenant query cannot skip `withAccount()` by accident.
- `withAccount(accountId, run)` opens a transaction, sets `app.account_id` for it with `set_config(…, true)`, and passes `run` a transaction client (`AccountDb`). Everything in `run` sees that account's rows only. Keep `run` short: it holds a connection.
- Every connection runs as the `app_service` role (`-c role=app_service`), which is not a superuser and does not own the tables, so every policy applies to it. The Docker superuser switches to it locally; each deployed environment grants it to the service's login user.
- Anything that is not one account's data gets its own named function on `Db`, never a general client:
  - `findAccountForUser(authSubject)`: the account id, or null. It calls the SQL function `app_account_for_user()`, which is `SECURITY DEFINER` (it runs as the table owner, so row-level security doesn't hide the row) with a fixed `search_path`, and returns only the one account id. Only `app_service` may execute it.
  - `createAccountForUser({ authSubject, email })`: inserts a new account with its user. If Postgres refuses a duplicate `auth_subject`, it returns the existing account, so calling it twice, or at the same moment, still makes one.
  - `recordSignUp(ip, limit)`: the per-IP sign-up limit. It calls `app_record_sign_up()`, a `SECURITY DEFINER` function that counts the IP's recent rows in `sign_up_attempts` and records the attempt in one step, under an advisory lock per IP so attempts arriving at once can't both take the last place. It returns true when the attempt is allowed. `app_service` has no grant on the table itself. A daily worker job deletes old rows through `purgeSignUpAttempts()`.
  - Admin features add theirs the same way. A `SECURITY DEFINER` function returns the narrowest answer it can, and gets a test that the tables stay hidden.
- **No raw SQL inside `withAccount()`.** The client it passes has no `$queryRaw` or `$executeRaw` (left out of `AccountDb`, and blocked at runtime if cast back), because raw SQL could change `app.account_id` partway through and read another account. A query that needs SQL (full-text search, say) becomes a named function in `packages/db` that takes its values as parameters and uses tagged templates, never a built string.

## Conventions

- **Tables arrive with their feature.** Add a table in the task that first needs it, not ahead of time.
- **`@map` everything.** camelCase fields with `@map("snake_case")`, and `@@map("snake_case_plural")` on every model.
- **Keys:** `id String @id @default(uuid()) @db.Uuid`; foreign keys `@db.Uuid` too.
- **Timestamps:** `createdAt DateTime @default(now()) @map("created_at")` on every table; `updatedAt DateTime @updatedAt @map("updated_at")` on rows that change.
- **Enums are Postgres enums** named after the vocabulary table in your docs (`@@map("task_status")`), with the values written exactly as there (`todo`, `in_progress`, `done`). The Zod enums in `@repo/shared` (`schemas/vocabulary.ts`) must list the same values.
- **`onDelete` is deliberate:** `Cascade` for rows that belong to their parent (users to an account, tasks to a project), and a comment saying why.
- **`account_id` on every tenant table**, indexed, `onDelete: Cascade` to `accounts` unless there is a reason not to.

## Adding a tenant table

1. Add the model to `schema.prisma` with `accountId` and `@@index([accountId])`.
   A table that belongs to another tenant row keys on the account as well: the parent (`Project`) gets `@@unique([id, accountId])` and the child's relation uses `fields: [projectId, accountId], references: [id, accountId]`. Then a task can only ever belong to a project of its own account, whatever row-level security allows; `tenant.test.ts` checks it ("cannot attach a task to B's project").
   ```prisma
   model Task {
     id        String     @id @default(uuid()) @db.Uuid
     accountId String     @map("account_id") @db.Uuid
     projectId String     @map("project_id") @db.Uuid
     title     String
     status    TaskStatus @default(todo)
     createdAt DateTime   @default(now()) @map("created_at")
     updatedAt DateTime   @updatedAt @map("updated_at")

     account Account @relation(fields: [accountId], references: [id], onDelete: Cascade)
     project Project @relation(fields: [projectId, accountId], references: [id, accountId], onDelete: Cascade)

     @@index([accountId])
     @@index([projectId])
     @@map("tasks")
   }
   ```
2. Create the migration without applying it:
   ```bash
   pnpm --filter @repo/db migrate --create-only --name add_<table>
   ```
3. Append row-level security to the generated `migration.sql`:
   ```sql
   GRANT SELECT, INSERT, UPDATE, DELETE ON "<table>" TO app_service;

   ALTER TABLE "<table>" ENABLE ROW LEVEL SECURITY;
   CREATE POLICY account_isolation ON "<table>"
     USING ("account_id" = app_account_id())
     WITH CHECK ("account_id" = app_account_id());
   ```
4. Apply it: `pnpm --filter @repo/db migrate`. The next `pnpm dev`, `pnpm test` or `pnpm typecheck` regenerates the client.
5. Add the table to `tenant.test.ts` in the same change: A cannot read, change, delete or add B's rows, and a query with no account sees nothing. The suite also fails on any table without row-level security and a policy.
6. Export the model type from `src/index.ts` if a service needs it.

A table that is not tenant data (for example `leads` from the public site) still needs row-level security enabled, with a policy written for how it is used, or the schema test fails. Say why in the migration. A table reached only through a `SECURITY DEFINER` function gets no grant for `app_service` and a policy that refuses every row (`sign_up_attempts`: `USING (false) WITH CHECK (false)`), with a test that the role can't read or write it.

## Migrations

```bash
pnpm db:up                                              # local Postgres (Docker)
pnpm --filter @repo/db migrate                          # create and apply migrations (does not regenerate the client in Prisma 7)
pnpm --filter @repo/db migrate --create-only --name <verb>_<scope>   # write one to edit before applying
pnpm --filter @repo/db generate                         # regenerate the client (Turborepo also runs this before build, dev, lint, typecheck, test)
pnpm db:seed                                            # generate, then upsert the sample accounts (prisma/seed.ts)
pnpm --filter @repo/db migrate:deploy                   # apply without prompting (CI, staging, production)
```

- Names are snake_case verb and scope: `add_accounts_and_users`, `add_projects`.
- Hand-edit SQL for row-level security, extensions, partial indexes and backfills. Edit before the migration is committed.
- Prisma ignores partial indexes, but drops any other index it doesn't know about in its next migration: declare it in the schema, or check the generated SQL.
- `migrate dev` stops when Prisma has a warning to confirm and the terminal isn't interactive (Claude Code). Then write the SQL with `pnpm exec prisma migrate diff --from-config-datasource --to-schema prisma/schema.prisma --script` into a new `migrations/<timestamp>_<name>/migration.sql`, add the hand-written parts, and apply it with `migrate:deploy`.
- Once a migration is merged to `main`, never edit it: write a new one.
- Prisma 7 runs neither `generate` nor the seed after `migrate dev` or `migrate reset`. Run `pnpm db:seed` after a reset (it generates first).
- CI starts Postgres, runs `migrate:deploy`, then the tests.

## Checklist

- [ ] Table added in the task that needs it
- [ ] Every field `@map`ped, every model `@@map`ped
- [ ] Tenant table: `account_id`, index, grant, row-level security and policy in the migration
- [ ] Child of a tenant row keys on `[parentId, accountId]`
- [ ] Table added to `tenant.test.ts` in the same change
- [ ] Enum values match the vocabulary table and `@repo/shared`
- [ ] `onDelete` chosen on purpose
- [ ] Queries go through `withAccount()` or a named function in `packages/db`
- [ ] No merged migration edited

Related: [[project]], [[testing]], [[creating-services]].
