# Skillforger

An AI skill creator that generates complete skills with built-in quality loops. Instead of just writing a skill file, Skillforger produces the full closed-loop system: the skill itself, a scoring rubric with evolving focus criteria, a judge skill, and calibration tracking that feeds back into every future run.

## What it does

Skillforger has three modes. The `$skillforge` syntax below is the Claude Code slash-command form; Codex and other runtimes without slash-command support invoke the skill via a tiny launcher instead — see [Invocation modes](#invocation-modes) below.

### CREATE — build a new skill from scratch
```
$skillforge create <description>
```
Generates everything: SKILL.md, rubric, judge skill, calibration file, and wiring.

### FORGE — wrap an existing skill with quality loops
```
$skillforge forge <skill-name>
```
Audits what exists, builds only the missing pieces (rubric, judge, calibration, wiring). Doesn't overwrite working parts.

### REBUILD — fix a skill that's underperforming
```
$skillforge rebuild <skill-name>
```
Re-reads the latest output, scores it, identifies quality gaps, and rewrites the SKILL.md to fix them.

## The closed-loop pattern

```
Skill runs -> Output saved -> Judge scores against rubric -> Calibration updated
                                                                    |
                                                                    v
                                            Next skill run reads calibration
                                            and adjusts based on past feedback
```

Every skill Skillforger creates follows this pattern. The quality loop is not optional — it's the whole point.

## Key concepts

### Evolving rubrics with focus criteria

Rubrics aren't static. The judge identifies recurring weaknesses across scoring runs and proposes **focus criteria** — temporary scoring dimensions that get added to the rubric with 5% weight taken proportionally from core criteria. Max 2 active at a time. When a focus criterion scores 8+ for 3 consecutive runs, it rotates out. This makes the rubric adapt to the skill's actual failure patterns.

### Distilled rules

When the same quality issue appears across 3+ scoring runs, the judge promotes it to a **distilled rule** in the calibration file. Distilled rules are permanent lessons — they persist even after the specific issue is fixed, so the skill never regresses.

### Blind judging

The judge scores output against the rubric with mandatory verification steps (mechanical checks run before any subjective scoring). Hard caps prevent good scores on fundamentally broken output — if one critical thing is wrong, the overall score is capped at 4.0 regardless of other criteria.

### The 7.0 bar

Pass bar is 7.0/10. Skillforger never lowers the bar to make tests pass — it fixes the skill instead. One round of fixes is allowed; if still failing, it reports and stops rather than looping forever.

## What gets generated

For each skill, Skillforger creates:

| File | Purpose |
|------|---------|
| `SKILL.md` | The skill itself, with calibration-reading and output-saving built in |
| `judge-score-<name>/SKILL.md` | Judge skill — 4-part workflow: find output, score, update calibration, review focus criteria |
| `judge-score-<name>/references/rubric.md` | Weighted scoring rubric with hard caps, verification steps, and evolving focus criteria |
| Calibration file | Distilled rules, recent quality issues, and accuracy lessons — read by the skill before every run |
| Registry entries | Wiring so both the skill and its judge are discoverable |

## Invocation modes

The skill body is identical in every runtime. Only the invocation shape changes.

| Runtime | Invocation | Why |
|---|---|---|
| **Claude Code** | `$skillforge create ...` (slash command) | Native slash-command support |
| **Codex / OpenClaw** | `python3 path/to/skillforge.py create ...` (launcher) | Codex can't resolve slash commands; a thin launcher reads the SKILL.md and pipes it to the model |
| **Other frameworks** | Read `SKILL.md` into context, pass `create|forge|rebuild <argument>` | Anything that can load a markdown file and follow instructions |

The launcher is trivial (~70 lines): read the canonical SKILL.md, print it with a `mode: <create|forge|rebuild>` and `description:` / `skill-name:` header, and let the model follow it. The real work lives in the SKILL.md itself, which is framework-agnostic.

## How to use it

### As a Claude Code skill

```bash
# Copy to your skills directory
cp -r skillforger/ ~/.claude/skills/skillforger/

# Create a slash command
mkdir -p ~/.claude/commands
cat > ~/.claude/commands/skillforge.md << 'EOF'
Read and execute the skill at `~/.claude/skills/skillforger/SKILL.md` in full.

Apply the skill to the user's request: $ARGUMENTS
EOF
```

Then:
```
/skillforge create a skill that summarizes pull requests
/skillforge forge my-existing-skill
/skillforge rebuild underperforming-skill
```

### As a reference for other agent frameworks

The core ideas are framework-agnostic:

- **Rubric-first design**: Define what "good" looks like with weighted criteria before building the skill
- **Evolving focus criteria**: Let the judge adapt the rubric based on actual failure patterns
- **Distilled rules**: Promote recurring issues to permanent calibration rules
- **Calibration feedback loop**: Every run reads past scoring feedback, so the skill improves over time
- **Mandatory verification**: Mechanical checks before subjective scoring prevents inflated scores

## Customization

Skillforger is opinionated but adaptable:

- **Paths**: Edit Step 3.1 and the judge template to match your project structure
- **Wiring**: Step 5 is generic — adapt it to your skill registry format
- **Scoring scale**: Default is 0-10 with 7.0 pass bar. Adjust in the rubric template if needed
- **Focus criteria rules**: Change the max active count (default 2) or rotation threshold (default 3 consecutive runs at 8+)
- **Notification**: Add a notification step to the judge if you want scoring alerts
