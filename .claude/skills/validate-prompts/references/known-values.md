# Known Values Reference

Authoritative lists used by validation rules. If a value isn't on these lists, it's likely an extraction artifact.

## Known Tool Names

Valid entries for `agentMetadata.tools` and `agentMetadata.disallowedTools` (rules A11, A12):

```
Agent, AskUserQuestion, Bash, Edit, EnterPlanMode, EnterWorktree,
ExitPlanMode, Glob, Grep, LSP, NotebookEdit, Read, Skill, Sleep,
Task, TodoWrite, WebFetch, WebSearch, Write, *
```

Anything not on this list (e.g., `"pq"`, `"Fz"`) is an unresolved minified variable name.

## Known agentMetadata Fields

Valid fields inside `agentMetadata` (rule A14):

```
agentType, color, criticalSystemReminder, disallowedTools,
maxTurns, model, permissionMode, tools, toolsNote,
whenToUse, whenToUseDynamic
```

## Prompt Name Prefixes

Valid category prefixes for prompt names (rule A15) and their corresponding filename slug prefixes (rule A16):

| Name Prefix | Filename Slug |
|-------------|---------------|
| `Agent Prompt: ` | `agent-prompt-` |
| `System Prompt: ` | `system-prompt-` |
| `System Reminder: ` | `system-reminder-` |
| `Tool Description: ` | `tool-description-` |
| `Tool Parameter: ` | `tool-parameter-` |
| `Data: ` | `data-` |
| `Skill: ` | `skill-` |

## Stop Words for id/name Comparison

Words to exclude when comparing id and name overlap (rule A20):

```
system, prompt, tool, description, agent, data, skill,
reminder, parameter, a, the, for, of, in, to, and, is, with
```
