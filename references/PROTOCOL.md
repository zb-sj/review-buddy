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

The agent **MUST** terminate its current turn after presenting the gate. It **MUST NOT**:
- Assume "Proceed" and continue without user input
- Auto-select a default option after a timeout
- Combine the gate with the next phase's output in the same turn

**Enforcement**: If you find yourself writing analysis or fetching data after displaying a gate in the same turn, **STOP** — you are violating this rule. Present the gate, then yield control to the user.

### 4. Unrecognized Input

If the user's response doesn't match any option, number, label, or known free-form command:

1. Echo back what was received: `I didn't recognize "{input}".`
2. Show the available options again (brief form — just the numbered list, not the full context)
3. Yield control again — do **not** pick a default

This loop continues until the user provides a recognized input. Never silently ignore unrecognized input.

### 5. Confirmation on Destructive Actions

Before executing any destructive free-form command (`discard`, bulk `deselect`), confirm with the user:
- `"discard"` → "This will remove {N} findings from this chunk. They won't be recoverable. Proceed? [y/n]"
- `"discard"` in synthesis → "This will discard all {N} findings and exit without posting. Proceed? [y/n]"

If the user confirms (y/yes), proceed. If not, re-display the gate.

## Standard Options Reference

Use these standard labels for consistency across all user gates:

| Context | Primary Options | Free-form Commands |
|---------|----------------|-------------------|
| **Phase Gate** | `Proceed`, `Set focus`, `Quick mode`, `Cancel` | — |
| **Visual Discovery** | `Proceed with Visual Review`, `Skip Visual Review`, `Refresh Context` | — |
| **Chunk Plan** | `Start from chunk 1`, `Skip to chunk N`, `Review all at once`, `Cancel` | — |
| **Chunk Loop** | `Continue`, `Deep-dive`, `Pause & save`, `Skip to synthesis` | `discard`, `todo: {text}`, `deselect {IDs}` |
| **Deep-dive** | `Analyze context`, `Suggest fix`, `Check similar patterns`, `Return to chunk` | — |
| **Synthesis** | `Post all marked`, `Post only Action Required`, `Select specific`, `Save locally` | `discard` |
| **Post-only** | `Post all marked`, `Post only Action Required`, `Select specific`, `Cancel` | `toggle {IDs}`, `all`, `none` |
| **Continue (mismatch)** | `Resume saved session`, `Start fresh` | — |

## Agnostic Input Handling

When the user responds, normalize their input:
- **Number** (e.g., "1", "2"): Map to the corresponding option.
- **Label** (e.g., "Continue", "Proceed"): Match to the option by label (case-insensitive).
- **"todo: {text}"**: Divert to `scripts/todo-manager.md` to save a task, then re-prompt the same gate.
- **"deep-dive {finding_id}"**: Enter a specialized sub-turn for that finding, then return to the gate.
- **"deselect {IDs}"**: Unmark specified finding IDs from GitHub posting, then re-prompt.
- **"discard"**: Context-dependent — in chunk loop, removes chunk findings; in synthesis, exits without posting.
- **"pause"**: Trigger state save and exit.
