# AGENTS.md

This file provides guidance to AI coding agents (Claude Code, Cursor, Copilot, etc.) when working with code in this repository.

## Overview

This repository contains **Claude Skills** — custom extensions that enhance Claude's capabilities in Claude Code and Claude.ai.

Skills are Markdown files with YAML frontmatter that instruct Claude on how to handle specific tasks. They can optionally bundle Python scripts, reference documents, and assets.

## Repository Structure

```
skills/                        # Released/packaged skills
  └── stage/
      └── SKILL.md            # A released skill for code review/staging

template/                      # Template for new skills
  └── SKILL.md

skills-lock.json              # Lock file tracking installed skills
```

## Skill Format and Architecture

A skill is fundamentally a **SKILL.md** file with this structure:

```yaml
---
name: skill-identifier
description: "When to use this skill and what it does. Include specific contexts where Claude should trigger it."
---

# Skill Name

[Markdown instructions for how Claude should use this skill]
```

**Key design patterns:**

1. **Metadata matters** — The `name` and `description` fields are the primary triggering mechanism. Descriptions should be explicit about when Claude should use the skill, with examples of user phrases that should trigger it.

2. **Progressive disclosure** — Keep SKILL.md under 500 lines. Reference larger documentation through `references/` directory (loaded on-demand). Scripts can be in `scripts/` and invoked without loading the file.

3. **Bundled resources:**
   - `scripts/` — Python helpers for deterministic tasks (evaluation, benchmarking, code generation)
   - `references/` — Documentation loaded into context when needed
   - `assets/` — Templates, icons, or data files used in output

## Common Development Tasks

### Creating a New Skill

1. Copy `template/SKILL.md` or create a new directory under `.claude/skills/`
2. Define YAML frontmatter: `name`, `description` (required), optionally `compatibility` or `metadata`
3. Write Markdown instructions explaining how Claude should accomplish the task
4. If needed, add test cases in `evals/evals.json`:
   ```json
   {
     "skill_name": "my-skill",
     "evals": [
       {
         "id": 1,
         "prompt": "User's task description",
         "expected_output": "What success looks like",
         "files": []
       }
     ]
   }
   ```

### Testing and Evaluating Skills

The skill-creator includes a full testing pipeline. From `.claude/skills/skill-creator/`:

**Run test cases with evaluation:**
```bash
# Spawn parallel test runs (with-skill and baseline)
# Results go to: <skill>-workspace/iteration-1/
# Then generate the viewer:
python eval-viewer/generate_review.py <workspace>/iteration-1 \
  --skill-name "my-skill" \
  --benchmark <workspace>/iteration-1/benchmark.json
```

**Aggregate benchmark results:**
```bash
# After all runs complete and are graded:
python -m scripts.aggregate_benchmark <workspace>/iteration-1 --skill-name skill-name
```

**Optimize skill triggering description:**
```bash
# After the skill is complete, run description optimization:
python -m scripts.run_loop \
  --eval-set trigger-evals.json \
  --skill-path .claude/skills/my-skill \
  --model claude-opus-4-6 \
  --max-iterations 5 \
  --verbose
```

### Packaging a Skill

To package a skill for distribution:
```bash
python -m scripts.package_skill .claude/skills/my-skill
# Outputs: my-skill.skill (a bundled archive)
```

### Validating a Skill

Quick validation before running full evals:
```bash
python -m scripts.quick_validate.py .claude/skills/my-skill
```

## Skill Evaluation Workflow

The complete iteration workflow (documented in detail in `.claude/skills/skill-creator/SKILL.md`):

1. **Draft** — Write the SKILL.md and define test cases
2. **Test** — Run test cases with and without the skill in parallel
3. **Review** — Launch the eval viewer; user provides feedback on outputs and benchmark metrics
4. **Improve** — Revise the skill based on feedback
5. **Iterate** — Repeat steps 2-4 until satisfied
6. **Optimize** — Refine the description for better triggering
7. **Package** — Prepare final .skill file for distribution

