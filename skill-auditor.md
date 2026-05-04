
---
name: skill-auditor
description: Audit Claude SKILL.md files for quality, abstraction-fit, and verbosity. Use when the user asks to review, audit, critique, grade, or score skills under `.claude/skills/`.
argument-hint: "[path-to-skill-or-directory]"
---

# Skill Auditor

## Inputs

- `$ARGUMENTS` — path to either a single `SKILL.md` file, or a directory containing skills (recursively scans `**/SKILL.md`).
- Default if `$ARGUMENTS` is empty: `.claude/skills/`.

## Output

- A markdown report at `SKILLS-AUDIT.md` (project root) by default. The user may specify a different path in `$ARGUMENTS` after the input path.
- The report contains, in order: per-skill scorecard table, cross-cutting issues (multi-skill mode only), top 5 fixes, what's well-designed, then per-skill detail sections.

## Rules

- **Critical, not glowing.** Honest critique is the value. Affirm strengths in a separate section so they aren't punished by silence.
- **Flag, don't fix.** Each finding includes dimension, severity, and a concrete remediation — but the auditor never edits the audited skill files.
- **Multi-skill mode synthesizes cross-cutting patterns.** Single-skill mode produces a per-skill section only and skips cross-cutting.
- **Don't overwrite without confirmation.** If the report file already exists, **STOP — WAIT** and ask: overwrite, append, or pick a new path.
- **Defer to project conventions over global ones** when they conflict (e.g., custom HITL marker vocabulary, non-standard section names that are consistent across the project).

## Audit dimensions

### 1. Frontmatter quality

