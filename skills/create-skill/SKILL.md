---
name: create-skill
description: Authors a new skill for the repo in .claude/skills/ — the quality gate, structure and registration. Use when adding a skill, capturing a repeated convention, or the user says "make a skill for this" / "/create-skill".
---

# Create Skill

Skills encode **repo-specific knowledge a model doesn't already have**, so the agent gets it right first time. This skill is how you add one. See [[refine-skill]] to improve an existing one.

## The quality gate (write this down first)

Answer in one sentence:

> **What does this skill encode that the model wouldn't already know?**

If the answer is "how to use Fastify / Prisma / React / Next.js", **stop**. That is a tutorial the model already has. A skill is worth writing only when it captures one of:

- a **repo convention**: where files live, which package exports what, naming;
- a **non-obvious wiring**: two files that must stay in sync, a start-up order, a row-level security step;
- a **gotcha** paid for in real debugging, stated as the fix with the symptom;
- a **decision** with a rationale the code doesn't state (link the project's requirements doc).

Subjective rules must become **measurable checks**. "Keep routes thin" is noise; "handlers only parse input and call a service; no Prisma import in `routes/`" is a skill. If a rule can't be made checkable, cut it.

## Layout

```
.claude/skills/<kebab-name>/
└── SKILL.md
```

Claude Code loads skills from `.claude/skills/`. Name in kebab-case; the folder name is the invocation (`/coding-style`).

## Frontmatter (required)

```md
---
name: <kebab-name>
description: <what it does>. Use when <trigger phrases the agent will match on>.
---
```

The `description` decides whether the agent loads the skill. Lead with what it does, then an explicit **"Use when …"** listing the tasks and phrases that should trigger it. Name the app or package it applies to (`apps/api`, `apps/web`, `packages/db`) so it doesn't load for the wrong one.

## Body: short, scannable, executable

- One paragraph stating the rule or decision.
- **Tables** for file-to-role maps; **code snippets copied from this repo**, not invented.
- The gotchas, stated as the fix with the symptom.
- A **Checklist** of the measurable rules at the end.
- Link related skills with `[[skill-name]]`.

One fact per section. If a section restates what the model knows, delete it.

## Every claim must be executable

**No aspirational tooling.** Before committing, grep every path, symbol, env var and script the skill names; run `pnpm typecheck` on any snippet you adapted. A skill that names a flag that doesn't exist is worse than no skill.

Exception while the repo is being built: a skill written ahead of the code must say so in an italic note under the title and name the task in the project's task list after which it gets verified with [[refine-skill]].

## Register and verify

1. Add the name to the skills list in `CLAUDE.md`.
2. If it changed code, run `pnpm typecheck && pnpm lint && pnpm test`.
3. Sanity-check the trigger: would the `description` make *you* load this skill for the task it's for?

## Checklist

- [ ] One-sentence answer to "what does this encode that the model doesn't know?"
- [ ] Subjective rules rewritten as measurable checks, or cut
- [ ] Frontmatter `name` matches the folder; `description` has an explicit "Use when" and names its app or package
- [ ] Snippets copied from this repo; every path, symbol and flag verified (or marked as ahead of the code)
- [ ] Registered in `CLAUDE.md`; related skills `[[linked]]`
- [ ] No framework tutorial; no aspirational tooling
