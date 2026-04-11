# Output Validation Rules (B1–B6, C1–C7)

These rules validate the generated markdown files in `system-prompts/` and `README.md`. Use after `tools/updatePrompts.js` has been run.

---

## Part B — Markdown Files

Read each `.md` file in `system-prompts/`. Each should have an HTML-comment frontmatter block at the top:

```markdown
<!--
name: 'System Prompt: Example'
description: >
  What this prompt does.
ccVersion: 2.1.XX
variables:
  - FIRST_VARIABLE
  - SECOND_VARIABLE
-->

Prompt content with ${FIRST_VARIABLE} placeholders...
```

### B1: Valid frontmatter fields

Every markdown file must have an HTML-comment frontmatter block (`<!-- ... -->`) containing at minimum `name` and `ccVersion` fields.

- **Severity**: Error

### B2: No duplicate variables in frontmatter

The `variables:` list must not contain the same variable name twice.

- **Severity**: Error
- **Receipt**: Live bug — `WRITE_TOOL_NAME` appeared twice in `plan-mode-is-active-iterative.md`

### B3: Frontmatter variables referenced in body

Every variable listed in the frontmatter `variables:` section should appear somewhere in the markdown body below the frontmatter. Unreferenced variables indicate stale frontmatter.

- **Severity**: Warning

### B4: Body variable references listed in frontmatter

Every `${VARIABLE_NAME}` pattern (where NAME matches `[A-Z_][A-Z_0-9]*`) found in the body should be listed in the frontmatter `variables:` section.

- **Severity**: Warning

### B5: Filename matches name

The markdown filename must match what `nameToFilename(prompt.name)` would produce (see the nameToFilename algorithm in structural-rules.md). Mismatches mean the file was manually renamed or the name changed without regenerating.

- **Severity**: Error

### B6: Description no double period

Same as A19 — description in frontmatter must not end with `..` (unless `...`).

- **Severity**: Error

---

## Part C — README

Parse `README.md`. Prompt entries follow this pattern:

```
- [**Name**](./system-prompts/filename.md) (**NNN** tks) - Description text.
```

or without bold on the name:

```
- [Name](./system-prompts/filename.md) (**NNN** tks) - Description text.
```

### C1: Every markdown file has a README entry

Every `.md` file that exists in `system-prompts/` must have a corresponding entry in the README.

- **Severity**: Error
- **Receipt**: 4d9eb7b — 7 Skill + 1 Tool Parameter prompts missing from README

### C2: Every README entry points to an existing file

Every file path referenced in a README entry must correspond to an actual file in `system-prompts/`.

- **Severity**: Error

### C3: No double periods in README descriptions

Descriptions in README entries must not contain `..` (unless `...` for ellipsis).

- **Severity**: Error
- **Receipt**: 4d9eb7b, 65964fe — 19 double-period entries

### C4: No known typos

Check for recurring typos in README:
- `Desscriptions` → should be `Descriptions`
- `Desciptions` → should be `Descriptions`

- **Severity**: Error
- **Receipt**: 65964fe — `"Tool Desscriptions"`

### C5: Section heading levels correct

The subsections `Sub-agents`, `Creation Assistants`, `Slash Commands`, and `Utilities` must be h4 (`####`), not h3 or other levels.

- **Severity**: Error
- **Receipt**: 4d9eb7b — h3 used where h4 needed

### C6: No duplicate README entries

Each file path must appear exactly once in the README. Duplicates indicate a generation bug.

- **Severity**: Error

### C7: Token counts are positive integers

The token count for each entry must be a positive integer.

- **Severity**: Warning
