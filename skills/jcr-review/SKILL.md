---
name: jcr-review
description: "Code quality review. Use when the user requests code review, code quality check, or code inspection. Can review by git diff, file, or folder."
---

# JCR Review — Code Quality Review

A review skill focused on code quality. Not about security or bug detection, but about "can this code be made cleaner?"

## Determining the Review Target

If no arguments are given, `git diff` (unstaged changes) is the default target.

If arguments are provided, determine the target as follows:
- `--staged` → `git diff --staged`
- `--branch` → diff against the base branch (`git diff <base-branch>...HEAD`). The base branch is auto-detected via `git symbolic-ref refs/remotes/origin/HEAD`. If `base-branch` is specified in `.jcr.md`, that takes precedence.
- File/folder path → read and review the code at that path directly

When there are no changes (diff is empty or no target files):

```
## Review Summary

No changes found to review.
```

## Excluded from Review

The following are not reviewed:
- Auto-generated code (`*.generated.*`, `*.g.*`, `*.gen.*`)
- vendor, node_modules, lock files (`package-lock.json`, `yarn.lock`, `Cargo.lock`, etc.)
- Test fixtures, snapshot files
- Patterns explicitly excluded in `.jcr.md`

## Project-Specific Configuration

Search for a `.jcr.md` file in the current working directory (cwd). For monorepos, look at the git repository root. If not found, only the default rules from references are applied.

Supported sections in `.jcr.md`:
- **base-branch**: Default comparison branch (e.g., `develop`, `master`)
- **Naming conventions**: Project naming rules
- **Ignore patterns**: Files/directories/situations to exclude from review
- **Style conventions**: Code style rules
- **Additional rules**: Project-specific review rules
- **Project info**: Tech stack hints (optional)

## Review Procedure

1. If `.jcr.md` exists, read it first (to understand exclusion patterns and conventions).
2. Read the target code (applying `.jcr.md` exclusion patterns).
3. Identify the language/framework of the code.
4. Review using the relevant references below that apply to the target code (`.jcr.md` conventions take precedence).
5. Format the results according to the output format.

## References

Selectively reference the appropriate references based on the characteristics of the target code:

- `references/dead-code.md` — when detecting unnecessary variables, unused imports, unreachable code
- `references/comments.md` — when evaluating comment accuracy, necessity, or missing comments
- `references/naming.md` — when evaluating variable, function, class naming
- `references/duplication.md` — when identifying code duplication or reusable patterns
- `references/complexity.md` — when evaluating function length, branching complexity, separation of concerns
- `references/magic-values.md` — when detecting magic numbers or hardcoded strings
- `references/error-handling.md` — when evaluating error handling patterns
- `references/style.md` — when checking code style consistency
- `references/flutter-dart.md` — when reviewing Flutter/Dart code (widget design, state management, async patterns, Dart conventions)

## Severity Criteria

- **error** — Clear quality issue. Must be fixed. (e.g., unused variables, empty catch blocks, fully duplicated code)
- **warning** — Could be improved. (e.g., vague naming, excessive function length, magic numbers)
- **info** — For reference. Optional improvement. (e.g., extractable common patterns, style inconsistencies)

## Output Format

Must follow this format:

```
## Review Summary

- Total N issues (error: X, warning: Y, info: Z)
- Main areas: category(count), ...

## Review Results

### [Category Name]
- [severity] `file_path:line_number` - Issue description
  → Improvement suggestion
```

When no issues are found:

```
## Review Summary

No notable quality issues were found in the reviewed code.
```

Output language follows the user's language.

## Handling Large Code

When the diff or target files are hundreds of lines or more:
- Report error-severity issues first.
- Prioritize reviewing areas where changes are concentrated (hotspots).
- Rather than listing every issue, show representative patterns and summarize repeated patterns as "N additional instances found."

## Guidelines

- Focus on substantive quality improvements over trivial nitpicks.
- If project conventions (rules specified in `.jcr.md`) conflict with the default rules in references, project conventions take precedence. For matters not specified in `.jcr.md`, apply the default rules.
- Understand the code's intent before reviewing. Do not mechanically apply rules without context.
- For each issue, concisely explain "why it's a problem."
- If the same issue falls under multiple categories, report it only under the single most appropriate category. Do not duplicate reports.
- When suggesting improvements, consider potential overflow issues (integer overflow, buffer overflow, array index out-of-bounds, etc.). If a refactoring or improvement could introduce or overlook an overflow risk, flag it explicitly.
