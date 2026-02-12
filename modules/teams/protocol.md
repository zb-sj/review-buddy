# Agnostic Agent-to-Agent (A2A) Protocol

This protocol defines how the Leader and Teammates communicate in a model-independent way.

## 1. Message Formats

All A2A messages must be wrapped in specialized Markdown code blocks for easy parsing.

### [A2A-TASK-ASSIGNMENT]
Sent by Leader to Teammate.
```markdown
### [A2A-TASK-ASSIGNMENT]
**Role:** {Persona Name}
**Chunk ID:** {N}
**Diff Context:** 
{Diff}
**Instruction:** Analyze this diff through your specialized lens.
```

### [A2A-TECHNICAL-REPORT]
Sent by Teammate to Leader.
```markdown
### [A2A-TECHNICAL-REPORT]
**Findings:**
| Severity | Location | Description | Reasoning |
|----------|----------|-------------|-----------|
| {S}      | {L}      | {D}         | {R}       |
```

### [A2A-CHALLENGE-REQUEST]
Sent by Leader to Teammate.
```markdown
### [A2A-CHALLENGE-REQUEST]
**Source Report:** {Report ID}
**Content:** {Report Content}
**Instruction:** Review these findings. Identify false positives, conflicts with your specialty, or better alternatives.
```

### [A2A-CHALLENGE-RESPONSE]
Sent by Teammate to Leader.
```markdown
### [A2A-CHALLENGE-RESPONSE]
**Critique:**
- {Finding ID}: {Agree/Disagree/Alternative} — {Reasoning}
```

## 2. Orchestration Rules

1. **Hierarchy**: Teammates never communicate directly; the Leader acts as the hub.
2. **Parallelism**: Discovery tasks can be assigned to all Teammates simultaneously.
3. **Synthesis**: The Leader must consolidate all A2A reports into a single `all_findings` list for the session.
