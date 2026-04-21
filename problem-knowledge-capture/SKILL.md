---
name: problem-knowledge-capture
description: Use when coding produces errors, bugs appear, or commands fail - captures problems to /matter for later skill creation.
---

# Problem Knowledge Capture

**REQUIRED SUB-SKILL:** Use `superpowers:writing-skills` when creating or editing skills.

**REQUIRED:** Use subagent to create skills. Do NOT create skills directly in main session.

## When to Use

When you encounter:

- Compilation/interpretation errors
- Runtime bugs or exceptions
- Command execution failures
- Unexpected behavior in code

## Core Principle

**Matter files are append-only logs** - continuously capture the entire error chain from first occurrence to final resolution. Reference files are refined extracts from matter.

## Workflow

1. **Create Matter File**: On first error, create `/matter/{英文问题标题}-{timestamp}.md`

2. **Append Throughout**: As you debug (try solutions, encounter new errors, etc.), **append** to this file. Never overwrite.

3. **Mark Resolution**: When problem is solved, add "## Resolution" section at the end.

4. **Confirm**: Ask user if problem is solved and if they want to create a skill.

5. **Create/Update Skill** (if user confirms):
   
   - Use `superpowers:writing-skills` skill
   - Dispatch subagent with **only the most recent matter file** (or user-specified one)
   - Reference = refined extract from matter (concise, searchable)

## Matter File Naming

```
/matter/{english-problem-title}-{timestamp}.md
```

**Example**: `/matter/npm-peer-deps-conflict-1712001234567.md`

**Timestamp**: Unix milliseconds (from `Date.now()`)

**Rule**: Each error session = one matter file. Never load multiple matter files when creating a skill.

## Matter File Format (Append-Only)

```markdown
# {Problem Title}

**Created**: {timestamp}
**Status**: [In Progress / Resolved]

## Initial Error

[First error message + context]

## Debugging Log

### Attempt 1: {timestamp}
- Action: {what was tried}
- Result: {what happened}

### Attempt 2: {timestamp}
- Action: {what was tried}
- Result: {what happened}

## Resolution

[SOLUTION - final fix that worked]

## Key Learnings

[Brief note on root cause / prevention]
```

## Reference File Format

Reference files are **refined extracts** from matter files. Keep concise but complete.

```markdown
# {Problem Title}

## Symptoms
[How to recognize this problem - error keywords, messages]

## Root Cause
[Concise explanation of why]

## Solution
[Step-by-step fix - 3-5 steps max]

## Prevention
[How to avoid in future - optional]
```

## Skill Structure

```
matter-{category}/
  SKILL.md
  reference/
    {refined-problem-id}.md
```

## SKILL.md Format

```markdown
---
name: matter-{category}
description: Use when [specific problem symptoms].
---

# Matter {Category}

## Problems (≤50 chars each)

- {Problem 1 with error keyword}
- {Problem 2 with error keyword}

## Quick Reference

| Symptom | Reference |
|---------|-----------|
| {error keyword} | reference/{id}.md |
```

## Subagent Prompt Template

```
Create or Update (If this issue falls under an existing misclassified category) a matter skill for a resolved problem.

**IMPORTANT**: Only load ONE matter file - the most recent one or user-specified.

1. Read the matter file: /matter/{filename}.md
2. Extract and refine key information for reference file:
   - Symptoms (error keywords)
   - Root cause (concise)
   - Solution (step-by-step)
3. Create skill structure:
   - SKILL.md: ≤50 char problem descriptions with search keywords
   - reference/{id}.md: refined content from matter

Use superpowers:writing-skills for skill creation.
```

## Quality Requirements

**Matter files (append-only log):**

- Full error chain captured
- All debugging attempts logged
- Final resolution marked

**Reference files (refined extract):**

- ≤50 chars per problem in SKILL.md
- Include searchable error keywords
- Solution must be actionable (3-5 steps)

**Skill creation rules:**

- Always use subagent
- Always use superpowers:writing-skills
- Only one matter file per skill creation
- Reference must enable autonomous problem solving
