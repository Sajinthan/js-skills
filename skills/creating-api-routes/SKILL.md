---
name: creating-api-routes
description: Adds Fastify routes to apps/api and any other Fastify service in the monorepo — route modules, Zod schemas, tenancy — and the security rules every route follows. Use when adding or changing an endpoint or webhook, or touching auth, cookies, CORS, rate limits, error responses or request logging.
---

# Creating API Routes

Paths below are for `apps/api`. Any other Fastify service uses the same layout under `apps/<service>/src/`.

## Where things go

| Concern | Location |
|---|---|
| App factory: compilers, request ids, plugin registration | `apps/api/src/app.ts` (`buildApp`) |
| Start-up: env, listen, graceful shutdown | `apps/api/src/server.ts` |
| Env schema | `apps/api/src/env.ts` (via `parseEnv` from `@repo/shared`) |
| One module per resource | `apps/api/src/routes/<resource>.ts`, kebab-case, registered in `buildApp` |
| Route tests | `apps/api/src/routes/__tests__/<resource>.test.ts` ([[testing]]) |
| Schemas a frontend also uses | `packages/shared/src/schemas/` ([[shared-validation]]) |
| Tenant data access | `withAccount()` from `@repo/db` ([[prisma-and-migrations]]) |
| Logic beyond parse-and-call | a service ([[creating-services]]) |

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

Register it in `buildApp` with `app.register(healthRoutes)`. Handlers parse, call a service, and return; no business logic in the handler ([[coding-style]]).

## Security rules

Every rule here is a requirement, not advice.

### Input

- **Schema for everything**: `params`, `querystring`, `body` and every `response` status. The response schema strips fields it doesn't list, so a new database column never leaks by accident.
- **Cap every free-text field** with `.max()`. Unbounded text costs storage, and money wherever it reaches a paid API.
- **Never trust counts, limits or ownership sent by the client.** Usage counts, caps and account id come from the session and the database. A counter taken from the browser lets a user lift their own limit.

### Authentication: deny by default

- **Every route needs a session and an account** unless it sets `config: { public: true }`. The auth hook (`apps/api/src/plugins/auth.ts`, added in `buildApp`) runs on every request:
  - it checks the session cookie, and re-checks it with the auth provider every 15 minutes, with an hour's grace when the provider is down
  - it finds the account with `db.findAccountForUser()`
  - it sets `request.account` (`{ id, email }`) and adds the account id to the request's logger
- **Status codes:** no session, a forged one, or one the auth provider refuses gets `401`, and the cookie is cleared. A session with no account gets `403`. Unknown routes still get `404`.
- **Public routes:** `/health`, and the auth routes a person uses before they have a session (sign up, verify email, resend code, sign in, sign out, forgot and reset password). Making a route public means adding it to `PUBLIC_ROUTES` in `src/__tests__/protection.test.ts` in the same change. That test visits every registered route and fails if any other route answers without a session.
- **Account creation:** the auth hook never creates an account. `POST verify-email` creates it once the auth provider confirms the code. `POST sign-in` creates it only if it is missing.
- **Public routes must not read `request.account`.**

### Tenancy

- Every tenant query runs inside `db.withAccount(request.account.id, …)`. The account id comes from the authenticated session, never from a path, query or body.
- Another account's resource id returns `404`, the same as one that doesn't exist. Row-level security gives this for free; don't add a `403` that confirms the id exists.

### Sign-up, sign-in and sessions

- **Email must be verified** before an account can use the product. Never auto-confirm or auto-verify in an auth provider hook, and reject ID tokens whose `email_verified` claim is not `true`.
- **Never link or merge accounts by email alone.** Link by the auth provider's subject id (`sub`).
- **Session cookie:** `__Host-` prefix, `HttpOnly`, `Secure`, `SameSite=Lax`, encrypted and authenticated (AES-256-GCM), with the key derived per purpose from a 32-byte secret loaded from the env schema.
- **Same origin:** the API is served on the dashboard's origin at `/api` (Vite proxy locally, the CDN or reverse proxy in production), so the cookie is host-only and the dashboard needs no CORS. The cookie (`__Host-session`, built in `routes/auth.ts` and `lib/session.ts`) holds the provider's refresh token encrypted, never tokens in plain form.

### Roles

