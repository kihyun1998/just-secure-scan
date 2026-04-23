---
name: jcr-refactor
description: "Refactoring suggestions. Use when the user requests refactoring, code structure improvement, or code cleanup. Provides concrete refactoring proposals with actual code changes."
---

# JCR Refactor — Refactoring Suggestions

If code quality review (jcr-review) is about "finding and reporting problems," this skill is about "suggesting how to actually fix them with code."

## Determining the Refactoring Target

If no arguments are given, `git diff` (unstaged changes) is the default target.

If arguments are provided, determine the target as follows:
- `--staged` → `git diff --staged`
- `--branch` → diff against the base branch (`git diff <base-branch>...HEAD`). The base branch is auto-detected via `git symbolic-ref refs/remotes/origin/HEAD`. If `base-branch` is specified in `.jcr.md`, that takes precedence.
- File/folder path → read the code at that path directly

## References

Shares references with jcr-review. Used as criteria for refactoring decisions:

- `../jcr-review/references/dead-code.md` — identifying code to remove
- `../jcr-review/references/duplication.md` — identifying extraction/consolidation targets
- `../jcr-review/references/complexity.md` — identifying separation/simplification targets
- `../jcr-review/references/naming.md` — suggesting name improvements
- `../jcr-review/references/error-handling.md` — improving error handling
- `../jcr-review/references/magic-values.md` — suggesting constant extraction
- `../jcr-review/references/comments.md` — cleaning up comments
- `../jcr-review/references/style.md` — unifying style
- `../jcr-review/references/flutter-dart.md` — Flutter/Dart specific refactoring (widget decomposition, state management, async patterns)

## Refactoring Procedure

1. If `.jcr.md` exists, read it first to check project conventions and exclusion patterns.
2. Read the target code and understand the overall context.
3. Reference the references to identify areas for improvement.
4. Present before/after code for each refactoring.
5. Explain the reason and effect of each refactoring.

## Refactoring Types

- **Extract Function** — Separate independent logic from long functions into their own functions
- **Extract Variable/Constant** — Give names to magic values or complex expressions
- **Simplify Conditionals** — Flatten complex conditionals using guard clauses, early returns, etc.
- **Remove Duplication** — Consolidate repeated code into common functions/modules
- **Improve Naming** — Change to clearer names
- **Improve Error Handling** — Replace empty catches, swallowed errors, etc. with proper handling
- **Remove Dead Code** — Clean up unused variables, imports, functions

## Output Format

Must follow this format:

```
## Refactoring Summary

- Total N refactoring suggestions
- Main types: type(count), ...

## Refactoring Suggestions

### 1. [Title] — `file_path:line_range`

**Reason**: Why this refactoring is needed

**Before**:
(current code)

**After**:
(improved code)

**Effect**: Benefits gained from this refactoring
```

Output language follows the user's language.

## Guidelines

- Only suggest refactorings that do not change behavior. Feature additions/modifications are outside this skill's scope.
- Suggest up to 5-7 items at a time. Present the highest priority items first, and if there are additional improvement opportunities, indicate "N more possible."
- For code with tests, suggest within the scope that won't break tests.
- Respect project conventions (`.jcr.md`).
- When suggesting refactorings, consider potential overflow issues (integer overflow, buffer overflow, array index out-of-bounds, etc.). Ensure suggested code changes do not introduce or overlook overflow risks.
