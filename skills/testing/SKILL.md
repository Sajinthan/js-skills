---
name: testing
description: Writes and runs Goodo tests — Vitest unit and integration tests for services, packages and React components, Fastify inject route tests, row-level security isolation tests, Playwright end-to-end tests, and model evals. Use when adding tests, fixing test failures, mocking providers, or configuring Vitest or Playwright.
---

# Testing

_Adapted 2026-09-23 from rn-boilerplate `testing` and Debrief's `testing`. Written before the code exists: configs and paths are the plan in `docs/TASKS.md`. Run [[refine-skill]] once task 1.0 lands._

## Suites

| Command | Tool | Scope | When |
|---|---|---|---|
| `pnpm test` | Vitest | every package: logic, routes, jobs, components | every PR (CI) |
| `pnpm --filter @goodo/<pkg> test` | Vitest | one package | while working |
| `pnpm e2e` | Playwright | site pages, dashboard flows | every PR once 15.5 lands |
| `pnpm evals` | engine + real model | `evals/chat`, `evals/voice` | before a release; reported in CI, not blocking at first |

## Where tests live

Co-located `__tests__/` next to the code it tests:

```
apps/chat/src/routes/__tests__/widget-chat.test.ts
apps/worker/src/jobs/__tests__/crawl-source.test.ts
packages/engine/src/__tests__/hours.test.ts
packages/db/src/__tests__/tenant.test.ts
apps/web/src/pages/Conversations/__tests__/Conversations.test.tsx
```

Name: `<module>.test.ts(x)`. Playwright specs live in `apps/<app>/e2e/*.spec.ts`.

## Vitest config

- Node environment for services and packages; `jsdom` for `apps/web`, `apps/site` and `apps/widget` component tests.
- `globals: false`: import `describe`, `it`, `expect`, `vi` from `vitest`.
- `@/` alias mirrors the app's `tsconfig`.
- Env stubs in the config's `test.env`, never a real `.env`.

## Route tests: Fastify `inject`

Build the app, inject requests, assert on status and body. They run against the Docker Postgres (`pnpm db:up`, `pnpm db:migrate`; CI starts its own). `buildTestApp()` (`apps/api/src/test/buildTestApp.ts`):
- connects to it through `createDb`
- gives each fake Cognito user its own random `sub` (`testIdentity()`), so test files running at the same time never share an account
- deletes every account the app created when the app closes
- leaves the sign-up limit out unless built with `limitSignUps: true`, since every `inject` comes from 127.0.0.1, the address local sign-ups use too. A test that turns it on sends each request from a random `remoteAddress` (`routes/__tests__/sign-up-limits.test.ts`)

To sign in, call `POST sign-in` (the fake Cognito accepts any password) and send back the cookie. To test a session with no account, seal one with `sealSession(…, TEST_SESSION_KEY)` for a `testIdentity()`.

```ts
import { afterAll, beforeAll, describe, expect, it } from 'vitest';

import { buildTestApp } from '@/test/buildTestApp';

describe('POST /leads', () => {
  // A fake Cognito and a fixed session key; pass `{ cognito: fakeCognito({ … }, identity) }` to change one
  // call or fix the user.
  const app = buildTestApp();

  beforeAll(() => app.ready());

  afterAll(() => app.close());

  it('rejects a non-Australian mobile', async () => {
    const response = await app.inject({ method: 'POST', url: '/leads', payload: { mobile: '+1 555 0100' } });

    expect(response.statusCode).toBe(400);
  });
});
```

## Mocking providers

Never call Anthropic, Bedrock, ElevenLabs, Twilio, Stripe or SES from `pnpm test`. Mock the client module with `vi.hoisted` before importing the code under test (ESM order matters):

```ts
const { mockSend } = vi.hoisted(() => ({ mockSend: vi.fn() }));

vi.mock('@aws-sdk/client-sesv2', () => ({
  SESv2Client: class {
    send = mockSend;
  },
  SendEmailCommand: class {
    constructor(public input: unknown) {}
  },
}));

import { sendHandoffEmail } from '../handoff-email';
```

Reset in `beforeEach` (`mockSend.mockReset()`).

**Webhooks** (ElevenLabs, Twilio, Stripe) are tested with recorded payloads saved under `__tests__/fixtures/`, including the signature header, so signature checking is exercised too.

**Streaming** (`/widget/:id/chat`, `/brain`): mock the model client to yield a fixed token sequence; assert on the SSE frames in order.

## Tenant isolation tests

`packages/db/src/__tests__/tenant.test.ts` seeds two accounts and, for every tenant table, asserts that queries inside `withAccount(A)` never return B's rows, including a vector similarity query. **Any new tenant table is added to this suite in the same PR.** A failure here blocks merge.

## Data-driven tests

Scenario arrays with `it.each` for rules with many cases (hours, phone numbers, outcome mapping):

```ts
it.each([
  { at: '2026-10-06T18:00:00+11:00', expected: true },
  { at: '2026-10-05T18:00:00+11:00', expected: false }, // Monday, closed
])('isOpen at $at is $expected', ({ at, expected }) => {
  expect(isOpen(marlowsHours, new Date(at))).toBe(expected);
});
```

Test data builders live in `src/lib/test-helpers.ts` of the package that owns the type.

## Evals

`evals/chat/*.yaml` and `evals/voice/*.yaml`: a conversation plus the expected behaviour (answered from content, said "I don't know" and offered a person, refused off-topic, never confirmed a booking). They call the real model, cost money, and run with `pnpm evals`. Add one whenever a transcript shows the assistant doing something wrong.

## Conventions

- One `describe` per function or route; one `it` per behaviour; happy path and edge cases.
- `async`/`await` in test bodies, not `.then()` chains.
- Assertions: `toBe`, `toEqual`, `toContain`, `toHaveBeenCalledWith`.
- Dashboard tests render through `renderApp` (`apps/web/src/test/renderApp.tsx`): Bayside Pilates a week in by default, `bayside: 'justSetUp'` just after the first scan, or `bayside: null` to load the account from `api` stubs. Bayside's data is in `apps/web/src/test/bayside.ts`; its scans and pages come from API stubs, since the dashboard loads them itself. `apps/web/src/fixtures/` holds the dashboard's view types and blank assistants, not test data.
- Follows [[coding-style]] (blank lines, guard clauses).

## Checklist

- [ ] Test in `__tests__/` next to the code, named `<module>.test.ts(x)`
- [ ] No real provider calls; clients mocked with `vi.hoisted` + `vi.mock` before imports
- [ ] Webhook tests use recorded payloads with signatures
- [ ] New tenant table added to the isolation suite
- [ ] Rule-heavy logic covered with `it.each` scenarios
- [ ] A wrong answer seen in a transcript gets an eval
