# Semantic Variable Analysis (A23)

Evaluate whether template variable names make semantic sense in their usage context. This catches the single most frequent and damaging class of extraction bug — names that are structurally valid but semantically wrong for where they appear.

Structural rules can catch duplicates, empty strings, and format issues. They CANNOT catch:
- `WRITE_TOOL_NAME.agentType` (tool names are strings, not objects)
- `EXPLORE_SUBAGENT_NOTE.name` in text about "create your plan"
- `SEARCH_TOOL_NAME` where `GLOB_TOOL_NAME` was meant

You can.

## Procedure

For each prompt that has template variables:

1. Reconstruct the context around each variable: ~100 characters before and after the interpolation point
2. Evaluate whether the variable name fits the surrounding context
3. Check `tools/semantic-rules.json` — skip any variable+prompt combination that has a suppression rule
4. Report findings as warnings (you may produce false positives)

### Diff Mode (two JSON files)

When given both an old and new JSON, only analyze prompts that actually changed:

1. Match prompts between old and new by `id`
2. For each prompt in the new file, compare `pieces`, `identifiers`, and `identifierMap` against the old version (use JSON.stringify for each field)
3. If all three are identical → **skip** (unchanged)
4. If any differ, or if the prompt is new (no matching `id` in old) → **analyze**
5. If a prompt has no `id`, treat it as changed (can't match)
6. Report the breakdown: N modified, N new, N unchanged (skipped)

## Variable Naming Conventions

Claude Code uses these naming patterns:

| Pattern | Type | Example |
|---------|------|---------|
| `*_TOOL_NAME` | Plain string — a tool's display name | `"Write"`, `"Edit"`, `"Bash"` |
| `*_TOOL` or `*_TOOL_OBJECT` | Tool object with properties | Has `.name`, `.description` |
| `*_FN` | Function reference | `IS_TRUTHY_FN`, `JSON_STRINGIFY_FN` |
| `*_AGENT` or `*_SUBAGENT` | Agent type/config object | Has `.agentType`, `.color` |
| `*_NOTE` | Text note/string | Inline guidance text |
| `PROCESS_OBJECT` / `PROCESS_ENV` | Node.js process | `process` object or `process.env` |
| `SYSTEM_REMINDER` | System state object | Has `.planFilePath`, `.planExists` |

## What to Look For

### 1. Property access mismatch

A `*_TOOL_NAME` variable (which is a string) accessed with `.name`, `.agentType`, or other properties. Strings don't have those properties.

**Example**: `${WRITE_TOOL_NAME.name}` — should be `${WRITE_TOOL.name}` or `${WRITE_TOOL_OBJECT.name}`

### 2. Semantic context mismatch

A variable whose name implies one concept used in text describing a different concept.

**Example**: `${EXPLORE_SUBAGENT_NOTE.name}` in text about "create your plan" — the text describes writing/planning, not exploring.

**Example**: `${SEARCH_TOOL_NAME}` in text about file globbing/pattern matching — should be `${GLOB_TOOL_NAME}`

### 3. Wrong function assignment

A function variable used where its name doesn't match the operation described.

**Example**: `${BASH_TOOL(...)}` where the text describes checking if a value is truthy — should be `${IS_TRUTHY_FN(...)}`

**Example**: `${IS_BACKGROUND_TASKS_DISABLED_FN(...)}` where the text checks for teammate context — should be `${IS_IN_TEAMMATE_CONTEXT_FN(...)}`

### 4. Object/string confusion

Using a `*_NAME` variable (string) where an object is needed, or vice versa.

**Example**: `${TASK_TOOL_NAME}` used with `.description` access — should be `${TASK_TOOL_OBJECT}`

### 5. Agent type confusion

Using an explore-related variable in a plan context or vice versa.

**Example**: `${PLAN_SUBAGENT}` in a section describing exploration agents

## What NOT to Flag

- Variables used correctly in context (most will be correct)
- Variables too generic to evaluate — `SYSTEM_REMINDER` is a catch-all state container; its property accesses are valid
- Same variable appearing multiple times with the same access pattern — that's normal
- `UNRESOLVED_KEY_*` variables — these are already caught by structural rule A7
- Minor naming preferences that don't indicate an actual mistake

## Suppression Rules

Before reporting any finding, check `tools/semantic-rules.json`. Each entry specifies:

```json
{
  "prompt": "Tool Description: Task",
  "variable": "GLOB_TOOL",
  "position": 5,
  "verdict": "correct",
  "reason": "Confirmed against source — code does pass GLOB_TOOL here."
}
```

If a finding matches on `prompt` + `variable` (and optionally `position`), skip it.

### Interactive Review

When discussing findings with the user:
- If they confirm a finding is a **false positive**, add a suppression rule to `tools/semantic-rules.json` with their explanation
- If they confirm a finding is **real**, note it as an error to fix in the JSON
- Ask them to check against the Claude Code source when uncertain
