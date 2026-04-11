# Structural Validation Rules (A1–A21)

Check every prompt in the JSON against these rules. Each rule exists because someone committed that exact bug. Git receipts trace each rule to its origin.

Load [known-values.md](known-values.md) for the reference lists (tool names, metadata fields, name prefixes).

---

## A1: Non-empty name

Every prompt must have a non-empty `name` field.

- **Severity**: Error
- **Receipt**: c2892a0 — prompts had empty names

## A2: Non-empty id

Every prompt must have a non-empty `id` field.

- **Severity**: Error
- **Receipt**: c2892a0

## A3: No duplicate ids

No two prompts may share the same `id`. Track all ids seen so far; if a duplicate appears, report both.

- **Severity**: Error

## A4: Non-empty description

Named prompts should have a non-empty `description`.

- **Severity**: Warning
- **Receipt**: c2892a0

## A5: identifierMap values must not be empty strings

Every value in `identifierMap` must be a non-empty string (UPPER_SNAKE_CASE variable name). Empty `""` values mean the extraction failed to resolve a name.

- **Severity**: Error
- **Receipt**: 3777ee5, c2892a0
- **Check**: For each key-value in `identifierMap`, verify `value !== ""`

## A6: No duplicate values in identifierMap

Within a single prompt, no two identifierMap keys should map to the same variable name. Duplicates mean the extraction assigned the same name to different variables.

- **Severity**: Error
- **Receipt**: a8756be, da78403, 565d98a, 347a18f
- **Check**: Collect all non-empty values; flag any that appear more than once, noting which keys share them

## A7: identifiers ↔ identifierMap sync

Bidirectional consistency:
- Every numeric value in `identifiers[]` must exist as a key in `identifierMap` → **Error** if missing
- Every key in `identifierMap` should be referenced by at least one entry in `identifiers[]` → **Warning** if orphaned

- **Receipt**: 2750dea, 80c4ca6
- **Check**: Build a set from `identifiers`, build a set from `Object.keys(identifierMap)` as numbers, compare both directions

## A8: Sequential identifierMap keys

The keys in `identifierMap` must be sequential integers starting from 0: `{0, 1, 2, 3, ...}` with no gaps. Gaps indicate lost variables during extraction.

- **Severity**: Error
- **Receipt**: ceb6f64, 80c4ca6
- **Check**: Sort numeric keys, verify they equal `[0, 1, 2, ..., n-1]`

## A9: pieces.length === identifiers.length + 1

The template interpolation model requires exactly one more piece than identifiers. Pieces are the static text segments between variable interpolation points.

- **Severity**: Error
- **Receipt**: 8c91932, 347a18f
- **Check**: Only when `identifiers` exists and is non-empty. Verify `pieces.length === identifiers.length + 1`

## A10: identifiers and identifierMap both present

If a prompt has a non-empty `identifiers[]`, it must also have a non-empty `identifierMap`, and vice versa.

- **Severity**: Error

## A11: disallowedTools entries are valid tool names

Every entry in `agentMetadata.disallowedTools` must be a recognized Claude Code tool name (see known-values.md). Unrecognized names like `"pq"` indicate the extraction left an unresolved minified variable.

- **Severity**: Error
- **Receipt**: c601cc4 — minified `"pq"` instead of `"Agent"`

## A12: tools must be an array of valid tool names

`agentMetadata.tools` must be an array (not a string or other type), and each entry must be a known tool name.

- **Severity**: Error
- **Receipt**: e2ea59e — tools was the string `"(inherited from Explore)"`

## A13: whenToUse should not be null

`agentMetadata.whenToUse` being `null` typically means a getter function wasn't resolved during extraction.

- **Severity**: Warning
- **Receipt**: e2ea59e

## A14: No unknown agentMetadata fields

Every field in `agentMetadata` must be from the known set (see known-values.md). Unknown fields indicate incomplete extraction.

- **Severity**: Warning
- **Receipt**: e2ea59e

## A15: Prompt name format

Prompt names must start with a recognized category prefix (see known-values.md).

- **Severity**: Error

## A16: No duplicate filename slugs

Convert each prompt name to its filename slug using the nameToFilename algorithm (below). No two prompts should produce the same slug. Also check for degenerate slugs (`.md`, contains `--`).

- **Severity**: Error

### nameToFilename algorithm

1. Identify the category prefix from the name (e.g., `"System Prompt: "` → slug prefix `"system-prompt-"`)
2. Take the remaining text after the prefix
3. Lowercase it
4. Replace non-alphanumeric characters with hyphens
5. Collapse consecutive hyphens
6. Trim leading/trailing hyphens
7. Concatenate: `slug-prefix + slug-body + ".md"`

Example: `"System Prompt: Main system prompt"` → `"system-prompt-main-system-prompt.md"`

## A17: No literal ${VAR} references in pieces

Static text pieces should not contain literal `${KNOWN_VARIABLE_NAME}` strings where the name matches any variable defined in ANY prompt's identifierMap across the file.

This catches un-interpolated references — the extraction left a raw `${VAR}` in static text instead of splitting it into pieces + identifiers.

- **Severity**: Error
- **Receipt**: f983cda, 45689b1
- **Check**: First, collect ALL variable names from all prompts' identifierMaps into a set. Then scan each piece for `${NAME}` where NAME is in that set.

## A18: No self-referencing variable calls

Detect patterns like `FN(FN)` where the same variable appears at consecutive identifier positions with a `(` between them. This indicates the extraction duplicated a variable.

- **Severity**: Error
- **Receipt**: 347a18f — `JSON_STRINGIFY_FN(JSON_STRINGIFY_FN, null, 2)`
- **Check**: For each pair of consecutive identifiers at positions `j` and `j+1`, if both map to the same variable name AND `pieces[j+1]` starts with `(` but doesn't contain `)`, flag it.

## A19: Description double periods

Descriptions ending with `..` (but NOT `...` which is a valid ellipsis) indicate a truncation or concatenation bug. These propagate to the README.

- **Severity**: Error
- **Receipt**: 4d9eb7b, 65964fe — 19 double-period README entries
- **Check**: After normalizing whitespace, check if description ends with `..` but not `...`

## A20: id ↔ name slug consistency

The `id` field should share meaningful words with the `name` field. Low overlap suggests the id was carried over from a different prompt.

- **Severity**: Warning
- **Receipt**: d7654a5 — id `"update-agent-docs"` for name `"Update Magic Docs"`
- **Check**: Extract content words (excluding stop words like "system", "prompt", "tool", "description", "agent", "data", "skill", "reminder", "parameter", "a", "the", "for", "of", "in", "to", "and", "is", "with") from both name and id. If less than 30% overlap, flag it.

## A21: Prompts sorted by id

The `prompts[]` array must be sorted by `id` (lexicographic ascending). Report the first out-of-order pair.

- **Severity**: Error
- **Receipt**: 91cf6b1, 9a69739
