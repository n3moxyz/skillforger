---
name: skillforge
description: >
  Build or complete the closed-loop quality system for any skill.
  Two modes: CREATE (new skill from scratch) or FORGE (wrap an existing skill
  with rubric, judge, calibration, and wiring).
  Use when: creating a new skill, or when an existing skill needs its quality loop built.
  NOT for: editing skill logic directly (edit the SKILL.md), or scoring outputs (that's the judge).
---

# Skill Forge

Build production-ready skills with full quality control in one pass.

Every skill follows the **closed-loop pattern**: the skill runs, a judge scores the output against a rubric, scores feed a calibration file, and the calibration file feeds back into the next run. Skillforge builds all of that.

## Modes

- **CREATE**: `$skillforge create <description>` — Build a new skill from scratch. Writes SKILL.md, rubric, judge skill, calibration file, and all wiring.
- **FORGE**: `$skillforge forge <skill-name>` — Take an existing skill and build the missing quality loop pieces around it. Audits what exists, builds what's missing.
- **REBUILD**: `$skillforge rebuild <skill-name>` — Re-read the skill's latest output, re-score it, identify quality gaps, and rewrite the SKILL.md to fix them. Used when a skill exists with its full loop but the output quality needs improvement.

If no mode is specified, auto-detect:
- If a SKILL.md already exists for the named skill -> FORGE
- Otherwise -> CREATE

## Hard Rules

- NEVER skip the test invocation. Every skillforge run ends with a real test.
- NEVER skip wiring. If the skill isn't registered and connected to its judge, it doesn't exist.
- Rubrics MUST have weighted criteria that sum to 100%.
- Rubrics MUST include mandatory verification steps (mechanical checks before scoring).
- Rubrics MUST include a `## Focus Criteria` section (starts empty with v1.0 — the evolving focus system).
- Judge skills MUST follow the 3-part pattern: find output -> score quality -> update calibration.
- Calibration files start with a `## Distilled Rules` section (empty) and `## Recent Quality Issues` section (empty).
- Pass bar is 7.0/10 unless the skill warrants a different bar (justify in the rubric).
- All file paths must be absolute.

---

## Step 1 — Understand the Skill

### If FORGE or REBUILD mode:

1. Read the existing SKILL.md.

2. Find the most recent output. Check the skill's configured output path, temp directories, or any output path mentioned in the SKILL.md.

3. Read the output. Understand what the skill produces, who it's for, and what "good" looks like.

4. Audit what exists vs what's missing:

| Piece | Required |
|-------|----------|
| SKILL.md | YES |
| Rubric | YES |
| Judge skill | YES |
| Calibration file | YES |
| Registry/wiring entries | YES |

5. List which pieces are missing. Build only what's missing (don't overwrite working pieces).

### If CREATE mode:

1. Parse the description. Identify:
   - What the skill does (one sentence)
   - Who triggers it (cron, user command, or automated)
   - What the output looks like (message, file, both)
   - What data sources it needs

2. All pieces are missing. Build everything.

### If REBUILD mode:

1. Read the SKILL.md and the most recent output (same as FORGE step 1-3).
2. Read the existing rubric and calibration file.
3. Read the most recent judge score JSON if one exists.
4. Identify the quality gaps from the score + calibration + your own reading of the output.

---

## Step 2 — Build the Rubric

The rubric is the source of truth for quality. Build it first — it shapes everything else.

Create the rubric at: `<judge-skill-dir>/references/rubric.md`

### Rubric Template

```markdown
# <Skill Name> Rubric — Quality Scoring

Score each criterion 0-10. Provide specific evidence for every score.

## Context

<1-2 sentences: what this skill produces and what the core quality differentiator is.>

## Criteria

| # | Criterion | Weight | What to check |
|---|-----------|--------|---------------|
| 1 | <name> | <weight>% | <Specific, measurable check. Include VERIFY steps where mechanical checking is possible.> |
| 2 | ... | ... | ... |

<Criteria should cover: data accuracy (if applicable), core output quality, communication quality, and actionability. 6-9 criteria is the sweet spot.>

## Mandatory Verification Steps (run BEFORE scoring)

<List mechanical checks the judge must run before scoring. Examples: verify data against source, count format violations, check required sections exist.>

## Hard Caps

<List conditions that cap the overall score regardless of other criteria. Example: wrong source data = overall capped at 4.0.>

## Focus Criteria

_No active focus criteria. Version: v1.0_

<Focus criteria are added by the Judge based on scoring patterns. Max 2 active at a time. Each gets 5% weight taken proportionally from core criteria. A focus criterion scoring 8+ for 3 consecutive runs rotates out.>

## Scoring Guide

| Range | Meaning |
|-------|---------|
| 9-10 | Exceptional — could be shown to external stakeholders unedited |
| 7-8 | Good — meets the bar, minor improvements possible |
| 5-6 | Below bar — specific issues need fixing before this is reliable |
| 3-4 | Poor — fundamental gaps in the output |
| 0-2 | Broken — output is wrong, misleading, or missing |

Pass bar: 7.0/10
```

### Rubric Design Rules

- **Weight the differentiator highest.** What makes this skill's output good or bad? That criterion gets 15-20%.
- **Data accuracy gets 15-20% if the skill uses external data.** Wrong data cascades into wrong conclusions.
- **Communication quality gets 5%.** It matters but it's not the differentiator.
- **Include at least 2 VERIFY steps** in the criteria — things the judge must mechanically check (count sections, run a script, compare numbers).
- **Hard caps prevent good scores on broken outputs.** If one thing being wrong makes the whole output dangerous, cap it.

