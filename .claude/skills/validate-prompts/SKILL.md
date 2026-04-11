---
name: validate-prompts
description: >-
  Validate extracted Claude Code prompt data by reading files and checking
  rules directly — no external scripts or API calls needed.
  Checks JSON structure (30+ rules), generated markdown files, README
  consistency, and semantic variable name correctness.
  Use whenever asked to validate prompt JSON files, check generated output,
  run pre-release checks, debug validation errors, or analyze variable naming.
  Trigger phrases: "validate", "check prompts", "run validation",
  "verify prompts", "structural checks", "semantic check", "release prep".
  Also use when investigating a specific validation rule (A1–A21, B1–B6,
  C1–C7, A23) or when encountering errors in prompt data.
---

# Prompt Validation

You validate the prompt pipeline by reading files and checking rules directly. No external scripts or API calls are needed — you are the validator.

## Repository Layout

Two sibling repos are involved. Know where each thing lives:

| What | Where | Contains |
|------|-------|----------|
| **CCSP** (this repo) | `/code/claude-code-system-prompts/` (cwd) | `system-prompts/*.md`, `README.md`, `CHANGELOG.md`, `tools/` scripts |
| **TweakCC** | `/code/tweakcc/` | `data/prompts/prompts-*.json` (all versions), `prompts.json` (working copy) |

The JSON files you validate live in **TweakCC**, not here. When running tools or reading JSON, use paths like `/code/tweakcc/data/prompts/prompts-2.1.XX.json`. The generated output you check (markdown files, README) lives here in **CCSP**.

## Validation Parts

There are four validation parts. Load the relevant reference files for detailed rules:

| Part | Scope | Rules | Reference |
|------|-------|-------|-----------|
| A | JSON structure | A1–A21 | [structural-rules.md](references/structural-rules.md) |
| B | Generated markdown | B1–B6 | [output-rules.md](references/output-rules.md) |
| C | README | C1–C7 | [output-rules.md](references/output-rules.md) |
| D | Semantic variables | A23 | [semantic-analysis.md](references/semantic-analysis.md) |

Reference data (tool names, metadata fields, prefixes) is in [known-values.md](references/known-values.md).

## Approach

### When the user provides a JSON file

1. Read the JSON file. It has structure: `{ "version": "2.1.XX", "prompts": [...] }`
2. Load [structural-rules.md](references/structural-rules.md) and [known-values.md](references/known-values.md)
3. Check every prompt against rules A1–A21
4. Report errors (❌) and warnings (⚠️) with rule codes

### When asked to also check generated output

5. Read the `.md` files in `system-prompts/` and `README.md`
6. Load [output-rules.md](references/output-rules.md)
7. Check rules B1–B6 (markdown files) and C1–C7 (README)

### When asked for semantic validation

8. Load [semantic-analysis.md](references/semantic-analysis.md)
9. For each prompt with template variables, examine whether variable names make sense in their usage context
10. Check `tools/semantic-rules.json` for known false positives — skip those
11. Report findings as warnings (you may have false positives — discuss with the user)

### When asked for semantic diff validation (two JSON files)

This is the most common real-world case: a new version was extracted and you only need to check what changed.

8. Read BOTH JSON files from TweakCC (old baseline + new version)
9. Match prompts between old and new by `id`. Classify each prompt:
   - **Added** — exists in new but not old → analyze it
   - **Changed** — exists in both but `pieces`, `identifiers`, or `identifierMap` differ → analyze it
   - **Unchanged** — identical in both → **skip it**
10. Load [semantic-analysis.md](references/semantic-analysis.md)
11. Run semantic analysis ONLY on the added + changed prompts
12. Check `tools/semantic-rules.json` for known false positives — skip those
13. Report: how many changed, how many new, how many skipped, then any findings

### When asked to review semantic findings interactively

If the user confirms a finding is a false positive, add a suppression rule to `tools/semantic-rules.json`:

```json
{
  "prompt": "Exact Prompt Name",
  "variable": "VARIABLE_NAME",
  "position": 5,
  "verdict": "correct",
  "reason": "Explanation of why this is correct"
}
```

## Understanding the JSON Structure

Each prompt object:

```json
{
  "name": "System Prompt: Example",
  "id": "system-prompt-example",
  "description": "What this prompt does.",
  "pieces": ["static text before var", " text between vars ", " text after last var"],
  "identifiers": [0, 1],
  "identifierMap": { "0": "FIRST_VARIABLE", "1": "SECOND_VARIABLE" },
  "agentMetadata": { ... }
}
```

The template model: `pieces[0] + ${identifierMap[identifiers[0]]} + pieces[1] + ${identifierMap[identifiers[1]]} + pieces[2]`

So `pieces.length` should always equal `identifiers.length + 1`.

## Reporting

- **Errors (❌)** — must be fixed. Structural inconsistencies that will cause downstream problems.
- **Warnings (⚠️)** — informational. May be benign (upstream quirks, semantic false positives).

Report each finding with its rule code (e.g., "A5", "B2", "C3") and the prompt name. For errors, include enough context to fix the issue.

## Severity Quick Reference

| Rule | Severity | What it catches |
|------|----------|----------------|
| A1–A3 | Error | Missing/empty/duplicate name or id |
| A4 | Warning | Missing description |
| A5–A6 | Error | Empty or duplicate identifierMap values |
| A7 | Error/Warning | identifiers ↔ identifierMap sync |
| A8–A10 | Error | Sequential keys, pieces count, paired arrays |
| A11–A12 | Error | Garbled tool names (minification artifacts) |
| A13–A14 | Warning | Null whenToUse, unknown metadata fields |
| A15–A16 | Error | Name format, filename slug collisions |
| A17–A18 | Error | Un-interpolated variable refs, self-referencing calls |
| A19 | Error | Double-period descriptions |
| A20 | Warning | id/name mismatch |
| A21 | Error | Unsorted prompts array |

| B1–B6 | Error/Warning | Markdown frontmatter and content issues |
| C1–C7 | Error/Warning | README consistency issues |
| A23 | Warning | Semantic variable name correctness |

## Alternate Scripts

The same validation logic exists as standalone scripts in `tools/` (in CCSP) for CI/CD or batch use. Run from the CCSP root, passing JSON paths into TweakCC:

- `node tools/validateEverything.js /code/tweakcc/data/prompts/prompts-2.1.XX.json` — structural rules A1–A21
- `node tools/validateEverything.js /code/tweakcc/data/prompts/prompts-2.1.XX.json --also-output` — adds B1–B6, C1–C7
- `node tools/validateEverything.js /code/tweakcc/data/prompts/prompts-2.1.XX.json --semantic` — A23, all prompts (requires Agent SDK + Claude Max)
- `node tools/validateEverything.js /code/tweakcc/data/prompts/prompts-2.1.NEW.json --semantic-diff /code/tweakcc/data/prompts/prompts-2.1.OLD.json` — A23, only changed prompts
- `node tools/verifyChangelog.js /code/tweakcc/data/prompts/prompts-2.1.OLD.json /code/tweakcc/data/prompts/prompts-2.1.NEW.json changelog.md` — changelog verification (requires Agent SDK)

These scripts require the Agent SDK and Claude Max auth for the Claude-powered checks. This skill lets you do everything without those dependencies.
