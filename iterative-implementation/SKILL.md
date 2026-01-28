---
name: iterative-implementation
description: "Implements tasks from an approved spec using configurable implementation and review models. Creates git worktrees per task, runs implementation→review loops until approved, commits incrementally, and squashes on completion. Tracks progress in the spec file itself."
---

# Iterative Implementation

Implements tasks from an approved spec with configurable implementation and review models, git worktree isolation, and incremental commits.

## When to Use

- After a spec has been approved (e.g., via `/iterative-plan-review`)
- To implement a multi-task feature with review gates
- When you want isolated branches per task with squash-on-complete

## Prerequisites

- `codex` CLI installed and authenticated (if using codex for impl or review)
- `claude` CLI available (if using claude for impl or review)
- Git repository with clean working state
- Approved spec file with parseable tasks

## Configuration

| Setting | Default | Options |
|---------|---------|---------|
| Implementation model | `claude` | `codex`, `claude` |
| Review model | `claude` | `codex`, `claude` |
| Max review iterations | `5` | 1-10 |
| Worktree base path | `.worktrees/` | Any path (gitignored recommended) |
| Branch prefix | `impl/` | Any prefix |
| Auto-approve task breakdown | `false` | `true`, `false` |

Override by saying:
- "Use claude for implementation and codex for review"
- "Set max review iterations to 3"
- "Use branch prefix `feature/`"
- "Auto-approve task breakdowns" or "Don't ask about task breakdown"

## Task Format in Specs

The skill parses tasks from the spec file. Supported formats:

### Simple Checklist

```markdown
## Tasks

- [ ] Implement user authentication endpoint
- [ ] Add input validation
- [x] Set up project structure (completed)
- [ ] Write integration tests
```

### Sectioned Checklist

```markdown
## Backend Tasks

- [ ] Create database schema
- [ ] Implement REST endpoints

## Frontend Tasks

- [ ] Build login form
- [ ] Add error handling
```

### With Branch Tracking (added automatically)

```markdown
## Tasks

- [ ] Implement auth endpoint
  - branch: `impl/auth-endpoint`
  - status: `in-review` | `approved` | `committed`
- [x] Set up project structure
  - branch: `impl/project-structure`
  - status: `squashed`
```

## Core Flow

```
┌─────────────────────────────────────────────────────────────────┐
│  Input: specs/feature.md (approved)                             │
│                              ↓                                  │
│  1. Parse tasks from spec                                       │
│  2. For each incomplete task:                                   │
│     ┌─────────────────────────────────────────────────────┐     │
│     │ a. Check if already claimed                         │     │
│     │    ├─ Claimed → prompt: resume / abandon / skip     │     │
│     │    └─ Unclaimed → continue                          │     │
│     │ b. CLAIM on main first (commit + push)              │     │
│     │ c. Create worktree + branch from main               │     │
│     │ d. Implementation model implements                  │     │
│     │ e. Commit (iteration N)                             │     │
│     │              ↓                                      │     │
│     │ f. Review model reviews                             │     │
│     │              ↓                                      │     │
│     │ g. Decision:                                        │     │
│     │    ├─ APPROVED → exit inner loop                    │     │
│     │    ├─ ISSUES → fix, commit (N+1), goto (f)          │     │
│     │    ├─ NEEDS_CLARIFICATION → ask user                │     │
│     │    └─ MAX_ITERATIONS → ask user                     │     │
│     └─────────────────────────────────────────────────────┘     │
│     h. Squash commits for this task                             │
│     i. Mark task complete in spec (commit to main)              │
│                              ↓                                  │
│  3. All tasks done → summary                                    │
└─────────────────────────────────────────────────────────────────┘
```

## Phase 1: Parse Spec and Identify Tasks

Read the spec file and extract tasks:

```bash
# Example: specs/open-sourcing.md
cat specs/open-sourcing.md
```

Parse for:
- Unchecked items: `- [ ]` 
- Already checked: `- [x]` (skip)
- Existing branch annotations (resume in-progress work)

Build task list:
```
Task 1: "Implement user authentication endpoint" [pending]
Task 2: "Add input validation" [pending]  
Task 3: "Set up project structure" [complete, skip]
Task 4: "Write integration tests" [pending]
```

## Phase 1b: Task Breakdown (if needed)

If the spec lacks explicit checklist tasks (`- [ ]` format), generate them from the spec content.

### Step 1: Analyze spec structure

Look for implementable units:
- Numbered steps in "Approach" or "Implementation" sections
- Distinct features or components described
- Logical boundaries (backend/frontend, by endpoint, by module)

### Step 2: Generate checklist

Create tasks at appropriate granularity:
- Each task should be completable in one worktree session
- Tasks should be independently testable where possible
- Order tasks by dependency (foundations first)

### Step 3: Approval (configurable)

**If `auto_approve_breakdown` is `false` (default):**

```
📋 No explicit tasks found in spec. Proposed breakdown:

## Tasks
- [ ] Set up OAuth2 provider interface
- [ ] Implement token storage schema
- [ ] Add callback endpoint
- [ ] Implement token refresh logic
- [ ] Add integration tests

Options:
1. Accept this breakdown
2. Edit (provide your own task list)
3. Specify different granularity (more/fewer tasks)
```

Wait for user confirmation before proceeding.

**If `auto_approve_breakdown` is `true`:**

```
📋 No explicit tasks found in spec. Auto-generated breakdown:

## Tasks
- [ ] Set up OAuth2 provider interface
- [ ] Implement token storage schema
- [ ] Add callback endpoint
- [ ] Implement token refresh logic
- [ ] Add integration tests

Proceeding with implementation...
```

### Step 4: Commit breakdown to spec

Add the tasks to the spec file and commit:

```bash
# Edit spec to add Tasks section
git add specs/feature.md
git commit -m "Add task breakdown for: [spec name]"
```

This ensures the breakdown is versioned and visible to other workers.

## Phase 2: Claim Task and Set Up Worktree

For each pending task:

### 2a. Check task status

First, check if task is already claimed:

```markdown
# Unclaimed - available to work on
- [ ] Implement user authentication endpoint

# Claimed - someone started this
- [ ] Implement user authentication endpoint
  - branch: `impl/implement-user-auth`
  - status: `claimed`

# Complete - skip
- [x] Implement user authentication endpoint
  - branch: `impl/implement-user-auth`
  - status: `complete`
```

**If task is already claimed**, prompt user:

```
⚠️ Task "Implement user authentication endpoint" is already claimed.

Branch: impl/implement-user-auth
Status: claimed (not complete)

Options:
1. Resume work on existing branch
2. Abandon claim and start fresh (deletes existing branch)
3. Skip this task and move to next
```

Wait for user input before proceeding.

### 2b. Claim task on main FIRST (prevents race conditions)

**CRITICAL: Claim must be committed to main before creating worktree.**

This prevents multiple workers from claiming the same task.

```bash
# Ensure we're on main and up to date
git checkout main
git pull origin main  # if using remote

# Generate branch name from task
# "Implement user authentication endpoint" → impl/implement-user-auth
BRANCH_NAME="impl/implement-user-auth"
```

Update the spec to mark as claimed:

```markdown
- [ ] Implement user authentication endpoint
  - branch: `impl/implement-user-auth`
  - status: `claimed`
```

Commit the claim immediately:

```bash
git add specs/open-sourcing.md
git commit -m "Claim task: Implement user authentication endpoint

Branch: impl/implement-user-auth
Worker: claude/codex"

# Push if using remote (makes claim visible to other workers)
git push origin main  # optional, for multi-worker setups
```

### 2c. Create worktree from claimed main

Now that the claim is recorded, create the worktree:

