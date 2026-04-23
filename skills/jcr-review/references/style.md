# Style — Code Style Consistency

## Why It Matters

Style inconsistency increases cognitive load for readers. If some parts of the same codebase use camelCase while others use snake_case, you have to think "what's the convention in this file?" every time. Style is not about right or wrong — it's about consistency.

## Principles

- Follow the existing style within the project. Don't impose your own style.
- If linter/formatter configuration exists, that is the source of truth.
- Separate style changes from functional changes into different commits.
- For new projects, follow the language/framework's official style guide.

## Checklist

- [ ] Naming case inconsistency (mixing camelCase / snake_case / PascalCase)
- [ ] Indentation inconsistency (mixing tabs / spaces, 2-space / 4-space)
- [ ] String quote inconsistency (mixing single quotes / double quotes)
- [ ] Import order/grouping inconsistency
- [ ] Semicolon usage inconsistency
- [ ] Blank line usage inconsistency (between functions, between blocks)
- [ ] Brace style inconsistency (same line / next line)
- [ ] Comparison operator inconsistency (mixing `==` / `===`)
- [ ] File structure inconsistency (same type of files written with different structures)
- [ ] Inconsistent language in API error response messages (some in one language, some in another)
- [ ] Inconsistent i18n approach (some using `t()` function, some using inline ternary operators)

## Good Examples / Bad Examples

Bad example — mixed styles in the same file:
```
const user_name = getUserName();
const userAge = getAge();
const UserEmail = getEmail();

if (isValid == true) {
  console.log("valid")
}
if (isActive === false) {
  console.log('inactive');
}
```

Good example — consistent style:
```
const userName = getUserName();
const userAge = getAge();
const userEmail = getEmail();

if (isValid) {
  console.log("valid");
}
if (!isActive) {
  console.log("inactive");
}
```

## Anti-patterns

- **Style Wars** — Team members insisting on different styles. Automate with linters/formatters to eliminate debates.
- **Mixing Style Changes with Feature Commits** — Pollutes the diff, making it hard to identify actual changes.
- **Ignoring/Disabling Linter Rules** — Overusing `// eslint-disable`, `# noqa`, etc. If you must disable a rule, leave a comment explaining why.
- **Excessive Style Nitpicking** — Only pointing out style issues in reviews. Delegate automatable concerns to tools, and let reviewers focus on logic.
- **Inconsistent i18n Strategy** — Mixing `t("key", locale)` functions and `isKorean ? "Korean text" : "English text"` ternary operators in the same codebase. Unify to a single approach for easier maintenance.
- **Mixed Error Message Languages** — Validation errors in one language and business errors in another within the same API layer. Unify the error message language or apply i18n.
