---
name: skill-authoring-guide
description: "Guidance for creating and maintaining agent skills. Use when writing a new skill, refactoring an existing one, or auditing a skill library. Triggers: 'write a skill', 'skill too verbose', 'skill needs cleanup', 'agent skill best practice', 'skill drift', 'skill bloat'."
version: 1.0.0
license: MIT
metadata:
  tags: [skills, authoring, governance, standards, tdd]
---

# Skill Authoring Guide

## Core Principles (in priority order)

1. **Concise language (top priority)**: One word beats one sentence. Delete causal explanations, redundancy, and jargon. Ask of every paragraph: "Is this worth the token cost?"
2. **Cold-run acceptance (second priority)**: After creating or editing a skill, always run a cold test — a zero-context sub-agent executes the task using only the skill doc. If it drifts, fix the judgment anchors in the skill itself; do not add hard rules. Two consecutive clean passes = done. See `references/coldrun-subagent.md` for the sub-agent template.
3. **Give direction, not answers**: Only lock in things that genuinely must not change (invariants, fixed parameters, hard node order, response format). For everything else, give a judgment anchor and let the model reason it out in context.
   **Ratio criterion (per rule)**: Ask "If I delete this line, can a zero-context model still get the right answer from the remaining doc?"
   1. Active action can be inferred correctly → position is right, leave it.
   2. Cannot be inferred, but the line already carries a concrete value (number / path / command / parameter, including threshold-like numbers; a wrong value usually shows as an observable failure) → already fixed, leave it.
   3. Cannot be inferred, and the line carries no value → split by surface form:
      - **Should be fixed but isn't** (the sentence asks the model to pick a value, e.g. "choose optimal params / pick the right tier"): value has a source in the same doc (some line or reference) → fix it to that value; no source → leave it, mark "needs verification: <this line>" in the decision table, do not edit the doc.
      - **Anchor too vague** (the sentence only points to a source, e.g. "params = recipe / see recipe"): sharpen to "follow <doc> section X (value = <concrete>)" — cite the value, not just the file.
4. **Anchors must be sharp**: Write to the point where the model can execute the judgment directly. Never write "params = recipe" type vague pointers.
5. **Troubleshooting: linear diagnosis flow, not "symptom → conclusion" lookup table**: Write "check X first, then Y" as a fixed sequence. New errors not in the table are still diagnosed by following that sequence. Do not memorize conclusions.

## Workflow (Write → Audit → Cold-run Loop)

