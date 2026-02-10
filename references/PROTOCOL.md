# Agnostic Interaction Protocol

This module defines the standard protocol for soliciting user input and flow control. It ensures that Review Buddy remains interactive regardless of whether it is running in a rich IDE, a specialized agent environment, or a basic CLI.

## Protocol: Soliciting User Intent

Every time the agent reaches a "User Gate" (a point where the user must make a choice), it must follow this logic:

### 1. Detection
Identify available interaction tools:
- **`AskUserQuestion`**: The primary tool for structured selection (Gemini CLI / specialized agents).
- **`gh pr comment`**: For remote interaction if running in a headless CI/bot environment.
- **`STDOUT`**: The universal fallback.

### 2. Execution

#### Preference A: Structured Tool (e.g., Gemini CLI)
If `AskUserQuestion` is available, call it with clear, numbered options.

#### Preference B: Standard I/O (Basic CLI)
If no specialized tool is available, output a **Protocol Block**:

```markdown
### 🛑 REVIEW GATEWAY: {Phase/Context}

{Brief summary of current state or findings}

**Select an action:**
1. [Option 1]
2. [Option 2]
3. [Option 3]

*Please type the number or the command to proceed.*
```

### 3. The "Stop" Rule
The agent **MUST** terminate its current turn after presenting the Protocol Block or calling the tool. It cannot assume "Proceed" unless the user explicitly provides that intent in the next turn.

## Standard Options Reference

Use these standard labels for consistency:

| Context | Recommended Options |
|---------|---------------------|
| **Phase Gate** | `Proceed`, `Cancel`, `Change Focus` |
| **Chunk Loop** | `Continue`, `Deep-dive`, `Mark for GitHub`, `Add Todo`, `Pause & Save` |
| **Deep-dive Loop**| `Analyze Context`, `Suggest Fix`, `Check Similar Patterns`, `Return to Chunk` |
| **Synthesis** | `Post Review`, `Edit Findings`, `Save Locally`, `Discard` |

## Agnostic Input Handling

When the user responds:
- **"1" or "Proceed"**: Execute the primary next step.
- **"Add Todo: <text>"**: Divert to `scripts/todo-manager.md` to save a task, then re-prompt.
- **"Deep-dive: {finding_id}"**: Enter a specialized sub-turn. Use `search_file_content` to find similar patterns in the codebase to validate if the finding is a recurring issue or an isolated case.
- **"Pause"**: Trigger the state save and exit.
