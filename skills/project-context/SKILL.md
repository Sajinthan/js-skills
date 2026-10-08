---
name: project-context
description: What Goodo is and the product decisions behind it — channels, audiences, answering rules, tenancy, data residency and the non-goals of the first release. Load at the start of any session working on Goodo features, or before changing prompts, tools, handoff, limits or billing.
---

# Project Context

_Written 2026-09-23 from `docs/PRD.md` and `docs/goodo.md`. When a decision changes there, change it here in the same PR._

## What Goodo is

One AI assistant per business that answers two channels:

- **Web chat**: a widget on the business's site; answers from the site's own content with links to the pages used; hands off to a person by email.
- **Phone**: picks up calls the staff can't (conditional forwarding on busy / no answer); answers questions, takes booking requests and messages, transfers urgent calls.

Customers turn on one channel or both. Same assistant, same facts, same handoff rules, one conversation list, one bill. Australia first; chat sells worldwide, phone is Australia only for now. The assistant is called Ruby by default, renameable per assistant.

## Who uses it

| Audience | Leads with | How they buy |
|---|---|---|
| Restaurants | Phone | Founder sets up by hand; free first month |
| Web agencies | Chat, many assistants | Self-serve; per-assistant volume price |
| Any business | Chat, one assistant | Self-serve; trial with a hard cap |

## Rules the code must enforce

These are product rules, not prompt suggestions. Each needs a test or an eval.

1. **Answer only from the assistant's own profile and pages.** No general knowledge about the business, no invented facts. No answer → say so and offer a person.
2. **Never confirm availability or a booking.** Take a request; staff confirm by SMS (request mode). A wrong confirmation is a customer at the door with no table.
3. **Never transfer to the business's public number.** The call loops back.
4. **Refuse off-topic requests** in one sentence; never reveal instructions or another tenant's data.
5. **Voice reads the profile; chat reads profile and pages.** Voice searches pages only through a tool when the profile has no answer.
6. **Opening hours are computed in code** from structured hours, timezone and public holidays, never guessed by the model.
7. **A phone channel goes live only with a reviewed profile.**

## Data and tenancy

- Every tenant row has `account_id`; every tenant query runs inside `withAccount()`; row-level security is the net, including vector queries. One leak between customers ends the company.
- All customer data, recordings and vectors stay in AWS Sydney. Accounts carry a `region` field (always `ap-southeast-2` for now).
- Recordings are deleted after 30 days; transcripts are kept. Provider training is off, retention shortest available.
- No personal data (phone numbers, emails, message text) in logs.

## Costs and limits

- Every conversation records model tokens, voice minutes, phone minutes, SMS segments and cost in cents. Pricing will be set from these numbers.
- Plans, limits and prices come from config, never code. Prices are not decided yet.
- The open widget is a free LLM endpoint unless limited: allowed domains, per-visitor and per-assistant limits, trial cap, cost alerts.

## Not in the first release

Food orders, booking-platform connectors (request mode only), customer API actions, white-label, team logins, self-serve phone setup, phone outside Australia, languages other than English, outbound calls except the demo callback, clinic packs, non-Sydney hosting, channels other than the web widget and phone.

If a task seems to need one of these, stop and ask.

## Vocabulary

Use the table in `docs/TASKS.md` verbatim (account kind, channel kind and status, outcomes, booking statuses, speakers, plans).

## Where the detail lives

| Topic | Document |
|---|---|
| Requirements (numbered) | `docs/PRD.md` |
| Order of work | `docs/TASKS.md` |
| Product reasoning | `docs/goodo.md` |
| Phone design | `~/Projects/ideas/ai-voice-architecture.md` |
| Demo page brief | `~/Projects/ideas/ai-voice-demo-site-brief.md` |
| Legal approach | `~/Projects/ideas/ai-voice-restaurants.md` (Legal) |

Related: [[project]], [[coding-style]], [[testing]].
