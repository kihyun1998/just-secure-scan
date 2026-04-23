# Duplication — Code Duplication / Reuse

## Why It Matters

Duplicated code multiplies the cost of change. When you modify one place, you must modify the others too, and missing one creates a bug. However, not all duplication is bad. Sometimes a bit of duplication is better than a premature abstraction.

## Principles

- Consider extraction when the same code is repeated 3 or more times (Rule of Three).
- Two repetitions don't need extraction yet. Wait until the pattern is established.
- Even if code looks similar, it's not duplication if the reasons for change are different (accidental duplication).
- If the cost of abstraction (increased complexity) exceeds the cost of duplication, keep the duplication.

## Checklist

- [ ] Identical code blocks repeated 3 or more times
- [ ] Similar functions created by copy-paste (only parameters differ)
- [ ] Same utility logic across multiple files
- [ ] Same validation/transformation logic in multiple places
- [ ] Constant strings/configuration values hardcoded in multiple places (see magic-values reference)
- [ ] Same boilerplate (auth, JSON parsing, error response) repeated across all API route handlers
- [ ] Inline repetition despite a common utility function already existing (see dead-code reference)
- [ ] Client-side identical logic (e.g., error response parsing) repeated across multiple components

## Good Examples / Bad Examples

Bad example — identical logic repeated:
```
// File A
const fullName = `${user.firstName} ${user.lastName}`;
const initials = `${user.firstName[0]}${user.lastName[0]}`;

// File B (same logic)
const displayName = `${member.firstName} ${member.lastName}`;
const memberInitials = `${member.firstName[0]}${member.lastName[0]}`;
```

Good example — common function extracted:
```
function formatFullName(person) {
  return `${person.firstName} ${person.lastName}`;
}

function getInitials(person) {
  return `${person.firstName[0]}${person.lastName[0]}`;
}
```

## Anti-patterns

- **Premature Abstraction** — Extracting immediately after seeing something twice. Abstracting before the pattern is established can lead to wrong boundaries.
- **Excessive DRY** — Trying to eliminate all duplication can make code harder to understand. An abstraction that forces readers to jump between multiple files is worse than duplication.
- **Removing Accidental Duplication** — Forcibly merging code that looks similar but changes for different reasons. Separating it later costs more.
- **Inheritance for Deduplication** — Creating inheritance hierarchies solely to remove duplication. Composition is more appropriate in most cases.
- **Utility Bypass** — Repeating the same logic inline at each call site despite a common function already existing. This causes subtle behavioral divergence between the utility and inline versions, leading to bugs.
- **Boilerplate Copy-Paste** — Copying auth/parsing/error handling patterns for every API route handler. Consider extracting into middleware or wrapper functions.
