---
name: refine-skill
description: Improves an existing skill in .claude/skills/ — catches drift, verifies every claim against the current repo, and tightens it. Use when a skill is stale, wrong or bloated, after a refactor or dependency upgrade, when a skill was written ahead of the code and its task has landed, or the user says "update/fix this skill" / "/refine-skill".
---

# Refine Skill

Skills rot: a refactor moves a file, an upgrade renames an API, a section collects padding. Refining keeps them **true and tight**. To write a new one, see [[create-skill]].

## When to refine

- A skill names a file, symbol, env var, script or flag that changed or is gone.
- A skill carries a "written before the code exists" note and the task it names in the project's task list is done.
- The same gotcha was hit **twice**: promote it into the skill.
- A section restates framework knowledge, duplicates another skill, or describes tooling that doesn't exist.
- The skill is long enough that the agent skims past the one rule that matters.

## Process

1. **Read the whole skill**, then re-apply the quality gate: what does it encode that the model doesn't already know? Anything failing it is a deletion candidate.
2. **Verify every concrete claim against the repo.** For each path, symbol, flag or script:

   ```bash
   ls apps/api/src/routes/projects.ts             # path still exists?
   grep -rn "withAccount" packages/db/src          # symbol still there?
   grep -n '"typecheck"' package.json              # script still named that?
   ```

   For an adapted snippet, compare it with the real source or typecheck it. A claim that no longer holds gets fixed, not softened.
3. **Fix drift**: update paths, names and signatures; a contradiction between skill and repo is a finding, not a nuisance.
4. **Tighten**: one fact per section; cut padding, framework tutorial and anything duplicated in another skill (link `[[it]]`); turn remaining subjective rules into checks.
5. **Add newly earned gotchas** as the fix plus the symptom.
6. **Remove the "written before the code exists" note** once every claim has been verified.
7. **Check the router**: does the `description`'s "Use when" still match the tasks this skill should trigger on?

## Guardrails

- **Don't invent.** If a claim can't be verified, cut it or flag it to the user; never restate a plausible API from memory.
- **Smaller is a win.** A refine that only deletes stale content and fixes three paths is a good refine.

## Verify

- Re-grep every claim you touched: zero references to the old name or path.
- If any code changed, run `pnpm typecheck && pnpm lint`.

## Checklist

- [ ] Quality gate re-applied; content that fails it is gone
- [ ] Every path, symbol, flag and script re-verified against the repo
- [ ] Drift fixed; contradictions surfaced, not smoothed over
- [ ] Padding, duplication and tutorial removed; `[[links]]` instead of repeats
- [ ] "Written before the code exists" note removed if everything now checks out
- [ ] `description` "Use when" still matches the skill's scope
- [ ] Nothing invented from memory
