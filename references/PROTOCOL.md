# Agnostic Interaction Protocol

This module defines the standard protocol for soliciting user input and flow control. It ensures that Review Buddy remains interactive regardless of the host agent, IDE, or CLI environment.

## Protocol: Soliciting User Intent

Every time the agent reaches a "User Gate" (a point where the user must make a choice), it must follow this logic:

### 1. Detection

Check what interaction capabilities are available in the current runtime:
- **Structured selection tool** — e.g., `AskUserQuestion` (Claude Code), prompt UIs in IDE extensions, or equivalent in other agent frameworks.
- **Remote comment** — e.g., `gh pr comment` for headless CI/bot environments.
- **Standard I/O** — The universal fallback (plain text output + wait for user input).

### 2. Execution

#### Preference A: Structured Selection Tool
If a structured selection tool is available, use it with clear, numbered options. Respect the tool's constraints (e.g., max option count) — if the gate has more options than the tool supports, present primary actions via the tool and document additional commands in descriptions or as free-form input.

#### Preference B: Standard I/O
If no structured tool is available, output a **Protocol Block**:

```markdown
### 🛑 REVIEW GATEWAY: {Phase/Context}

{Brief summary of current state or findings}

**Select an action:**
1. [Option 1]
2. [Option 2]
3. [Option 3]
...

*Type a number or command to proceed.*
```

### 3. The "Stop" Rule
The agent **MUST** terminate its current turn after presenting the gate. It cannot assume "Proceed" unless the user explicitly provides that intent in the next turn.

## Standard Options Reference

Use these standard labels for consistency across all user gates:

| Context | Primary Options | Free-form Commands |
|---------|----------------|-------------------|
| **Phase Gate** | `Proceed`, `Set focus`, `Quick mode`, `Cancel` | — |
| **Chunk Plan** | `Start from chunk 1`, `Skip to chunk N`, `Review all at once`, `Cancel` | — |
| **Chunk Loop** | `Continue`, `Deep-dive`, `Pause & save`, `Skip to synthesis` | `discard`, `todo: {text}`, `deselect {IDs}` |
| **Deep-dive** | `Analyze context`, `Suggest fix`, `Check similar patterns`, `Return to chunk` | — |
| **Synthesis** | `Post all marked`, `Post only Action Required`, `Select specific`, `Save locally` | `discard` |

## Agnostic Input Handling

When the user responds, normalize their input:
- **Number** (e.g., "1", "2"): Map to the corresponding option.
- **Label** (e.g., "Continue", "Proceed"): Match to the option by label (case-insensitive).
- **"todo: {text}"**: Divert to `scripts/todo-manager.md` to save a task, then re-prompt the same gate.
- **"deep-dive {finding_id}"**: Enter a specialized sub-turn for that finding, then return to the gate.
- **"deselect {IDs}"**: Unmark specified finding IDs from GitHub posting, then re-prompt.
- **"discard"**: Context-dependent — in chunk loop, removes chunk findings; in synthesis, exits without posting.
- **"pause"**: Trigger state save and exit.
