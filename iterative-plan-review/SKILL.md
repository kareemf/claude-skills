---
name: iterative-plan-review
description: "Orchestrates iterative plan review between Claude (planner) and a reviewer model (default: Codex). Claude creates plans, the reviewer validates and provides feedback, and the loop continues until the plan is approved, user input is required, or max iterations reached."
---

# Iterative Plan Review

Orchestrates a feedback loop where Claude creates plans and a configurable reviewer (default: Codex) validates them.

## When to Use

- Before implementing complex features that need validation
- When you want a second opinion on architecture decisions
- For plans that affect multiple files or systems
- When the user asks for plan review or validation

## Configuration

The reviewer model is configurable. Defaults shown below.

| Setting | Default | Options |
|---------|---------|---------|
| Reviewer CLI | `codex` | `codex`, `claude` (subagent), or any CLI that accepts piped input |
| Sandbox mode | `read-only` | `read-only`, `workspace-write`, `danger-full-access` |
| Max iterations | `5` | 1-10 |
| Wait timeout | `120s` | Adjust based on plan complexity |

To override, the user can say things like:
- "Use claude as the reviewer instead of codex"
- "Set max iterations to 3"
- "Allow workspace writes for the reviewer"

## Core Loop

```
┌─────────────────────────────────────────────────────────┐
│  1. Claude creates/refines plan                         │
│                          ↓                              │
│  2. Send plan to reviewer (codex exec)                  │
│                          ↓                              │
│  3. Parse reviewer feedback                             │
│                          ↓                              │
│  4. Decision point:                                     │
│     ├─ APPROVED → Exit loop, proceed to implementation  │
│     ├─ NEEDS_CLARIFICATION → Ask user, then continue    │
│     ├─ ISSUES_FOUND → Refine plan, go to step 1         │
│     └─ MAX_ITERATIONS → Present final state to user     │
└─────────────────────────────────────────────────────────┘
```

## Phase 1: Create Initial Plan

Before invoking the reviewer, Claude creates a structured plan:

```markdown
## Plan: [Feature/Task Name]

### Objective
[Clear statement of what this plan achieves]

### Approach
1. [Step 1]
2. [Step 2]
...

### Files Affected
- `path/to/file1.ts` - [what changes]
- `path/to/file2.ts` - [what changes]

### Risks & Edge Cases
- [Risk 1]
- [Risk 2]

### Open Questions (if any)
- [Question that may need user input]
```

## Phase 2: Send to Reviewer

### Using codex exec (Default)

**CRITICAL: Use `codex exec` for non-interactive, piped execution.**

```bash
# Basic review request
echo 'Review this implementation plan and identify any issues:

[PLAN CONTENT HERE]

Analyze for:
- Logic errors or flawed assumptions
- Missing edge cases
- Security concerns
- Performance implications
- Architectural issues

Respond with one of:
1. APPROVED - if the plan is sound
2. ISSUES_FOUND - list specific issues that need addressing, (ordered by severity with file/line references)
3. NEEDS_CLARIFICATION - list questions requiring user input' | codex exec --sandbox read-only
```

### Recommended Flags

| Flag | Purpose |
|------|---------|
| `--sandbox read-only` | Reviewer cannot modify files (safest) |
| `--json` | Get structured JSON output for parsing |
| `--model gpt-5-codex` | Specify model (optional) |
| `-c 'model_reasoning_effort="high"'` | Request deeper analysis |

### Full Command Template

```bash
echo '[REVIEW_PROMPT]' | codex exec \
  --sandbox read-only \
  --json \
  2>/dev/null
```

**Note:** `2>/dev/null` suppresses stderr (thinking tokens) to avoid context bloat. Remove if debugging.

### Session Resume for Multi-Turn

If refinements are needed, resume the same session for context continuity:

```bash
# First review
echo '[INITIAL_PLAN]' | codex exec --sandbox read-only --json

# Subsequent reviews (after refinement)  
echo 'Review the updated plan:

[REFINED_PLAN]

Previous issues raised:
[LIST OF ISSUES]

Confirm if these have been addressed.' | codex exec resume --last --sandbox read-only
```

## Phase 3: Parse Reviewer Response

Look for these signals in the response:

### Approval Signals
- "APPROVED"
- "plan is sound"
- "no issues found"
- "ready for implementation"
- "looks good"

### Issue Signals
- "ISSUES_FOUND"
- "concern about..."
- "missing..."
- "should consider..."
- "potential problem..."
- Numbered lists of problems

### Clarification Signals
- "NEEDS_CLARIFICATION"
- "unclear whether..."
- "depends on..."
- "need to know..."
- "question for the user..."

## Phase 4: Iterate or Exit

### On APPROVED
```
✅ Plan approved by reviewer after [N] iteration(s).

[FINAL PLAN]

Ready to proceed with implementation?
```

### On ISSUES_FOUND (iteration < max)
1. Log the issues
2. Refine the plan to address each issue
3. Go back to Phase 2

### On NEEDS_CLARIFICATION
```
⏸️ Reviewer needs clarification:

[LIST OF QUESTIONS]

Please provide answers so I can refine the plan.
```
Wait for user response, incorporate answers, continue loop.