- `name` present, lowercase-hyphen, matches the folder name, not generic (`helper`, `utils`, `tool`).
- `description` is **one line**, third person, contains both *what* and *when*, with concrete trigger words a user would say.
- Multi-line YAML values are a silent footgun (see GH #9817) — flag any.
- Side-effecting skills (deploy, push, ship, commit, delete) should set `disable-model-invocation: true`.
- Optional: `allowed-tools`, `when_to_use`, `argument-hint`, `paths`, `model`, `effort`.

Severity: missing description = **high**; vague description that won't trigger reliably = **medium**; missing optional fields = **low**.

### 2. Body structure

- Standard section vocabulary: `Inputs → Output → Rules → Steps → HITL checkpoints`. Section names may vary; structural shape should match.
- Multi-step skills should include a copy-pasteable `- [ ]` progress checklist so the model tracks where it is across compactions.
- Section order should be consistent across sibling skills in the same project.

Severity: unstructured prose body = **high**; missing one section = **medium**; non-standard names but consistent across project = **low**.

### 3. Length and density

- Under ~500 lines. Above that → split into sibling reference files (one level deep, with a TOC if >100 lines).
- Above ~15 lines of substantive content — below that an inline prompt would do the same job, and the slash-command overhead isn't earning its keep.
- Reference chains deeper than one level → the agent partial-reads them and misses content.

Severity: >500 lines = **high**; <15 substantive lines = **medium**; multi-level reference chain = **medium**.

### 4. HITL gate consistency

- Three markers (project convention): `STOP — WAIT` (approval gate), `HUMAN TRIGGER` (active user work), `NO STOP — Agent-owned` (proceeds).
- Every irreversible operation (commit, push, file deletion, network call, package install) should have an explicit gate.
- For irreversible ops, look for a validator → fix → repeat loop nearby.
- Markers should be used consistently within and across skills.

Severity: irreversible op without a gate = **high**; inconsistent marker vocabulary = **medium**.

### 5. Standing instructions, examples, and *why*

- Phrased as standing rules ("Read the requirements file") not one-shots ("Now read the file"). One-shot phrasings rot after compaction.
- Critical format/output rules have **concrete examples** (input → output pairs).
- Rules **explain *why***. Bare ALL-CAPS ALWAYS/NEVER without rationale is a yellow flag — the model can't generalize past the literal rule.
- **Negative-phrasing rules** ("Don't X", "Never X", "Avoid X", "No X") rephrase as positive standing instructions where possible. Models follow positive rules ("Quote the bullet text verbatim") more reliably than the equivalent negative ("Don't paraphrase bullets"). Keep negatives only where the rule is a genuine prohibition with no positive equivalent (security boundary, irreversible action, etc.) — and pair with rationale. The auditor **lists every negative-phrased rule found** in each skill's per-skill detail under a `Negative-phrasing rules` block, with a positive-rewrite suggestion or a "keep — genuine prohibition" note.

Severity: ALL-CAPS commandments without why = **medium**; missing examples on format-strict rules = **medium**; one-shot phrasing throughout = **medium**; dense negative-phrasing = **medium**; isolated negative phrasing with clear rationale = **low**.

### 6. Anti-patterns

- **Time-sensitive language** ("after August 2025…") — rots; put deprecated info under `<details>`.
- **Fossilized paths** to specific projects/days/artifacts — break when files move.
- **Menu of options** ("you could use X or Y or Z") — pick a default; offer escape hatches separately.
- **Magic constants** ("set retries to 3") without rationale.
- **Dead references** to external numbering schemes the agent doesn't load (e.g., `step 05b.4`).
- **Dead file references** — `## Reference` entries pointing to files that have been deleted or moved. Verify each referenced path exists.
- **Windows-style backslash paths** in cross-platform projects.
- **Mixed methodology and recipe** — methodology fragments belong in process docs; the skill should reference, not embed, them.
- **Off-workflow skill/file references** — naming a sibling skill or concrete file path that isn't actually invoked, read, or written by the workflow. Each `/skill-name` and named path should map to an input the skill reads, an output it writes, or a step that invokes it. Orienting-prose mentions that only duplicate a later use-site mention also count — drop the forward reference and let the use site speak.
- **Project-specific vocabulary** — terms tied to one project's lifecycle, structure, or naming conventions: e.g., `day-3` / `day-N`, `Phase 5`, "aggregate across days," project-specific module/feature names like `tasks-service`. Unlike fossilized paths, these don't break file resolution — they break the skill's portability the moment it's reused in a project that doesn't have "days" or "Phase 5." Replace with project-shape-agnostic phrasing ("aggregate across runs," "across prior outputs") or generic placeholders (`<phase>`, `<feature-slug>`).

Severity: dead refs = **high**; dead file references = **high**; fossilized paths, methodology bleed, off-workflow references, or project-specific vocabulary = **medium**; menus or magic constants = **low**.

### 7. Abstraction-fit

- **Bullseye:** one input shape → one output artifact (or one bounded interactive session), one HITL contract, repeatable across multiple invocations.
- **Misfit signals**, with the better home for each:

| Signal in the skill | Better home |
|---|---|
| Pure facts, no steps | CLAUDE.md (or `user-invocable: false` reference skill) |
| Three-line procedure | Bash script or inline prompt |
| Branching state across sessions | Orchestrator with skills as primitives |
| Always-fires-on-event | Hook (PostToolUse, Stop, etc.) |
| Pure code/data work | MCP server or bash script |
| Judgment-heavy investigation | Sub-agent dispatch + methodology doc (the project's Phase 4 pattern) |
| One-shot, never repeated | Inline prompt or runbook |
| Multi-artifact bundle | Split into multiple skills, one per artifact |
| Sibling-dispatch / failure routing | Orchestrator step (skills don't dispatch each other; artifacts are the seams) |
| Methodology >50% of body | Move methodology to `process/` doc, link from skill |

- Verdict: **Bullseye** / **Acceptable** / **Misfit**.
- Recommendation: Keep / Refine boundary / Split / Demote to doc / Promote to orchestrator-step / Move to other surface.

Severity: misfit shape = **high**; orchestrator routing leaked into skill = **high**; acceptable-but-refine = **medium**.

### 8. Verbosity and language

- Filler vs specification ratio. Politeness, hedging, restatement of model-known background = filler.
- **Imperative voice** preferred ("Run the build" not "You should run the build").
- **Distractor risk** — near-paraphrases of wrong answers are worse than unrelated boilerplate (Chroma "Context Rot," 2025). Audit critical instructions for nearby look-alike-but-wrong content.
- **Opening descriptor paragraph** — a prose paragraph right after the `# heading` that restates what the frontmatter `description` already says. Remove it; the description field is the summary.
- **`## Why it earns a skill` section** — justification for the skill's existence. Remove entirely; skills don't justify themselves.
- **`## See also` section** prefaced with "this skill does not invoke them — documentation pointers only" — pure cross-reference prose with no instructions. Remove the section; sibling skills are discoverable without it.
- **Inline `*Why:*` rationale lines** embedded inside rules — e.g., `- Do X\n  - *Why:* because Y`. The rationale is justification, not instruction. Remove the `*Why:*` sub-bullet; keep the rule.
- **`## When to run` / `## When to skip` sections** — duplicates what the frontmatter `description` already says about trigger conditions. Remove; the description field is the canonical trigger.
- **`## Notes` meta-commentary sections** — self-referential observations about the skill's own design ("this skill is itself a bullseye fit"). Remove; these add no operational instruction.

Severity: opening descriptor, `## Why it earns a skill`, `## See also` (non-invoking), inline `*Why:*` lines, `## When to run`, `## Notes` meta-commentary = **medium**; high filler ratio = **medium**; no imperative voice = **low**; distractors near critical instructions = **high**.

## Steps

- [ ] 1. **Resolve input path.** If `$ARGUMENTS` is a file → single-skill mode. If a directory → multi-skill mode (find all `**/SKILL.md` recursively). If empty → default to `.claude/skills/`.
- [ ] 2. **Parse each skill.** Extract YAML frontmatter and body sections. Count lines of substantive content (excluding blank lines and pure section headers).
- [ ] 3. **Run all 8 dimensions** per skill. For each finding, record: dimension, severity (low/med/high), one-line description, concrete remediation.
- [ ] 4. **Multi-skill mode only — synthesize cross-cutting patterns.** Issues recurring across ≥3 skills go into the cross-cutting section. Common ones: fossilized paths, inconsistent HITL markers, missing `## Inputs` sections, orchestrator leakage.
- [ ] 5. **Compile the top 5 fixes** ordered by impact (severity × number of skills affected). Each entry: what to change, where, why it's #N priority.
- [ ] 6. **Compile "what's well-designed."** Affirm patterns to keep — don't punish wins with silence.
- [ ] 7. **Resolve report path.** Default `SKILLS-AUDIT.md` at the repo root. **STOP — WAIT** if the file exists: overwrite, append, or pick a new path.
- [ ] 8. **Write the report** in the format below.
- [ ] 9. **HUMAN TRIGGER — review the audit.** The user opens the report file and either replies *approve* or names specific verdicts to revise. The auditor accepts pushback and re-runs affected dimensions if the user disputes a call.

## Report format

```markdown
# Skills Audit

Run: <date>. Scope: <single skill | N skills under <path>>.

## Per-skill scorecard

| Skill | Lines | Frontmatter | Structure | Length | HITL | Anti-patterns | Abstraction-fit | Max severity | Top issue |
|---|---|---|---|---|---|---|---|---|---|
| ... | ... | ✓/⚠/✗ | ... | ... | ... | ... | Bullseye/Acceptable/Misfit | low/med/high | one-line summary |

The **Max severity** column reflects the highest-severity finding for that skill — per-skill detail below may list multiple findings of varying severity.

## Cross-cutting issues  *(multi-skill mode only)*

[patterns recurring across ≥3 skills, with affected skills listed]

## Top 5 fixes

1. [what + where + why #1]
2. ...

## What's well-designed

[affirmations — patterns to keep]

## Per-skill detail

### <skill-name>

[all findings for this skill, grouped by dimension, with severity and remediation]

#### Negative-phrasing rules

| # | Quoted negative rule | Suggested positive rewrite (or "keep — genuine prohibition") |
|---|---|---|
| 1 | "Don't paraphrase bullets" | "Quote bullets verbatim" |
| 2 | "Never push to main without approval" | keep — genuine prohibition with rationale |
```

