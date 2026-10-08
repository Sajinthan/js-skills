---
name: testing
description: Writes and runs tests across the monorepo — Vitest unit and integration tests for apps/api, apps/worker, packages and React components in apps/web, Fastify inject route tests, row-level security isolation tests, and Playwright end-to-end tests. Use when adding tests, fixing test failures, mocking providers, or configuring Vitest or Playwright.
---

# Testing

## Suites

| Command | Tool | Scope | When |
|---|---|---|---|
| `pnpm test` | Vitest | every package: logic, routes, jobs, components | every PR (CI) |
| `pnpm --filter @repo/<pkg> test` | Vitest | one package | while working |
| `pnpm e2e` | Playwright | site pages, dashboard flows | every PR |

## Where tests live

Co-located `__tests__/` next to the code it tests:

```
apps/api/src/routes/__tests__/projects.test.ts
apps/worker/src/jobs/__tests__/send-digest.test.ts
packages/shared/src/__tests__/format-cents.test.ts
packages/db/src/__tests__/tenant.test.ts
apps/web/src/pages/Projects/__tests__/Projects.test.tsx
```

Name: `<module>.test.ts(x)`. Playwright specs live in `apps/<app>/e2e/*.spec.ts`.

## Vitest config

- Node environment for services and packages; `jsdom` for `apps/web` and `apps/site` component tests.
- `globals: false`: import `describe`, `it`, `expect`, `vi` from `vitest`.
- `@/` alias mirrors the app's `tsconfig`.
- Env stubs in the config's `test.env`, never a real `.env`.

## Route tests: Fastify `inject`

Build the app, inject requests, assert on status and body. They run against the Docker Postgres (`pnpm db:up`, `pnpm db:migrate`; CI starts its own). `buildTestApp()` (`apps/api/src/test/buildTestApp.ts`):

- connects to it through `createDb`
- gives each fake auth-provider user its own random `sub` (`testIdentity()`), so test files running at the same time never share an account
- deletes every account the app created when the app closes
- leaves rate limits out unless built with `rateLimits: true`, since every `inject` comes from 127.0.0.1. A test that turns them on sends each request from a random `remoteAddress`

To sign in, call `POST sign-in` (the fake auth provider accepts any password) and send back the cookie. To test a session with no account, seal one with `sealSession(…, TEST_SESSION_KEY)` for a `testIdentity()`.

```ts
import { afterAll, beforeAll, describe, expect, it } from 'vitest';

import { buildTestApp } from '@/test/buildTestApp';

describe('POST /projects', () => {
  // Pass `{ auth: fakeAuth({ … }, identity) }` to override one call or fix the user.
  const app = buildTestApp();

  beforeAll(() => app.ready());

  afterAll(() => app.close());

  it('rejects an empty name', async () => {
    const response = await app.inject({ method: 'POST', url: '/projects', payload: { name: '' } });

    expect(response.statusCode).toBe(400);
  });
});
```

## Mocking providers

Never call a third-party API (model, email, payments, auth provider) from `pnpm test`. Mock the client module with `vi.hoisted` before importing the code under test (ESM order matters):

```ts
const { mockSend } = vi.hoisted(() => ({ mockSend: vi.fn() }));

vi.mock('@/services/mailer', () => ({
  MailerClient: class {
    send = mockSend;
  },
}));

import { sendInviteEmail } from '../invite-email';
```

Reset in `beforeEach` (`mockSend.mockReset()`).

**Webhooks** from third parties are tested with recorded payloads saved under `__tests__/fixtures/`, including the signature header, so signature checking is exercised too.

**Streaming** endpoints (SSE): mock the upstream client to yield a fixed sequence; assert on the frames in order.

## Tenant isolation tests

`packages/db/src/__tests__/tenant.test.ts` seeds two accounts and, for every tenant table, asserts that queries inside `withAccount(A)` never return B's rows. **Any new tenant table is added to this suite in the same change.** A failure here blocks merge. See [[prisma-and-migrations]].

## Data-driven tests

Scenario arrays with `it.each` for rules with many cases (formatting, status transitions, permissions):

```ts
it.each([
  { cents: 0, expected: '$0.00' },
  { cents: 1999, expected: '$19.99' },
  { cents: -500, expected: '-$5.00' },
])('formatCents($cents) is $expected', ({ cents, expected }) => {
  expect(formatCents(cents)).toBe(expected);
});
```

Test data builders live in `src/lib/test-helpers.ts` of the package that owns the type.

## Conventions

- One `describe` per function or route; one `it` per behaviour; happy path and edge cases.
- `async`/`await` in test bodies, not `.then()` chains.
- Assertions: `toBe`, `toEqual`, `toContain`, `toHaveBeenCalledWith`.
- Dashboard tests render through `renderApp` (`apps/web/src/test/renderApp.tsx`), which wraps the providers and router around a seeded account; pass `account: null` to load it from API stubs instead. Seed data lives in `apps/web/src/test/`.
- If the project calls an LLM, model behaviour is checked with evals against the real model, outside `pnpm test`.
- Follows [[coding-style]] (blank lines, guard clauses).

## Checklist

- [ ] Test in `__tests__/` next to the code, named `<module>.test.ts(x)`
- [ ] No real provider calls; clients mocked with `vi.hoisted` + `vi.mock` before imports
- [ ] Webhook tests use recorded payloads with signatures
- [ ] New tenant table added to the isolation suite
- [ ] Rule-heavy logic covered with `it.each` scenarios