1. **Start**: New or edited skill → load this guide.
2. **Write**: TDD (RED → GREEN → REFACTOR). Concise, sharp anchors, linear diagnosis, de-vagueness.
3. **Audit**: Run `skill-audit` (or this guide's 9 checks) against the skill.
4. **Judge**: All checks pass AND ratio (fixed-where-should-be / sharp-where-should-be-thoughtful) is right?
   - Pass → backfill `cold_run` field in frontmatter, done.
   - Fail → back to "Write", fix the judgment anchors themselves (do not add hard rules) → two clean cold-runs → back to "Audit".

> Cold-runs belong to the **authoring side** (TDD step 4, before delivery); `skill-audit` only *reads* `cold_run`, it does not re-run.

## When to Write a Skill

- **Write**: technique not intuitive, cross-project reuse, universal pattern.
- **Don't write**: one-off solution, standard practice already documented elsewhere, mechanical constraint enforceable by regex / validation.
- **One skill, one job**: skill = single main flow; a separable sub-step (periodic maintenance, off-main-path task) = mixed responsibility → not compliant, split out or delete.

## Frontmatter

- `description`: only "when to use + trigger keywords". Do not summarize the process (the agent routes on `description` and skips the body).
- Keywords = literal words the agent will search for: verbatim error messages, symptoms (flaky / hanging), command names, synonyms.
- Negative triggers: `description` must contain ≥1 sentence about "what NOT to use for". 0 sentences = not compliant.
- Cap: 200 characters (injected into every turn), write enough, do not pad to the limit.
- Body changes require a `version` bump. After two clean cold-runs, backfill the `cold_run` field:
  ```yaml
  version: 1.0.0
  cold_run: "v1.0.0 @ <YYYY-MM-DD> | sub-agent cold-run"
  ```
  Bump version + one edit to the body → the old `cold_run` is stale (version mismatch) → must re-cold-run before next backfill. `cold_run` is a "which version was verified" marker, not a history log — keep only the current version's entry.

## Body Structure

- Main file = index + flow (< 500 lines). Move heavy reference material (100+ line API tables / pricing tables / command sections) to `references/`. Reusable code → `scripts/`. Main file keeps only navigation lines.
- Organize by process node, imperative voice (what to do + judgment point), no causal explanations.
- Troubleshooting: linear diagnosis flow (check X first, then Y, fixed order), not "symptom → conclusion" lookup. New errors not in table: follow the same order. Do not memorize.

## Writing Conventions (one line each, no elaboration)

- Imperative: "Extract text", not "I will / you should".
- Give direction not answers: write "what to do + judgment point", delete "why" causal explanations.
- Judgment anchors sharp: write to the point where the model can execute the judgment directly (e.g. "grep the line containing X from process list, verify all expected params are present"). Never write "params = recipe".
- De-vagueness: no "if needed / optional / ideally". Write "always X" or "only when Y, do X".
- Plain language: no jargon ("smoke test" → "first-words verification").
- Minimize scripts / py: if a plain shell command works directly, don't wrap it. Scripts only for pure mechanical batch work.
- Delete what the model already knows; do not re-teach what the agent reliably does.
- One excellent example > many mediocre ones.
- No narrative ("last time because X we did Y" → "do Y because X"). No date stamps.
- Do not document command flags. Write "run `--help`".
- Cross-skill references: use `**REQUIRED SUB-SKILL:**` marker, not vague "see xxx".
- Script stderr must be human-readable and specific ("file X not found, check path Y"), not a bare "failed".

## TDD Process

1. **RED**: Before writing the skill, let an agent run the scenario without it. Record how it fails. Execution = zero-context sub-agent (e.g. `delegate_task` or equivalent) runs the task with the user's original wording and reports the failure verbatim / broken steps. Never write a skill you haven't watched fail.
2. **GREEN**: Write rules only for the specific failures the baseline exposed. Do not add rules for hypothetical cases.
3. **REFACTOR**: Agent finds a new loophole → add an explicit countermeasure, retest until airtight.
4. **Cold-run acceptance**: Zero-context sub-agent executes the skill doc only, checks for drift; if drifted, **fix the judgment anchor itself** (sharpen it), do not add hard rules to block. Two consecutive clean passes = done; pass → backfill frontmatter `cold_run` (version + date + method). See `references/coldrun-subagent.md`.

**Form match** (pick the right writing style; do not apply mechanically):

| Failure type | Correct form | Wrong form |
|---|---|---|
| Skips rule under pressure | Prohibition + rationalization table + red flags | Soft guidance ("suggest…", "consider…") |
| Output shape wrong | Positive recipe (describe what the output IS) | Prohibition list ("don't repeat", "don't narrate") |
| Omits required item | Add REQUIRED slot in template | Prose reminder |
| Behavior should vary by condition | Conditional bound to observable predicate | Unconditional rule + exemption clause |

## Pre-deployment Checks (walk in order, not tick in order)

1. TDD complete (RED ran, GREEN compliant, REFACTOR plugged).
2. Structural validation (frontmatter, name, description ≤200 chars, body <500 lines, no dead links / dead references).
3. Content quality (no narrative, single source of truth, REQUIRED markers present, no context-dependent residue).
4. Cold-run acceptance (zero-context sub-agent, two clean passes) → backfill frontmatter `cold_run` (version + date + method). Hit 2/3 → fix, re-cold-run.

## Community Sources (read on demand)

Five references under `references/`: coldrun-subagent (cold-run 4 steps + sub-agent template, use directly in step 4), obra-writing-skills (TDD method), mgechev-skills-best-practices (concise structure), anthropic-skill-authoring-best-practices (conciseness principle), agentskills-spec (open standard). Read when making boundary judgments; they do not enter resident context. When sources conflict with this doc, **this doc wins**; sources are for provenance of criteria, not body evidence.
