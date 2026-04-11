---
name: verify-prompt-names
description: >-
  Verify Claude-generated prompt names, IDs, descriptions, and variable names
  in extracted prompt JSON files. Catches wrong category prefixes, duplicate
  IDs, fabricated categories, inconsistent variable names, and inaccurate
  descriptions by cross-referencing against actual repo conventions.
  Use after running `tools/nameWithClaude.js` to name or rename prompts,
  when reviewing newly named prompts before committing, or when validation
  errors mention A1–A3, A5, A15, A16, A20, or A23 rules.
  Trigger phrases: "verify names", "check names", "review naming",
  "verify prompt names", "are these names right", "validate naming".
---

# Verify Prompt Names

You verify that Claude-generated prompt metadata (names, IDs, descriptions, variable names) is correct by cross-referencing against the actual conventions in the repo. The naming tool (`tools/nameWithClaude.js`) uses Claude to propose these values, and Claude is prone to specific, predictable errors.

## Repository Layout

Two sibling repos are involved. Know where each thing lives:

| What | Where | Contains |
|------|-------|----------|
| **CCSP** (this repo) | `/code/claude-code-system-prompts/` (cwd) | `system-prompts/*.md` (naming conventions), `tools/nameWithClaude.js` |
| **TweakCC** | `/code/tweakcc/` | `data/prompts/prompts-*.json` (all versions), `prompts.json` (working copy) |

The JSON files with prompt metadata live in **TweakCC**, not here. When running `nameWithClaude.js`, pass JSON paths into TweakCC (e.g., `node tools/nameWithClaude.js /code/tweakcc/data/prompts/prompts-2.1.XX.json "Prompt Name"`). The `system-prompts/*.md` files you cross-reference for naming conventions live here in **CCSP**.

## Why This Exists

`nameWithClaude.js` has a stale `METADATA_SYSTEM_PROMPT` that gives Claude wrong examples. Common failures:

- **Wrong category prefix**: The tool says `"Agent:"` but every file uses `"Agent Prompt:"`. The tool omits `Data:`, `System Reminder:`, and `Tool Parameter:` from its category table, so Claude invents fake categories like `"Memory Description:"`.
- **Inconsistent variable names**: Claude names a variable `MEMORY_CONTEXT` when the established convention (in a sibling prompt) is `MEMORY_DIR_CONTEXT`.
- **Duplicate IDs**: Claude picks a name that's already used by another prompt in the same file.
- **Wrong variable suffix**: Claude names something `*_TOOL_NAME` when it's accessed with `.name` (should be `*_TOOL` or `*_TOOL_OBJECT`).

## Verification Checklist

For each prompt that was named or had variables named, check ALL of the following:

### 1. Category prefix is one of the 7 valid categories

Read the `name` field. The prefix before the colon MUST be one of:

| Valid Prefix | Used For |
|---|---|
| `Agent Prompt:` | Subagent system prompts |
| `System Prompt:` | Core system behavior, modes, instructions |
| `System Reminder:` | Injected reminders and status messages |
| `Tool Description:` | Tool definitions and usage notes |
| `Tool Parameter:` | Tool parameter descriptions |
| `Data:` | Reference data, templates, API docs |
| `Skill:` | User-invocable skill instruction blocks |

**How to determine the right category:** Read the prompt content.

