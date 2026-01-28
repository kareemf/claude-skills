# Implementation Review Prompt Template

---

Review these code changes for the following task:

## Task
{{TASK_DESCRIPTION}}

## Requirements from Spec
{{SPEC_REQUIREMENTS}}

## Changes

```diff
{{GIT_DIFF}}
```

## Files Modified
{{FILES_CHANGED}}

---

## Review Criteria

1. **Correctness**
   - Does the implementation satisfy the task requirements?
   - Are there logic errors or bugs?
   - Are edge cases handled?

2. **Completeness**
   - Is the task fully implemented?
   - Are there missing pieces?
   - Does it handle error conditions?

3. **Code Quality**
   - Does it follow existing codebase patterns?
   - Is the code readable and maintainable?
   - Are there unnecessary changes or scope creep?

4. **Security**
   - Are there security vulnerabilities?
   - Is input validation adequate?
   - Are secrets/credentials handled properly?

5. **Testing**
   - Are new code paths testable?
   - Should tests be added (if not already)?

---

## Your Verdict

Respond with ONE of:

**APPROVED**
```
APPROVED

The implementation satisfies the task requirements.

Notes (optional):
- [Any minor suggestions that don't block approval]
```

**ISSUES_FOUND**
```
ISSUES_FOUND

Issues that must be fixed:

1. [Issue description]
   - File: [path]
   - Problem: [what's wrong]
   - Suggestion: [how to fix]

2. [Next issue...]
```

**NEEDS_CLARIFICATION**
```
NEEDS_CLARIFICATION

Cannot complete review without answers to:

1. [Question about requirements]
2. [Question about intended behavior]
```
