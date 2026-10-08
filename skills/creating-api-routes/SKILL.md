---
name: creating-api-routes
description: Adds Fastify routes to Goodo's services (apps/api now, apps/chat from G1.4, apps/voice from G2.4) — route modules, Zod schemas, tenancy — and the security rules every route follows. Use when adding or changing an endpoint or webhook, or touching auth, cookies, CORS, rate limits, error responses or request logging.
---

# Creating API Routes

_Rewritten 2026-09-24 for Goodo; the earlier version described Debrief. Verified against `apps/api` at task 1.0. Items marked **(planned, task N)** do not exist yet: build them as written when that task lands, then run [[refine-skill]]. Security rules come from a review of Debrief's API on 2026-09-24._

## Where things go

| Concern | Location |
|---|---|
| App factory: compilers, request ids, plugin registration | `apps/api/src/app.ts` (`buildApp`) |
| Start-up: env, listen, graceful shutdown | `apps/api/src/server.ts` |
| Env schema | `apps/api/src/env.ts` (via `parseEnv` from `@goodo/shared`) |
| One module per resource | `apps/api/src/routes/<resource>.ts`, kebab-case, registered in `buildApp` |
| Route tests | `apps/api/src/routes/__tests__/<resource>.test.ts` ([[testing]]) |
| Schemas a frontend also uses | `packages/shared/src/schemas/` ([[shared-validation]]) |
| Tenant data access | `withAccount()` from `@goodo/db` ([[prisma-and-migrations]]) |
| Logic beyond parse-and-call | a service ([[creating-services]]) or `@goodo/engine` |

## Route pattern

Each resource is a `FastifyPluginAsyncZod` plugin. `apps/api/src/routes/health.ts` is the reference:

```ts
import type { FastifyPluginAsyncZod } from 'fastify-type-provider-zod';
import { z } from 'zod';

const healthResponseSchema = z.object({
  status: z.literal('ok'),
});

export const healthRoutes: FastifyPluginAsyncZod = async app => {
  app.get(
    '/health',
    { logLevel: 'warn', schema: { response: { 200: healthResponseSchema } } },
    async () => ({ status: 'ok' as const }),
  );
};
```

Register it in `buildApp` with `app.register(healthRoutes)`. Handlers parse, call a service or the engine, and return; no business logic in the handler ([[coding-style]]).

## Security rules

Every rule here is a requirement, not advice. Several exist because Debrief got them wrong.

### Input

- **Schema for everything**: `params`, `querystring`, `body` and every `response` status. The response schema strips fields it doesn't list, so a new database column never leaks by accident.
- **Cap every free-text field** with `.max()`. Text reaches the model, and the model is billed per token.
- **Never trust counts, limits or ownership sent by the client.** Message counts, caps and account id come from the session and the database. Debrief took `extensionsUsed` from the browser, which let a user lift its turn limit.

### Authentication: deny by default

- **Every route needs a session and an account** unless it sets `config: { public: true }`. The auth hook (`apps/api/src/plugins/auth.ts`, added in `buildApp`) runs on every request:
  - it checks the session cookie, and asks Cognito again every 15 minutes, with an hour's grace when Cognito is down
  - it finds the account with `db.findAccountForUser()`
  - it sets `request.account` (`{ id, email }`) and adds the account id to the request's logger
- **Status codes:** no session, a forged one, or one Cognito refuses gets `401`, and the cookie is cleared. A session with no account gets `403`. Unknown routes still get `404`.
- **Public today:** `/health`, and the auth routes a person uses before they have a session (sign up, verify email, resend code, sign in, sign out, forgot and reset password). Making a route public means adding it to `PUBLIC_ROUTES` in `src/__tests__/protection.test.ts` in the same PR. That test visits every registered route and fails if any other route answers without a session.
- **Account creation:** the auth hook never creates an account. `POST verify-email` creates it once Cognito confirms the code. `POST sign-in` creates it only if it is missing.
- **Public routes must not read `request.account`.**

### Tenancy

- Every tenant query runs inside `db.withAccount(request.account.id, …)`. The account id comes from the authenticated session, never from a path, query or body.
- Another account's resource id returns `404`, the same as one that doesn't exist. Row-level security gives this for free; don't add a `403` that confirms the id exists.

### Sign-up, sign-in and sessions

- **Email must be verified** before an account can create an assistant (PRD 1). Never auto-confirm or auto-verify in a Cognito PreSignUp trigger, and reject ID tokens whose `email_verified` claim is not `true`.
- **Never link or merge accounts by email alone.** Link by Cognito `sub`.
- **Session cookie:** `__Host-` prefix, `HttpOnly`, `Secure`, `SameSite=Lax`, encrypted and authenticated (AES-256-GCM), with the key derived per purpose from a 32-byte secret loaded from the env schema.
- **Decided 2026-09-26:** the API is served on the dashboard's origin at `/api` (Vite proxy locally, CloudFront in production), so the cookie is host-only and the dashboard needs no CORS. Built in `apps/api/src/routes/auth.ts` and `lib/session.ts`: `__Host-goodo_session` holds the Cognito refresh token, never the tokens in plain form.

### Roles