Test results are organized by iteration:
```
skill-workspace/
  ├── iteration-1/
  │   ├── eval-0/
  │   │   ├── with_skill/outputs/
  │   │   ├── without_skill/outputs/
  │   │   └── eval_metadata.json
  │   ├── benchmark.json
  │   └── feedback.json
  └── iteration-2/
      └── [same structure]
```

## Key Files and Scripts

| File | Purpose |
|------|---------|
| `scripts/run_eval.py` | Execute a single evaluation with Claude |
| `scripts/run_loop.py` | Optimize skill triggering description (5 iterations) |
| `scripts/aggregate_benchmark.py` | Combine test run results into benchmark stats |
| `scripts/package_skill.py` | Bundle a skill into .skill file for distribution |
| `scripts/quick_validate.py` | Quick syntax/structure check |
| `scripts/improve_description.py` | Single-iteration description improvement |
| `scripts/generate_report.py` | Generate markdown report from benchmark data |
| `agents/grader.md` | How to grade assertions against outputs |
| `agents/analyzer.md` | How to analyze benchmark results for patterns |
| `agents/comparator.md` | How to do blind A/B comparison between versions |
| `references/schemas.md` | JSON schemas for evals.json, grading.json, benchmark.json |

## Important Patterns

### Writing Effective Skill Descriptions

The description is the primary triggering mechanism. Make it explicit:

**Bad:** `"Create dashboards"`

**Good:** `"Build fast, interactive dashboards to visualize company data. Use this whenever the user mentions dashboards, data visualization, internal metrics, analytics, or wants to display any kind of company data — even if they don't explicitly ask for a dashboard."`

Include specific trigger phrases and contexts where Claude should consult the skill.

### Handling Edge Cases in Test Cases

When defining eval queries for description optimization, create realistic, substantive prompts — not trivial ones. Claude only triggers skills for multi-step tasks it would benefit from consulting about. Simple queries like "read file X" won't trigger skills regardless of description quality.

Good test queries include:
- Specific file names and contexts
- Column names, data details
- Company/domain context
- Multiple steps implied (not just "do X")
- Mix of formal and casual phrasing

### Bundling Repeated Code

If test runs show the model repeatedly inventing the same helper script across multiple evals (e.g., `create_docx.py`), that's a signal to bundle it in `scripts/` and instruct the skill to use it. This prevents every future invocation from reinventing the wheel.

## Notes on Python Scripts

All scripts in `.claude/skills/skill-creator/scripts/` are executable and can be imported as modules. They expect to run from the skill-creator directory or with proper Python path setup.

The scripts use a `utils.py` module for common functionality (file I/O, subprocess management, etc.).

## Related Documentation

- **In-depth workflow:** Read `.claude/skills/skill-creator/SKILL.md` for the complete skill creation and iteration guide
- **JSON schemas:** See `.claude/skills/skill-creator/references/schemas.md` for evals.json, benchmark.json, and grading.json structure
- **Subagent instructions:** See `agents/` directory for grader, comparator, and analyzer agent instructions

## Quick Reference: When to Use Which Script

| Task | Script |
|------|--------|
| Create test evals | Manually create `evals/evals.json` |
| Run a test case | Use subagent with skill path (not a script) |
| Grade assertions | Use subagent with `agents/grader.md` (not a script) |
| Review results in browser | `eval-viewer/generate_review.py` |
| Optimize description | `scripts/run_loop.py` (5 iterations, automated) |
| Single description iteration | `scripts/improve_description.py` |
| Bundle benchmark results | `scripts/aggregate_benchmark.py` |
| Package skill for release | `scripts/package_skill.py` |
| Quick sanity check | `scripts/quick_validate.py` |

## Running Commands in This Repo

Most work happens through:

1. **Skill creation** — Start with user intent, draft SKILL.md
2. **Python evaluation scripts** — From `.claude/skills/skill-creator/` directory
3. **Subagents** — For spawning test runs with and without the skill
4. **Browser viewer** — For human review of test results

There's no build step, no tests to run, no linting. The workflow is iterative: write → test → review → improve.
