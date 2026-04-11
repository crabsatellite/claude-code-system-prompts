---
name: verify-changelog
description: >-
  Verify changelog entries against actual prompt diffs by reading both JSON
  files and evaluating accuracy directly. Compares two prompt JSON versions
  (old → new), identifies added/removed/changed prompts, and checks that a
  human-written changelog accurately describes the changes.
  Use whenever writing, reviewing, or verifying a changelog entry for a new
  Claude Code version, when comparing prompt versions, when preparing a
  release, or when asked to "verify changelog", "check changelog",
  "changelog accuracy", or "diff vs changelog".
  Also use when asked whether a changelog is correct, complete, or well-worded,
  or when asked to help write a changelog for a version.
---

# Changelog Verification

You verify changelog accuracy by reading prompt JSON files directly, computing what changed, and evaluating changelog entries against those changes. No external scripts or API calls are needed — you do the analysis yourself.

## Repository Layout

Two sibling repos are involved. Know where each thing lives:

| What | Where | Contains |
|------|-------|----------|
| **CCSP** (this repo) | `/code/claude-code-system-prompts/` (cwd) | `CHANGELOG.md`, `changelog.md` (draft), `system-prompts/*.md`, `README.md` |
| **TweakCC** | `/code/tweakcc/` | `data/prompts/prompts-*.json` (all versions), `prompts.json` (working copy) |

The JSON files you compare live in **TweakCC**, not here. When reading prompt JSONs, use paths like `/code/tweakcc/data/prompts/prompts-2.1.XX.json`. The changelog you verify or write lives here in **CCSP**.

## Procedure

### 1. Read Both JSON Files

Read the old and new prompt JSON files from TweakCC (`/code/tweakcc/data/prompts/`). Each has this structure:

```json
{
  "version": "2.1.XX",
  "prompts": [
    {
      "name": "System Prompt: Example",
      "id": "system-prompt-example",
      "description": "What this prompt does.",
      "pieces": ["static text", " more text after variable ", " end"],
      "identifiers": [0, 1],
      "identifierMap": { "0": "VARIABLE_ONE", "1": "VARIABLE_TWO" }
    }
  ]
}
```

### 2. Compute the Diff

Match prompts between old and new files by `name` (preferred), falling back to `id`, falling back to a content-based match for unnamed prompts. Classify every prompt into one of:

- **Added** — exists in new but not old
- **Removed** — exists in old but not new
- **Changed** — exists in both but differs
- **Unchanged** — identical in both

For each **changed** prompt, note what specifically changed:

- Name or ID renamed
- Description wording changed
- Identifier count changed (variables added/removed)
- IdentifierMap values changed (variable names changed)
- Pieces text changed (the actual prompt content) — note the character count delta

To reconstruct a prompt's full text for comparison, interleave pieces with variable placeholders: `pieces[0] + ${VAR_0} + pieces[1] + ${VAR_1} + pieces[2] + ...`

### 3. Read the Changelog

Read the changelog file or text the user provides.

### 4. Evaluate Each Entry

For every changelog entry, assign one verdict:

| Verdict | Meaning |
|---------|---------|
| ✓ **accurate** | Entry correctly describes a real change |
| ✗ **inaccurate** | Entry describes something that didn't happen, or misdescribes what happened |
| ⚠ **misleading wording** | Technically correct but focuses on the wrong thing (see wording guidelines below) |
| ◐ **missing context** | Entry omits an important part of the change |

For non-accurate verdicts, explain why and suggest a rewrite.

### 5. Check for Gaps

- **Missing entries** — changes in the diff that have NO corresponding changelog entry. Suggest what the entry should say.
- **Fabricated entries** — changelog entries that don't correspond to any actual change.

### 6. Report

Summarize findings: count of each verdict type, list of issues, list of missing entries. Be specific — quote the changelog entry and the actual change it should describe.

## Wording Guidelines

Apply these when evaluating entries AND when helping write new ones:

### Describe behavior, not implementation

Good entries describe WHAT changed from the user's perspective:

| ❌ Bad | ✅ Good |
|--------|---------|
| Added `IS_BACKGROUND_TASKS_DISABLED_FN` variable with note about teammates | Added note that teammates cannot spawn other teammates |
| Renamed `SAFEUSER_VALUE` to `SAFE_USER_VALUE` | *(omit — pure variable rename, not user-visible)* |
| Replaced inline analysis instructions with `${ANALYSIS_INSTRUCTION_TAGS}` variable | Refactored analysis instructions into a shared variable *(or omit if output doesn't change)* |

### What to include

- New behavioral instructions ("added guidance on when to use plan mode")
- Removed capabilities or restrictions
- Changed tool descriptions that affect how Claude uses them
- New agent types or configurations
- Wording changes that alter Claude's behavior

### What to omit (unless user-visible)

- Pure variable renames (`FOO` → `BAR` with same content)
- Structural refactors that don't change output (extracting shared variables)
- Whitespace or formatting-only changes
- Variable name changes UNLESS they fix a user-visible typo (e.g., `CONDTIONAL` → `CONDITIONAL`)

### Changelog format

```markdown
# [2.1.XX](https://github.com/Piebald-AI/claude-code-system-prompts/commit/HASH)

_+/-NNN tokens_

- **NEW:** Prompt Category: Name — Description of what was added.
- **REMOVED:** Prompt Category: Name — Description of what was removed.
- Prompt Category: Name — Description of what changed.
```

- Use `**NEW:**` ONLY for entirely new prompt files (not new sections within existing prompts)
- Use `**REMOVED:**` for deleted prompt files
- No prefix for modified prompts — just describe the change

## Edge Cases

- If two prompts are renamed but otherwise identical, that's not worth a changelog entry (it's an internal reorganization).
- If a prompt's pieces change but only due to whitespace normalization, that's not a real change.
- If identifierMap values change but the prompt text output is identical (e.g., variable was renamed internally), that's not user-visible.
- If a prompt is split into two, treat as one removed + two added, and describe the behavioral motivation.