### On MAX_ITERATIONS Reached
```
⚠️ Maximum iterations ([N]) reached without full approval.

Current plan state:
[LATEST PLAN]

Outstanding concerns from reviewer:
[REMAINING ISSUES]

Options:
1. Proceed with current plan (accept risks)
2. Provide guidance on unresolved issues
3. Abandon and take different approach
```

## Gotchas & Edge Cases

### From claude-code-tools (tmux-cli)

These issues apply if using tmux panes instead of `codex exec`:

1. **Always send Enter as a separate argument**
   ```bash
   # WRONG - Enter inside quotes
   tmux send-keys -t $PANE "codex review plan\n"
   
   # CORRECT - Enter as separate argument
   tmux send-keys -t $PANE "codex review plan" Enter
   ```

2. **Add delays to prevent race conditions**
   - Fast CLI apps may not be ready for input immediately
   - Use `sleep 1` or `sleep 1.5` between send-keys and Enter

3. **Capture output properly**
   ```bash
   # Capture last N lines from pane
   tmux capture-pane -p -t $PANE | tail -20
   
   # Capture with extended history
   tmux capture-pane -p -S -100 -t $PANE
   ```

### From orchestrating-tmux-claudes

4. **Verify pane state before sending**
   ```bash
   # Check if pane shows a prompt (ready for input)
   tmux capture-pane -p -t $PANE | tail -1 | grep -q '>' && echo "ready"
   ```

5. **Respect timeouts**
   - Codex/Claude CLI typically timeout at 120 seconds
   - Don't wait longer than 110 seconds before checking status

6. **Session targeting format**
   ```bash
   # Full format: session:window.pane
   tmux send-keys -t mysession:0.1 "command" Enter
   
   # Or get current window dynamically
   WINDOW=$(tmux display-message -p '#I')
   tmux send-keys -t $SESSION:$WINDOW.$PANE "command" Enter
   ```

### From codex-claude-loop

7. **Suppress thinking tokens in piped mode**
   ```bash
   # Redirect stderr to avoid context bloat
   echo "prompt" | codex exec --sandbox read-only 2>/dev/null
   ```

8. **Use consistent models throughout**
   - Don't switch reviewer models mid-loop
   - Different models may have different opinions, causing infinite loops

### From claude_code_bridge

9. **Token efficiency**
   - Send concise prompts, not full file contents
   - The reviewer can read files itself if needed
   - Aim for 50-200 tokens per review request, not 5k-20k

10. **Session persistence**
    - `codex exec resume --last` continues the most recent session
    - Useful for multi-turn refinements with context

## Error Handling

### Reviewer CLI Not Found
```
❌ Reviewer CLI not available: codex

Options:
1. Install codex: npm install -g @openai/codex
2. Use claude subagent as reviewer instead
3. Skip review and proceed with plan
```

### Timeout During Review
```
⏱️ Reviewer timed out after 120s

The plan may be too complex for single-pass review.

Options:
1. Break plan into smaller pieces
2. Retry with simpler prompt
3. Proceed without full review
```

### Parse Failure
If reviewer response doesn't match expected patterns:
```
⚠️ Could not determine reviewer verdict.

Raw response:
[RESPONSE]

Treating as ISSUES_FOUND. Please review manually or clarify.
```

## Example Session

**User:** "Plan a refactor of the authentication module to support OAuth2"

**Claude (Iteration 1):**
```markdown
## Plan: OAuth2 Authentication Refactor

### Objective
Add OAuth2 support while maintaining existing session-based auth.

### Approach
1. Create OAuth2Provider interface
2. Implement GoogleOAuth2Provider
3. Add /auth/oauth2/callback endpoint
4. Store OAuth tokens in existing sessions table
...
```

**Sending to Codex...**

**Codex Response:**
```
ISSUES_FOUND:
1. Storing OAuth tokens in sessions table conflates concerns - tokens have different lifecycle
2. Missing PKCE flow for public clients
3. No token refresh strategy mentioned
```

**Claude (Iteration 2):**
```markdown
## Plan: OAuth2 Authentication Refactor (v2)

### Changes from v1
- Added separate oauth_tokens table
- Added PKCE support for public clients
- Defined token refresh strategy

### Approach
1. Create OAuth2Provider interface
2. Create oauth_tokens table with refresh_token, expires_at
3. Implement PKCE challenge/verifier generation
4. Add background job for token refresh before expiry
...
```

**Sending to Codex...**

**Codex Response:**
```
APPROVED - Plan addresses previous concerns. Token lifecycle is properly separated, PKCE added, refresh strategy is sound.
```

**Claude:**
```
✅ Plan approved by reviewer after 2 iterations.

Ready to proceed with implementation?
```

## Alternative: Using Claude Subagent as Reviewer

If Codex is unavailable or user prefers Claude-to-Claude review:

```bash
# Launch a Claude subagent for review
claude --print "Review this plan for issues: [PLAN]" 2>/dev/null
```

Or use a background subagent with TodoWrite/TodoRead for async handoff.

## Files in This Skill

```
iterative-plan-review/
├── SKILL.md              # This file
└── templates/
    └── review-prompt.md  # Optional: customizable review prompt template
```
