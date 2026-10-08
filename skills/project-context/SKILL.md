---
name: project-context
description: What the product is and the decisions behind it — audiences, the rules the code must enforce, tenancy, costs and limits, and what is out of scope for the first release. A fill-in template; replace every <placeholder>. Load at the start of feature work, or before changing product rules.
---

# Project Context

Fill in each section from the project's requirements doc. Keep it short: this is what an agent must know before touching a feature, not the full spec. When a decision changes in the requirements doc, change it here in the same PR.

## What the product is

Two or three sentences: what it does, for whom, and the main surfaces (dashboard, API, site).

> `<Product>` is `<one-line pitch>`. Users `<main thing they do>` in `<surface>`.

## Who uses it

One row per audience. Say what they lead with and how they buy, so trade-offs favour the right people.

| Audience | Leads with | How they buy |
|---|---|---|
| `<audience>` | `<feature>` | `<self-serve / sales-led / trial>` |

## Rules the code must enforce

Product rules, not suggestions. Each one needs a test that fails if the rule is broken. Number them so tests and reviews can cite them.

1. `<rule>`: `<one line on why breaking it hurts>`.
2. `<rule>`: `<why>`.

## Data and tenancy

- Every tenant row has `account_id`; every tenant query runs inside `withAccount()`; row-level security is the net. A leak between tenants is the worst bug the product can have.
- No personal data (emails, phone numbers, text users typed) in logs.
- `<where data is stored, how long it is kept, what is deleted and when>`

## Costs and limits

- `<what usage is recorded per tenant, and in what unit>`
- Plans, limits and prices come from config, never code.
- `<any open endpoint that needs rate limits or caps>`

## Not in the first release

`<comma-separated list of features that are out of scope>`

If a task seems to need one of these, stop and ask.

## Vocabulary

Use the vocabulary table in your docs verbatim for statuses, kinds and roles (`todo`, `in_progress`, `done`). Never invent a synonym.

## Where the detail lives

| Topic | Document |
|---|---|
| Requirements | `<path to the requirements doc>` |
| Order of work | `<path to the task list>` |
| `<topic>` | `<path>` |

Related: [[project]], [[coding-style]], [[testing]].