- Starts with "You are..." and gives an agent a task → `Agent Prompt:`
- Injected into the system prompt as behavioral guidance → `System Prompt:`
- A `<description>` tag describing a memory type → `System Prompt:` (NOT "Memory Description:")
- Tool usage examples injected into the system prompt → `System Prompt:` (NOT "Skill:" unless it's user-invocable)
- Defines a tool's name, description, parameters → `Tool Description:`
- A template, reference doc, or API data → `Data:`

### 2. ID is the correct slug of the name AND is unique

- The `id` must be derived from the `name`: lowercase, spaces/special chars → hyphens, strip the category prefix and re-add its slug form.
- Search the ENTIRE prompts array for duplicate IDs. The file has ~250 prompts — a name that seems unique in isolation may collide with an existing prompt.

**To check:** Run a quick scan:
```js
// Find duplicates
const ids = {};
data.prompts.forEach((p, i) => {
  if (ids[p.id]) console.log(`DUPE: "${p.id}" at ${ids[p.id]} and ${i}`);
  ids[p.id] = i;
});
```

If a collision exists, read both prompts' content and differentiate the new one's name. Common strategies:
- Add a qualifier: `"(compact)"`, `"(with explicit save)"`, `"(basic)"`
- Use a more specific detail: `"Subagent prompt-writing examples"` vs `"Subagent delegation examples"`

### 3. Variable names match existing conventions

For each variable in `identifierMap`, search `system-prompts/*.md` for prompts with similar content. Check their `variables:` frontmatter. If the same conceptual variable already has an established name, the new prompt MUST use that same name.

**Common convention mismatches to watch for:**

| Wrong | Right | Why |
|---|---|---|
| `DIRECTIVE` | `WORKER_DIRECTIVE` | Be specific — which directive? |
| `MEMORY_CONTEXT` | `MEMORY_DIR_CONTEXT` | Existing dream prompts use `MEMORY_DIR_CONTEXT` |
| `TOOL_OBJECT` (used as string) | `AGENT_TOOL_NAME` | No property access → `_TOOL_NAME` not `_TOOL_OBJECT` |

### 4. Variable suffixes match usage patterns

Read the template pieces around each variable to determine HOW it's used:

| If you see... | Suffix must be | Example |
|---|---|---|
| `${VAR}(args)` or `${VAR}()` | `*_FN` | `IS_TRUTHY_FN`, `POST_GATHER_FN` |
| `${VAR}.name` or `${VAR}.description` | `*_TOOL` or `*_TOOL_OBJECT` | `WRITE_TOOL`, `TASK_TOOL_OBJECT` |
| `${VAR}` used as plain text in prose | `*_TOOL_NAME` or `*_NOTE` | `BASH_TOOL_NAME`, `GIT_NOTE` |
| `${VAR}.env` | `PROCESS_OBJECT` | Node.js process |
| `${VAR}` in a ternary `${VAR}?...:...` | `HAS_*` or `IS_*` | `HAS_SUBAGENT_TYPES` |
| `${VAR}` as a multi-line content block | `*_BLOCK` or `*_CONFIG` | `AGENT_TYPES_BLOCK` |
| `${VAR}` as a count/number | `*_COUNT`, `*_LIMIT`, `MAX_*` | `INDEX_MAX_LINES` |

### 5. Description accurately reflects content

Read the FULL prompt content (not just the preview). The description should:
- Be one sentence
- Start with a verb or noun phrase
- Not end with a period
- Accurately describe what the prompt does — not what it sounds like from the first line

## How to Read Variable Usage Context

Reconstruct the template to see how each variable is used:

```
pieces[0] + ${identifierMap[identifiers[0]]} + pieces[1] + ${identifierMap[identifiers[1]]} + pieces[2] ...
```

For each NEW or CHANGED variable, read ~100 characters before and after to understand:
- Is it called as a function? (`()` immediately follows)
- Is a property accessed? (`.something` follows)
- Is it interpolated as a string in prose?
- Is it used in a conditional/ternary?

## Reporting

For each prompt checked, report one of:

- ✅ **Clean** — all fields correct
- ⚠️ **Uncertain** — a variable name is plausible but can't be confirmed without runtime data (e.g., the `_BLOCK` suffix is right but the prefix is a guess)
- ❌ **Error** — wrong category, duplicate ID, wrong variable suffix, or inconsistent with existing convention

For errors, state:
1. Which field is wrong
2. What it currently says
3. What it should be
4. Why (cite the existing file or convention that proves it)

## Quick Reference: Existing Variable Names

These variables appear across multiple prompts. If a new prompt uses the same concept, it MUST use the same name:

| Variable | Used in | Purpose |
|---|---|---|
| `MEMORY_DIR` | Dream prompts | Path to memory directory |
| `MEMORY_DIR_CONTEXT` | Dream prompts | Context text about the memory directory |
| `ADDITIONAL_CONTEXT` | Multiple agent prompts | Optional extra context appended at end |
| `AGENT_TOOL_NAME` | Delegation examples, tool descriptions | Plain string name of the Agent/Task tool |
| `SEND_MESSAGE_TOOL_NAME` | Tool descriptions | Plain string name of the SendMessage tool |
| `IS_TRUTHY_FN` | Tool descriptions | Function that checks truthiness |
| `PROCESS_OBJECT` | Tool descriptions | Node.js `process` object |
| `HAS_SUBAGENT_TYPES` | Tool descriptions | Boolean: are subagent types available? |
| `IS_SUBAGENT_CONTEXT_FN` | Tool descriptions | Function: are we in subagent context? |
| `IS_TEAMMATE_CONTEXT_FN` | Tool descriptions | Function: are we in teammate context? |
| `SUBAGENT_TYPE_DEFINITIONS` | Tool descriptions | Conditional type definitions block |
| `INDEX_FILE` | Dream consolidation | Path to memory index file |
| `INDEX_MAX_LINES` | Dream consolidation | Max line count for index |
| `TRANSCRIPTS_DIR` | Dream consolidation | Path to session transcripts |

This list is not exhaustive. Always search `system-prompts/*.md` for the authoritative set.