---

## Step 3 — Build the Judge Skill

Create: `<skills-dir>/judge-score-<skill-name>/SKILL.md`

### Judge Skill Template

```markdown
---
name: judge-<skillslug>
description: >
  Score a <skill-name> output for quality, update calibration notes.
  Triggered after each <skill-name> delivery, or manual via $judge-<skillslug>.
  NOT for: scoring other skills (separate judge skills).
---

# Judge — Score <Skill Name> Output

Three-part workflow: score the output, write the JSON, update calibration.

Read the rubric (`references/rubric.md`, loaded with this skill) before scoring.

## Part 1: Find and Read the Output

1. Check the trigger message for the output file path.

2. Read that file. If the path from the trigger doesn't exist, find the most recent output at the skill's configured output path.

3. **If no output file exists, stop. Do not score.**

## Part 2: Score Quality

### Pre-scoring: Check for approved focus criteria proposals

Before scoring, read the calibration file. If it contains a `## Pending Focus Criteria Proposal` section with `Status: approved`, apply it now (see Part 4, Step 5). Then proceed with scoring using the updated rubric.

Also check the rubric for any **active focus criteria**. If present, score them alongside the core criteria.

### Mandatory Verification Steps (run BEFORE scoring)

Run every verification step listed in the rubric. Record results. Then score.

### Score each criterion

For each criterion in the rubric:
1. State what you checked
2. State what you found (with specific evidence from the output)
3. Assign a score 0-10

### Calculate overall score

Weighted average of all criteria. Apply any hard caps. Round to 1 decimal.

### Verdict

- **PASS** (>= pass bar): Note strengths and any minor improvements
- **FAIL** (< pass bar): List the top 3 issues that must be fixed, in priority order

## Part 3: Write Score JSON + Update Calibration

1. Write score JSON to your configured scores directory:

Format:
\`\`\`json
{
  "skill": "<skill-name>",
  "date": "YYYY-MM-DD",
  "overall": 0.0,
  "verdict": "PASS|FAIL",
  "criteria": {
    "<criterion_name>": { "score": 0, "weight": 0, "note": "<evidence>" }
  },
  "top_issues": ["<issue1>", "<issue2>"],
  "focus_criteria_scores": {}
}
\`\`\`

2. Read the calibration file.

3. If the score is below pass bar, or a criterion scored below 5: add a dated entry to `## Recent Quality Issues` describing what went wrong and what should change.

4. If a pattern appears in 3+ recent scores: add or update a rule in `## Distilled Rules`.

## Part 4: Focus Criteria Review

1. Read the last 5 score JSONs for this skill.
2. If a pattern of weakness appears that isn't covered by core criteria: draft a focus criterion proposal.
3. Write the proposal to the calibration file under `## Pending Focus Criteria Proposal` with `Status: proposed`.
4. The proposal needs user approval before it takes effect.
5. When applying an approved proposal: add it to the rubric's Focus Criteria section, increment the version, remove the pending proposal from calibration.

Reply with no user-facing output. The judge runs silently — scoring results are persisted to files, not displayed inline.
```

**Adapt the template:** Replace all `<placeholders>` with actual values. Adjust Part 1 to match how the skill saves its output. Add skill-specific verification steps from the rubric into Part 2.

---

## Step 4 — Build the Calibration File

Create the calibration file at your configured feedback/calibration directory.

```markdown
# <Skill Name> — Calibration Notes

## Distilled Rules

_No distilled rules yet. Rules are added when patterns appear across 3+ scoring runs._

## Recent Quality Issues

_No quality issues recorded yet._

## Accuracy Lessons

_No accuracy lessons yet._
```

---

## Step 5 — Wire Everything

### 5a. Register the skill and judge

Add entries for BOTH the skill (if new) and the judge skill to your skill registry or configuration. Each entry should include:
- Trigger (how it's invoked)
- Description (what it does)
- Skill file path
- Data sources (rubric, calibration, score JSONs)

### 5b. Skill SKILL.md — Add self-review step (if not present)

If the skill's own SKILL.md doesn't already reference the judge or calibration file, add a step near the end:
- Read calibration file before running (so past lessons inform the current run)
- Save output to archive path (so the judge can find it)

---

## Step 6 — Test Invocation

Run the skill in draft-only / test mode:

1. Invoke the skill (use `draft-only` or `do not send` if it has a delivery step)
2. Read the output
3. Score it yourself against the rubric you just built — assign scores to every criterion
4. Calculate the overall score

### If score >= 7.0 (PASS):
- Skill is live with full quality loop
- Report: "Skillforge complete for <skill-name>. Score: <score>/10. All pieces wired."

### If score < 7.0 (FAIL):
- Identify the top 3 issues dragging the score down
- Fix the SKILL.md to address them (one round of edits)
- Re-run the test
- If still < 7.0 after one fix round: report the score and issues to the user. Don't loop forever.

---

## Step 7 — Report

Summarise what was built:

```
Skillforge: <skill-name> (<mode>)

Built:
- [ ] Rubric (<N> criteria, pass bar <X>)
- [ ] Judge skill (judge-<skillslug>)
- [ ] Calibration file
- [ ] Registry entries (skill + judge)
- [ ] Skill SKILL.md updated (reads calibration, saves to archive)
- [ ] Test invocation: <score>/10 (<PASS|FAIL>)

Top issues (if any):
1. ...
```
