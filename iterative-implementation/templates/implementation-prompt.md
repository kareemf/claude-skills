# Implementation Prompt Template

---

Implement the following task:

## Task
{{TASK_DESCRIPTION}}

## Context from Spec
{{SPEC_CONTEXT}}

## Relevant Files
{{EXISTING_FILES}}

## Requirements
- Follow existing code patterns in this repository
- Include appropriate error handling
- Add inline comments for complex logic
- Do not modify files outside the scope of this task
- Do not refactor unrelated code

## Constraints
{{CONSTRAINTS_IF_ANY}}

---

Implement the task. When complete, respond with:

```
IMPLEMENTATION_COMPLETE

Summary:
- [What you created/modified]
- [Key implementation decisions]
- [Any assumptions made]
```

If you cannot complete the task, respond with:

```
BLOCKED

Reason: [Why you cannot proceed]
Question: [What you need answered]
```