```bash
# Ensure base directory exists and is gitignored
mkdir -p .worktrees
grep -q "^.worktrees/$" .gitignore || echo ".worktrees/" >> .gitignore

# Create worktree with new branch from main (which now has the claim)
git worktree add .worktrees/implement-user-auth -b impl/implement-user-auth main
```

### 2d. Resuming a claimed task

If user chose to resume an existing claimed task:

```bash
# Check if worktree already exists
if [ -d ".worktrees/implement-user-auth" ]; then
    # Worktree exists, just cd into it
    cd .worktrees/implement-user-auth
else
    # Worktree was removed but branch exists
    git worktree add .worktrees/implement-user-auth impl/implement-user-auth
    cd .worktrees/implement-user-auth
fi

# Check current state
git log --oneline origin/main..HEAD  # commits since branch point
git status                            # any uncommitted changes
```

Then continue from Phase 3 (implementation) or Phase 4 (review) based on state.

### 2e. Abandoning a claimed task

If user chose to abandon and start fresh:

```bash
# Remove worktree if exists
git worktree remove .worktrees/implement-user-auth --force 2>/dev/null || true

# Delete the branch
git branch -D impl/implement-user-auth 2>/dev/null || true

# Update spec to unclaim
# Change status: `claimed` back to unclaimed (remove metadata)

git add specs/open-sourcing.md
git commit -m "Abandon claim: Implement user authentication endpoint"
git push origin main  # if using remote
```

Then restart Phase 2 from the beginning for this task.

## Phase 3: Implementation Loop

### 3a. Send task to implementation model

**If implementation model is `codex`:**

```bash
cd .worktrees/implement-user-auth

echo 'Implement the following task:

Task: Implement user authentication endpoint

Context from spec:
[RELEVANT SPEC SECTIONS]

Requirements:
- Follow existing code patterns in this repository
- Include appropriate error handling
- Add inline comments for complex logic

When complete, respond with IMPLEMENTATION_COMPLETE and summarize what you did.' \
| codex exec --sandbox workspace-write
```

**If implementation model is `claude`:**

```bash
cd .worktrees/implement-user-auth

claude --print "Implement the following task:

Task: Implement user authentication endpoint

[SAME PROMPT AS ABOVE]" 2>/dev/null
```

### 3b. Commit after implementation

```bash
cd .worktrees/implement-user-auth

git add -A
git commit -m "WIP: Implement user authentication endpoint (iteration 1)

[Auto-generated by iterative-implementation]
Implementation model: codex"
```

## Phase 4: Review Loop

### 4a. Send changes to review model

Generate a diff for review:

```bash
cd .worktrees/implement-user-auth

# Get diff from branch point
DIFF=$(git diff origin/main...HEAD)
```

**If review model is `claude`:**

```bash
echo "Review these code changes:

## Task
Implement user authentication endpoint

## Spec Requirements
[RELEVANT REQUIREMENTS FROM SPEC]

## Changes
\`\`\`diff
$DIFF
\`\`\`

## Review Criteria
1. Does the implementation satisfy the task requirements?
2. Are there bugs, logic errors, or edge cases missed?
3. Does it follow the codebase's existing patterns?
4. Are there security concerns?
5. Is error handling appropriate?

Respond with:
- APPROVED - if changes are good to merge
- ISSUES_FOUND - list specific issues to fix
- NEEDS_CLARIFICATION - list questions for the user" \
| claude --print 2>/dev/null
```

**If review model is `codex`:**

```bash
echo "[SAME PROMPT]" | codex exec --sandbox read-only
```

### 4b. Handle review verdict

**APPROVED:**
```
✅ Task approved by reviewer.

Proceeding to squash commits and mark complete.
```

**ISSUES_FOUND:**
```
Issues found by reviewer:
1. [Issue 1]
2. [Issue 2]

Sending back to implementation model for fixes...
```

Then loop back to Phase 3 with:

```bash
echo 'Fix the following issues in the current implementation:

Issues:
1. [Issue 1]
2. [Issue 2]

Make targeted fixes. Do not refactor unrelated code.
Respond with FIXES_COMPLETE when done.' \
| codex exec --sandbox workspace-write

# Commit the fixes
git add -A
git commit -m "WIP: Fix review feedback (iteration 2)

Issues addressed:
- [Issue 1]
- [Issue 2]"
```

**NEEDS_CLARIFICATION:**
```
⏸️ Reviewer needs clarification:

[QUESTIONS]

Please provide answers to continue.
```

Wait for user input, then continue loop.

**MAX_ITERATIONS:**
```
⚠️ Review loop reached maximum iterations (5).

Current state:
- [N] commits on branch
- Outstanding issues: [ISSUES]

Options:
1. Approve current state and continue
2. Provide guidance on remaining issues
3. Abandon this task and move on
```

## Phase 5: Complete Task

### 5a. Squash commits

```bash
cd .worktrees/implement-user-auth

# Count commits since branch point
COMMIT_COUNT=$(git rev-list --count origin/main..HEAD)

# Squash all commits into one
git reset --soft origin/main
git commit -m "Implement user authentication endpoint

[Detailed description of what was implemented]

Spec: specs/open-sourcing.md
Task: Implement user authentication endpoint
Reviewed-by: claude
Iterations: 3"
```

### 5b. Update spec to mark complete

Back on main branch, update the spec:

```markdown
- [x] Implement user authentication endpoint
  - branch: `impl/implement-user-auth`
  - status: `complete`
```

Commit:
```bash
git add specs/open-sourcing.md
git commit -m "Mark complete: Implement user authentication endpoint"
```

### 5c. Clean up worktree (optional)

```bash
# Remove worktree but keep branch
git worktree remove .worktrees/implement-user-auth

# Or keep worktree for reference until all tasks done
```

## Phase 6: Repeat or Finish

Continue to next pending task, or if all complete:

```
✅ All tasks implemented!

Summary:
- Task 1: Implement user authentication endpoint ✓
  - Branch: impl/implement-user-auth
  - Iterations: 3
- Task 2: Add input validation ✓
  - Branch: impl/add-input-validation
  - Iterations: 2
- Task 3: Write integration tests ✓
  - Branch: impl/write-integration-tests
  - Iterations: 1

Next steps:
1. Review branches and merge to main
2. Or create PR(s) for team review
```

## When to Stop and Ask User

### Requirement Ambiguity
```
⏸️ The task description is ambiguous:

Task: "Add input validation"

Questions:
- Which endpoints need validation?
- What validation rules should apply?
- Should validation errors return 400 or 422?

Please clarify so I can continue.
```

### Trade-off Decisions
```
⏸️ Trade-off requires your input:

Task: "Implement caching"

Options:
1. In-memory cache (simple, loses data on restart)
2. Redis (persistent, adds infrastructure dependency)
3. SQLite (persistent, no new dependencies, slower)

Which approach should I take?
```

### Max Iterations
```
⚠️ Review loop reached 5 iterations without approval.

[Show current state and outstanding issues]
```

### Implementation Failure
```
❌ Implementation model failed to complete task:

Error: [ERROR MESSAGE]

Options:
1. Retry with different approach
2. Provide guidance
3. Skip this task
```

### Merge Conflicts
```
⚠️ Merge conflict detected in worktree.

Conflicting files:
- src/auth/handler.ts

Options:
1. Show me the conflicts to resolve manually
2. Abort and retry task from scratch
```

## Error Recovery

### Resuming Interrupted Work

The skill detects claimed-but-incomplete tasks from the spec:

```markdown
- [ ] Implement user authentication endpoint
  - branch: `impl/implement-user-auth`
  - status: `claimed`
```

When this is detected:

```
⚠️ Task "Implement user authentication endpoint" is already claimed.

Branch: impl/implement-user-auth

Checking branch state...
- Commits since main: 2
- Last commit: "WIP: Fix review feedback (iteration 2)"
- Uncommitted changes: none

Options:
1. Resume work (continue with review)
2. Abandon claim and start fresh
3. Skip this task
```

If worktree exists:
```bash
cd .worktrees/implement-user-auth
# Continue from current state
```

If worktree was removed but branch exists:
```bash
git worktree add .worktrees/implement-user-auth impl/implement-user-auth
cd .worktrees/implement-user-auth
# Continue from current state
```

### Abandoning a Task

```bash
# Remove worktree
git worktree remove .worktrees/implement-user-auth --force

# Delete branch
git branch -D impl/implement-user-auth

# Update spec to remove claim
git checkout main
# Edit spec to remove branch/status metadata from task
git add specs/open-sourcing.md
git commit -m "Abandon claim: Implement user authentication endpoint"
```

### Handling Concurrent Workers

The claim-on-main-first approach prevents race conditions:

1. Worker A reads spec, sees Task 1 unclaimed
2. Worker A commits claim to main, pushes
3. Worker B pulls main, sees Task 1 claimed
4. Worker B moves to Task 2

If two workers try to claim simultaneously:
- First push wins
- Second push fails with merge conflict
- Second worker pulls, sees claim, moves to next task

```bash
# Worker B's push fails
git push origin main
# error: failed to push some refs
# hint: Updates were rejected because the remote contains work that you do not have locally.

# Worker B recovers
git pull --rebase origin main
# Now sees Task 1 is claimed, picks Task 2 instead
```

## Gotchas

### From iterative-plan-review

1. **Suppress stderr for clean parsing**: `2>/dev/null`
2. **Use consistent models**: Don't switch impl/review models mid-task
3. **Token efficiency**: Send diffs, not full files when possible

### Git Worktree Specifics

4. **Worktrees share object database**: Efficient, but don't delete `.git`
5. **Can't checkout same branch twice**: Each worktree needs unique branch
6. **Worktree path in .gitignore**: Add `.worktrees/` to avoid committing
7. **Claim before worktree**: Always commit claim to main BEFORE creating worktree to prevent race conditions

### Cross-Model Communication

7. **File paths are relative to worktree**: `cd` into worktree before running impl model
8. **Main branch spec updates**: Switch back to main to update progress tracking

### Commit Hygiene

9. **WIP commits are throwaway**: They get squashed, so messages can be brief
10. **Final squashed commit matters**: Include task, spec reference, reviewer

## Example Session

**User:** "Implement the tasks in specs/open-sourcing.md using codex for implementation and claude for review"

**Claude:**
```
📋 Parsed 3 tasks from specs/open-sourcing.md:

1. [ ] Remove proprietary dependencies
2. [ ] Add MIT license  
3. [ ] Write contributing guidelines

Starting with task 1...

🌿 Creating worktree: .worktrees/remove-proprietary-deps
   Branch: impl/remove-proprietary-deps

📝 Updating spec with branch tracking...

🔨 Sending to codex for implementation...
```

**[Codex implements]**

**Claude:**
```
💾 Committed: "WIP: Remove proprietary dependencies (iteration 1)"

🔍 Sending to claude for review...
```

**[Claude reviews]**

**Claude:**
```
Issues found:
1. Left a reference to internal package in package.json
2. Missing license header in copied utility file

🔨 Sending fixes to codex...
```

**[Codex fixes]**

**Claude:**
```
💾 Committed: "WIP: Fix review feedback (iteration 2)"

🔍 Sending to claude for review...

✅ Approved!

🗜️ Squashing 2 commits...
💾 Final commit: "Remove proprietary dependencies"

📝 Marking task complete in spec...

Moving to task 2...
```

## Files in This Skill

```
iterative-implementation/
├── SKILL.md                    # This file
└── templates/
    ├── implementation-prompt.md          # Implementation prompt template
    └── review-prompt.md        # Review prompt template
```