---
name: to-issues
description: Break a plan, spec, or PRD into independently-grabbable GitHub issues using tracer-bullet vertical slices, GitHub issue dependencies, and AFK/HITL readiness labels. Use when the user wants to convert a plan into implementation issues, create tickets, or break down work for agents.
---

# To Issues

Break a plan into independently-grabbable GitHub issues using vertical slices, also known as tracer bullets.

The output should be optimized for an agent loop that can safely pick issues labeled `ready-for-agent`, skip blocked issues by reading GitHub issue dependency relationships, and work through the graph until no eligible issues remain.

## Core model

Use GitHub issue dependencies as the source of truth for blockers.

- If issue B cannot start until issue A is complete, then issue B is **blocked by** issue A.
- The issue body may include a human-readable dependency summary, but it is not the source of truth.
- Do not rely on prose such as `Blocked by: #12` as the only dependency record.
- Do not append `(COMPLETED)` to issue bodies when blockers finish.
- The agent loop should re-check GitHub dependencies each time it picks the next issue.

## Labels

Use workflow labels carefully. Do not remove domain labels like `bug`, `frontend`, `backend`, `auth`, `priority-high`, etc.

Required workflow labels:

- `ready-for-agent` — this issue is eligible for the agent loop once dependencies are satisfied.
- `in-progress` — an agent is currently working on this issue.
- `complete` — an agent completed the issue.
- `hitl` — requires human interaction before or during implementation.
- `needs-human` — not currently safe for an AFK agent.

Recommended behavior:

- AFK implementation issues should receive `ready-for-agent`.
- HITL issues should receive `hitl` and `needs-human`, not `ready-for-agent`, unless the user explicitly wants agents to open planning/design tasks.
- Blocked AFK issues may still receive `ready-for-agent`; the loop must skip them until their GitHub dependencies are complete.
- Completion should remove only workflow labels that no longer apply, such as `ready-for-agent` and `in-progress`, then add `complete`.

## Process

### 1. Gather context

Work from whatever is already in the conversation context.

If the user passes an issue reference, issue URL, PRD URL, file path, or project reference as an argument:

- Fetch it.
- Read the full body.
- Read relevant comments.
- Preserve important constraints, decisions, and user stories.

If the source is an existing GitHub issue, record it as the parent issue.

### 2. Explore the codebase

If you have not already explored the codebase, inspect enough of the repository to understand:

- The project structure.
- Existing patterns.
- Naming conventions.
- Domain vocabulary.
- Relevant tests.
- Existing ADRs or decisions in the touched area.

Issue titles and descriptions should use the project’s own vocabulary.

Avoid over-specifying implementation details unless they are important decisions.

### 3. Draft vertical slices

Break the plan into tracer-bullet issues.

Each issue must be a thin vertical slice that cuts through all necessary integration layers end-to-end. Do not create horizontal tickets like “create database schema”, “create API”, and “create UI” unless those are independently useful and demoable.

Slices may be:

- **AFK** — can be implemented by an agent without human interaction.
- **HITL** — requires human input, such as product decision, architecture review, design review, copy approval, data access, or credentials.

Prefer AFK slices where possible, but do not pretend ambiguous work is AFK.

<vertical-slice-rules>
- Each slice delivers a narrow but complete path through the product.
- A completed slice is demoable, testable, or verifiable on its own.
- Prefer many thin slices over few thick slices.
- Each slice should have clear acceptance criteria.
- Each slice should have minimal dependencies.
- Do not create dependency chains longer than necessary.
- Avoid circular dependencies.
</vertical-slice-rules>

### 4. Build the dependency graph

Before publishing, create a dependency graph between the proposed slices.

For each issue, identify:

- Direct blockers only.
- Whether the blocker is technical, product, design, data, or sequencing.
- Whether the blocker can be removed by splitting the issue differently.

Rules:

- Use direct dependencies only. Do not list transitive dependencies.
- If C is blocked by B and B is blocked by A, C should usually list only B.
- Avoid circular dependencies.
- If two slices block each other, split or reorder them.
- If a slice has many blockers, consider whether it is too large.
- If a slice blocks many others, make it small and early.

### 5. Quiz the user

Present the proposed breakdown as a numbered list.

For each slice, show:

- **Title**
- **Type**: AFK / HITL
- **Blocked by**: other proposed slices, by title for now
- **Unblocks**: other proposed slices, by title for now
- **User stories covered**
- **Why this slice is independently useful**

Ask the user:

- Does the granularity feel right: too coarse, too fine, or good?
- Are the dependency relationships correct?
- Should any slices be merged or split?
- Are the right slices marked AFK vs HITL?
- Should any issue be intentionally excluded from `ready-for-agent`?

Iterate until the user approves the breakdown.

### 6. Publish issues

Publish issues in dependency order, blockers first.

Use this strategy:

1. Create all approved issues.
2. Store the real GitHub issue number for each created issue.
3. Fetch and store each issue’s REST numeric `id`.
4. Update issue bodies to replace placeholders with real issue numbers.
5. Create GitHub issue dependency relationships using the REST API.
6. Apply labels.
7. Verify the dependency graph from GitHub after publishing.

Important:

- The visible issue number is used for human references like `#42`.
- GitHub’s issue dependency API requires the blocker issue’s REST numeric `id`.
- Do not confuse the visible issue number with the REST numeric `id`.
- Do not use the GraphQL node ID for the dependency API.

Example conceptually:

If issue `#50` is blocked by issue `#42`:

- `#50` is the blocked issue.
- `#42` is the blocking issue.
- Add a `blocked_by` relationship to `#50` using `#42`’s REST numeric issue `id`.

### 7. Issue body template

Use this body template for each issue.

<issue-template>
## Parent

A reference to the parent issue, if the source was an existing issue.

Use `#<number>` for issue references.

Omit this section if there is no parent issue.

## What to build

A concise description of this vertical slice.

Describe the end-to-end behavior, not a layer-by-layer implementation checklist.

Focus on what should be true when the issue is done.

Avoid specific file paths or code snippets unless they encode an important decision that would be ambiguous in prose.

## Acceptance criteria

- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

## Dependency summary

This section is for humans and agents to read.

GitHub issue dependencies are the source of truth.

Blocked by:

- None

Blocking:

- None

## Agent notes

- This issue is intended for: AFK or HITL
- The agent should not work on unrelated issues.
- The agent should run relevant tests, typechecks, linting, or build checks before marking complete.
- The agent should only say `COMPLETE` when all acceptance criteria are satisfied.

## Branch

`<type>-<issue-number>-<short-slug>`

Suggested branch naming convention.

Replace `<issue-number>` with the actual GitHub issue number after publishing.

Types:

- `feat-`
- `fix-`
- `chore-`
- `refactor-`

Example:

`feat-42-user-profile-page`
</issue-template>

### 8. Dependency summary format

After real issue numbers are known, update the dependency summary in each body.

For an unblocked issue:

```md
## Dependency summary

This section is for humans and agents to read.

GitHub issue dependencies are the source of truth.

Blocked by:

- None

Blocking:

- #45
- #46
```

For a blocked issue:

```md
## Dependency summary

This section is for humans and agents to read.

GitHub issue dependencies are the source of truth.

Blocked by:

- #42
- #43

Blocking:

- #50
```

Do not use checkboxes as the primary source of dependency state.

Do not write:

```md
- [x] #42
```

unless the project has explicitly chosen markdown task lists as a fallback dependency mechanism.

### 9. GitHub dependency API usage

When GitHub issue dependencies are available, use the dependency API after creating issues.

To fetch an issue’s REST numeric ID:

```bash
gh api "repos/OWNER/REPO/issues/ISSUE_NUMBER" --jq '.id'
```

To mark `CHILD_ISSUE_NUMBER` as blocked by `BLOCKER_ISSUE_ID`:

```bash
gh api \
  --method POST \
  "repos/OWNER/REPO/issues/CHILD_ISSUE_NUMBER/dependencies/blocked_by" \
  -f issue_id=BLOCKER_ISSUE_ID
```

To list what an issue is blocked by:

```bash
gh api "repos/OWNER/REPO/issues/ISSUE_NUMBER/dependencies/blocked_by"
```

To list what an issue is blocking:

```bash
gh api "repos/OWNER/REPO/issues/ISSUE_NUMBER/dependencies/blocking"
```

After adding dependencies, verify them:

```bash
gh api "repos/OWNER/REPO/issues/ISSUE_NUMBER/dependencies/blocked_by" \
  --jq 'map({number, title, state})'
```

### 10. Fallback when GitHub dependencies are unavailable

If the GitHub issue dependency API is unavailable, unsupported, or fails due to permissions, fall back to a structured body block.

Use this exact format:

```md
<!-- ralph:dependencies:start -->
## Dependency summary

Dependency source: markdown-fallback

Blocked by:

- #42
- #43

Blocking:

- #50
<!-- ralph:dependencies:end -->
```

Rules for fallback mode:

- Keep the block machine-editable.
- Use issue references only, like `#42`.
- Do not use full URLs.
- Do not append `(COMPLETED)`.
- The agent loop should check whether referenced blockers are closed or labeled `complete`.
- If all blockers are closed or labeled `complete`, the issue is eligible.

### 11. Labeling rules

For each created issue:

AFK and no human decision required:

- Add `ready-for-agent`.

AFK but blocked by other issues:

- Add `ready-for-agent`.
- Add GitHub dependency relationships.
- The loop will skip the issue until blockers are complete.

HITL:

- Add `hitl`.
- Add `needs-human`.
- Do not add `ready-for-agent` unless explicitly requested.

Parent/epic/meta issue:

- Do not add `ready-for-agent`.
- Use a label like `epic`, `plan`, or `tracking` if available.

### 12. After publishing

After all issues are created:

- Replace placeholder issue references with real `#<number>` references.
- Replace placeholder branch names with real issue numbers.
- Add GitHub issue dependency relationships.
- Verify there are no circular dependencies.
- Verify every blocked issue has a GitHub dependency relationship.
- Verify every issue’s body has an accurate dependency summary.
- Do not close or modify the parent issue unless explicitly asked.

### 13. Final response to user

After publishing, summarize:

- Created issues.
- Which issues are ready for agent.
- Which issues are HITL.
- Dependency order.
- Any issues blocked by human decisions.
- Any dependency relationships that could not be created.

Use a compact dependency tree.

Example:

```text
Created 6 issues:

Ready now:
- #42 Add basic account settings page
- #43 Add profile update API

Blocked:
- #44 Add avatar upload, blocked by #42
- #45 Add profile completion banner, blocked by #42 and #43

HITL:
- #46 Confirm billing copy
```

If dependency API calls failed, clearly say so and mention that the markdown fallback block was used instead.