- Roles come only from an explicit database field that an operator sets. Never derive a role from an email domain; that makes anyone who signs up with a staff-domain address an admin.
- **Admin routes** set `config: { admin: true }`. The auth hook then asks `db.isAdmin()` (a `platform_admins` table, read through a `SECURITY DEFINER` function) on every request and answers anyone else `404`, the same as a route that doesn't exist. Admins are added and removed only by an operator script run with the database owner's login. Test it as `src/__tests__/admin.test.ts` does.
- **No demo mode.** Missing auth config means the service refuses to start through its env schema. It never falls back to an anonymous or admin user.

### Client IP and rate limits

- **Never set `trustProxy: true`.** Fastify then returns the *first* `X-Forwarded-For` value, which the caller writes. Set it to the exact number of proxies in front of the service, or read the CDN's own viewer-address header. Test with a spoofed header.
- **In-memory counters are per instance** and reset on deploy. Put volume limits in a rate-based rule at the edge (WAF), and store cost and abuse limits in Postgres: sign-up, public form posts, per-account usage caps, trial limits.
- **The origin is reachable only through the edge.** The load balancer accepts traffic from the CDN only.
- **Sign-up is the pattern for a Postgres limit.** The route calls `db.recordSignUp(request.ip, SIGN_UP_LIMIT)` before the auth provider, and answers `429` with a message for the user when it returns false. The number lives in `apps/api/src/config/sign-up-limit.ts` (per IP, per hour). It logs that the limit was reached, never the address or email.

### CORS

Configure CORS per route group, not with one global list.

| Caller | Credentials | Allowed origins |
|---|---|---|
| Dashboard | session cookie | exact dashboard origin from the env schema; required in production |
| Public site forms (e.g. `/leads`) | none | the site origin and its preview domains |
| Public cross-origin clients (embeds, customers' sites) | **never** | the account's allowed origins, checked on every request |

Routes for public cross-origin clients never read the session cookie, even if one arrives.

### Webhooks

- Verify the signature against the **raw body** before parsing or acting on it. Reject stale timestamps where the provider signs one.
- Handle each event idempotently by its provider event id; providers retry.
- Test with recorded payloads that include the signature header ([[testing]]).

### Headers, errors and logs

- **Headers:** `@fastify/helmet` with the API's content security policy set to `default-src 'none'` and `frame-ancestors 'none'`; the API serves only JSON and SSE. Set up in `buildApp`.
- **Error shape:** always `{ error: string }`, including unknown routes (404). A 4xx message is written for the user: a validation error sends the schema's own message, and Fastify's developer messages (codes starting `FST_`) are swapped for plain ones. A 5xx always sends a generic message, and the real error is logged with the request id. Never send `err.message` from a 5xx; it leaks Prisma and SDK error text. Set up in `buildApp`; tested in `src/__tests__/app.test.ts`.
- **Writes are JSON only.** There is no parser for form posts, so they get `415`. With `SameSite=Lax` cookies, this is what stops another site's form posting to the API as the user. Don't add a form-body parser.
- **Logs:** request id (set up in `buildApp`) and account id (added by the auth hook) on every line. Pino `redact` covers `authorization`, `cookie`, `set-cookie`, and `password`, `code`, `email` and `refreshToken` one level into any logged object. Logged errors go through `serializeError`, which scrubs their message and stack of URL credentials, `password=`-style values, emails and phone numbers. Both live in `@repo/shared` (`packages/shared/src/redact.ts`), and `apps/worker` uses them too. Personal data (emails, phone numbers, user-written text) never reaches a log: don't log a whole request body.

### Every entry point gets the same protections

Every Fastify service needs the same headers, error handler, log redaction and IP handling as `apps/api`, and those must be shared code, not a copy. Put the shared plugins in a server-only package, never in `@repo/shared`, which must stay browser-safe.

## Checklist

- [ ] Route module in `routes/<resource>.ts`, registered in `buildApp`, test in `routes/__tests__/`
- [ ] Schemas for params, query, body and every response; free text capped with `.max()`
- [ ] Nothing about identity, ownership or limits taken from the client
- [ ] Route needs a session, or is public and listed in `PUBLIC_ROUTES`
- [ ] Tenant queries inside `withAccount`; another account's id returns 404
- [ ] Rate limit decided: edge (WAF) or Postgres-backed; never keyed on a client-written IP
- [ ] CORS matches the caller type; public cross-origin routes carry no credentials
- [ ] Webhooks verify the signature on the raw body and are idempotent
- [ ] 5xx responses are generic; no personal data in logs
