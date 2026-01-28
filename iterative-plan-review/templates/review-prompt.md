# Review Prompt Template

Use this template when sending plans to the reviewer. Customize as needed.

---

Review this implementation plan and identify any issues:

## Plan Summary
{{PLAN_TITLE}}

## Objective
{{OBJECTIVE}}

## Proposed Approach
{{APPROACH}}

## Files Affected
{{FILES}}

## Risks Identified by Planner
{{RISKS}}

---

Analyze the plan for:

1. **Logic & Correctness**
   - Are there flawed assumptions?
   - Will the approach actually achieve the objective?
   - Are there logical gaps in the steps?

2. **Edge Cases**
   - What scenarios might break this approach?
   - Are error conditions handled?
   - What about empty/null/boundary inputs?

3. **Security**
   - Are there authentication/authorization gaps?
   - Input validation concerns?
   - Data exposure risks?

4. **Performance**
   - Will this scale appropriately?
   - Are there N+1 query patterns?
   - Memory or CPU concerns?

5. **Architecture**
   - Does this fit the existing codebase patterns?
   - Are there better alternatives?
   - Is the coupling/cohesion appropriate?

6. **Missing Pieces**
   - What isn't addressed that should be?
   - Are there implicit dependencies?
   - What about rollback/migration strategy?

---

Respond with ONE of these verdicts:

**APPROVED**
Use if the plan is sound and ready for implementation. You may include minor suggestions that don't block implementation / mention any residual risks or testing gaps.

**ISSUES_FOUND**
Use if there are problems that should be addressed before implementation. List each issue clearly, ordered by severity with file/line references, with:
- What the issue is
- Why it matters
- Suggested fix (if you have one)

**NEEDS_CLARIFICATION**
Use if the plan cannot be properly evaluated without more information. List specific questions that need answers from the user or planner.

---

Your response:
