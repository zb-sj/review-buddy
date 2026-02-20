# Chunk Planner (Phase 3)

You are performing Phase 3 of a chunked PR review. Your goal is to intelligently group changed files into **semantically coherent chunks** — each chunk should tell a self-contained story about one logical change, feature, or concern.

## Prerequisites

- Phase 1 (Context Assembly) completed — `changed_files` list, `commits`, `pr_metadata` available
- Phase 2 (Comments Digest) completed — `comments_by_file` available

## Core Principle: Semantic Chunking

**Group files by the logical change they participate in, not by file type or directory.**

A single "Add user avatar upload" feature might touch a type definition, an API route, a React component, a CSS file, and a test — all of these belong in the same chunk because they form one reviewable unit of intent. Reviewing them separately would force the reviewer to mentally reconstruct the connections.

## Steps

### 1. Identify Logical Change Units

Analyze the PR to discover semantic groupings using multiple signals:

#### 1a. Commit-Based Signals
Each commit often represents one logical unit. Examine which files each commit touches:
```
For each commit in commits:
  commit_files = files changed in that commit
  commit_message = commit summary
  → potential logical unit
```

If the PR uses clean, atomic commits, this is the strongest signal.

#### 1b. Dependency-Based Signals
Scan diffs for import/require statements to build a dependency graph among changed files:
- `import ... from './relative/path'`
- `require('./relative/path')`
- `from relative.module import ...` (Python)

Files that import each other are very likely part of the same logical change.

#### 1c. Naming & Co-location Signals
Files that share a name stem or live in the same feature directory often belong together:
- `UserAvatar.tsx` + `UserAvatar.test.tsx` + `UserAvatar.module.css` → same unit
- `features/checkout/api.ts` + `features/checkout/types.ts` + `features/checkout/CheckoutForm.tsx` → same unit
- `migrations/001_add_avatar.sql` + `models/user.ts` → same unit (schema + model)

#### 1d. PR Description Signals
Parse the PR body for structure cues (headings, bullet lists of changes) that describe distinct logical changes.

### 2. Build Semantic Groups

Combine the signals from Step 1 to form semantic groups:

```
semantic_groups = []

1. Start with commit-based groupings as a baseline
2. Merge groups that share dependencies (files that import each other)
3. Merge groups that share name stems (e.g., Component + Component.test + Component.css)
4. Split any group that mixes truly unrelated changes (e.g., a commit that both adds a feature AND fixes a typo elsewhere)
```

Each semantic group gets a **descriptive label** derived from:
- The commit message(s) that touch those files
- The dominant file names or directory
- The nature of the change (e.g., "Avatar upload API + UI", "Config & tooling updates", "Checkout validation tests")

### 3. Handle Cross-Cutting Files

Some files don't belong to a single feature — they serve multiple logical units:

| Type | Examples | Strategy |
|------|----------|----------|
| Shared types/interfaces | `types/index.ts`, `models/schema.ts` | Attach to the group that primarily consumes them. If consumed by multiple groups, place in the **first chunk** so reviewers see definitions before usage. |
| Config changes | `package.json`, `tsconfig.json`, `.env` | Group together as a small "Config & Dependencies" chunk, reviewed first. |
| Global styles | `global.css`, `theme.ts` | Attach to the UI group they most affect, or make a standalone chunk if significant. |
| Documentation | `README.md`, `CHANGELOG.md` | Attach to the feature they document, or group as "Docs" at the end. |

### 4. Order Chunks for Reviewability

Arrange chunks so that earlier chunks provide context for later ones:

1. **Foundation first**: Chunks containing type definitions, schemas, or shared utilities go early
2. **Dependency order**: If Chunk B's files import from Chunk A's files, A comes first
3. **Core before periphery**: Backend/API logic before frontend UI before tests
4. **Config & docs**: Minor config at the start; docs at the end
5. **Within a chunk**: Order files from definitions → implementation → usage → tests

### 5. Apply Size Guardrails

After semantic grouping, apply size limits to keep chunks reviewable:

- **Target**: ~300 LOC per chunk (acceptable range: 150–500 LOC)
- LOC = additions + deletions for each file

```
final_chunks = []

for each semantic_group:
    group_loc = sum(file.additions + file.deletions for file in group.files)

    # Fits within limits — keep as-is
    if group_loc <= 500:
        final_chunks.append(semantic_group)
        continue

    # Too large — split by sub-concern within the semantic group
    # Prefer splitting along natural boundaries:
    #   1. Implementation vs. tests
    #   2. Backend vs. frontend
    #   3. Core logic vs. supporting files (styles, configs)
    # Label split chunks as "{group_label} (part 1: logic)", "{group_label} (part 2: tests)", etc.
    sub_chunks = split_by_sub_concern(semantic_group, target=300)
    final_chunks.extend(sub_chunks)

# Merge tiny leftover groups (< 80 LOC) with their most related neighbor
for chunk in final_chunks:
    if chunk.loc < 80 and has_related_neighbor(chunk, final_chunks):
        merge_into_neighbor(chunk, final_chunks)
```

**Splitting heuristic for large semantic groups:**
1. Separate tests from implementation (most natural split)
2. Separate type definitions from their consumers
3. Separate UI (components, styles) from logic (services, utils)
4. As a last resort, split by subdirectory

### 6. Handle Edge Cases

- **Single-file PR**: Skip chunking entirely. Display: "Single file change — reviewing directly."
- **Tiny PR (< 100 total LOC)**: Suggest quick mode. Display: "This PR is small ({N} LOC). Consider using `--quick` for a faster review."
- **Huge PR (50+ files)**: Display warning: "This PR changes {N} files. The review will be divided into {M} semantic chunks and may take a while. Consider focusing on specific areas with `--focus`."
- **Only deleted files**: Note that review will focus on verifying safe removal (no broken imports, no lost functionality).
- **Only renamed/moved files**: Note that review will focus on verifying rename correctness and updated imports.
- **Squash-style PR (1 giant commit)**: Commit signals are useless — rely heavily on dependency, naming, and co-location signals instead.

### 7. Display Chunk Plan

Present the plan in this format:

```
## Review Plan

{edge_case_message_if_any}

| Chunk | Semantic Group | Files | LOC | Est. Time | Existing Comments |
|-------|---------------|-------|-----|-----------|-------------------|
| 1 | Config & dependencies | `package.json`, `tsconfig.json` | 45 | ~2 min | 0 |
| 2 | User avatar upload | `types/user.ts`, `api/avatar.ts`, `components/Avatar.tsx`, `Avatar.module.css` | 320 | ~5 min | 2 (1 open) |
| 3 | Checkout validation | `services/checkout.ts`, `utils/validate.ts` | 280 | ~5 min | 5 (2 open) |
| 4 | Tests | `__tests__/avatar.test.ts`, `__tests__/checkout.test.ts` | 200 | ~3 min | 0 |
| **Total** | | **{N} files** | **{LOC}** | **~{time} min** | **{comments}** |

Review order: config → avatar upload → checkout validation → tests
```

Time estimate heuristic: ~1 min per 60 LOC, minimum 2 min per chunk.

The "Existing Comments" column uses data from `comments_by_file` to show how many existing review comments touch each chunk's files, with a breakdown of open/verify threads.

### 8. User Gate

Ask the user how to proceed:

**Options:**
1. **Start from chunk 1** — Begin sequential review
2. **Skip to chunk N** — Jump to a specific chunk (show chunk numbers)
3. **Review all at once** — Skip chunking, review everything in one pass (not recommended for large PRs)
4. **Cancel** — Stop the review

Present this gate following the **Agnostic Interaction Protocol** (`references/PROTOCOL.md`).

### 9. Output

Store the following for use by subsequent phases:
- `chunks` — ordered list of chunks, each with: files, total LOC, semantic label, estimated time
- `chunk_plan_display` — the formatted table for state persistence
- `user_start_chunk` — which chunk to start from (default: 1)
- `semantic_groups` — map of group label → files for reference