- Roles come only from an explicit database field that the founder sets. Never derive a role from an email domain. Debrief made anyone who signed up with a staff-domain address an admin.
- **Admin routes** set `config: { admin: true }`. The auth hook then asks `db.isAdmin()` (the `platform_admins` table, through `app_is_admin()`) on every request and answers anyone else `404`, the same as a route that doesn't exist. Admins are added and removed only by `pnpm --filter @goodo/db admin add|remove <email>`, with the database owner's login. Test it as `src/__tests__/admin.test.ts` does.
- **No demo mode.** Missing auth config means the service refuses to start through its env schema. It never falls back to an anonymous or admin user.

### Client IP and rate limits

- **Never set `trustProxy: true`.** Fastify then returns the *first* `X-Forwarded-For` value, which the caller writes (checked 2026-09-24). Set it to the exact number of proxies in front of the service, or read `CloudFront-Viewer-Address` when CloudFront fronts it. Test with a spoofed header.
- **In-memory counters are per task** and reset on deploy. Put volume limits in an AWS WAF rate-based rule at the edge, and store cost and abuse limits in Postgres: sign-up (PRD 7), leads, widget per visitor and per assistant (PRD 44), trial cap (PRD 74), demo callback (PRD 88).
- **The origin is reachable only through the edge.** The load balancer accepts traffic from CloudFront only.
- **Sign-up is the pattern for a Postgres limit** (S1.10). The route calls `db.recordSignUp(request.ip, SIGN_UP_LIMIT)` before Cognito, and answers `429` with a message for the user when it returns false. The number is in `apps/api/src/config/sign-up-limit.ts` (20 an hour per IP; per IP only, like Firebase, Auth0 and Supabase). It logs that the limit was reached, never the address or email. `request.ip` is the connection's address until CloudFront fronts the API; switching it to `CloudFront-Viewer-Address` is task 14.17, with the WAF rule.

### CORS

Goodo has three kinds of caller. Configure CORS per route group, not with one global list.

| Caller | Credentials | Allowed origins |
|---|---|---|
| Dashboard | session cookie | exact dashboard origin from the env schema; required in production |
| Site forms (`/leads`, `/callback`) | none | the site origin and Vercel preview domains (S12.1) |
| Widget (`apps/chat`, `/widget/:assistantId/*`) | **never** | the assistant's allowed domains, checked on every request (PRD 39) |

Widget routes never read the session cookie, even if one arrives.

### Webhooks (Twilio, ElevenLabs, Stripe)

- Verify the signature against the **raw body** before parsing or acting on it (PRD 100). Reject stale timestamps where the provider signs one.
- Handle each event idempotently by its provider event id; providers retry.
- Test with recorded payloads that include the signature header ([[testing]]).

### Headers, errors and logs

- **Headers:** `@fastify/helmet` with the API's content security policy set to `default-src 'none'` and `frame-ancestors 'none'`; the API serves only JSON and SSE. Built in `buildApp` (2026-09-28).
- **Error shape:** always `{ error: string }`, including unknown routes (404). A 4xx message is written for the user: a validation error sends the schema's own message, and Fastify's developer messages (codes starting `FST_`) are swapped for plain ones. A 5xx always sends a generic message, and the real error is logged with the request id. Never send `err.message` from a 5xx; Debrief leaked Prisma and AWS error text this way. Built in `buildApp`; tested in `src/__tests__/app.test.ts`.
- **Writes are JSON only.** There is no parser for form posts, so they get `415`. With `SameSite=Lax` cookies, this is what stops another site's form posting to the API as the user. Don't add a form-body parser.
- **Logs:** request id (set up in `buildApp`) and account id (added by the auth hook) on every line. Pino `redact` covers `authorization`, `cookie`, `set-cookie`, and `password`, `code`, `email` and `refreshToken` one level into any logged object. Logged errors go through `serializeError`, which scrubs their message and stack of URL credentials, `password=`-style values, emails and phone numbers. Both are in `@goodo/shared` (`packages/shared/src/redact.ts`), and `apps/worker` uses them too. Phone numbers, emails and message text never reach a log (PRD 104): don't log a whole request body.

### Every entry point gets the same protections

`apps/chat` and `apps/voice` are separate services (PRD section 7). They need the same headers, error handler, log redaction and IP handling as `apps/api`, and those must be shared code, not a copy. Debrief's streaming Lambda skipped its Fastify protections entirely.

**Open question (decide at G1.4, when `apps/chat` is created):** the shared Fastify plugins need a package. `@goodo/shared` must stay browser-safe and `@goodo/engine` has no HTTP code, so it would be a new package such as `packages/server`, allowed for `api`, `chat`, `voice` and `worker` in `.dependency-cruiser.cjs`.

## Checklist

- [ ] Route module in `routes/<resource>.ts`, registered in `buildApp`, test in `routes/__tests__/`
- [ ] Schemas for params, query, body and every response; free text capped with `.max()`
- [ ] Nothing about identity, ownership or limits taken from the client
- [ ] Tenant queries inside `withAccount`; another account's id returns 404
- [ ] Rate limit decided: edge (WAF) or Postgres-backed; never keyed on a client-written IP
- [ ] CORS matches the caller type; widget routes carry no credentials
- [ ] Webhooks verify the signature on the raw body and are idempotent
- [ ] 5xx responses are generic; no personal data in logs
