# Comments — Comment Quality

## Why It Matters

Good comments explain the "why" that code cannot express. Bad comments either convey false information by falling out of sync with the code, or merely repeat the code and add noise. A wrong comment is more dangerous than no comment at all.

## Principles

- Comments explain "why." Let the code itself express "what."
- Do not write comments that literally translate the code.
- If you feel a comment is needed, first consider whether you can make the code itself clearer.
- Maintain comments alongside code. When you change code, update the comments.

## Checklist

- [ ] Comments that literally repeat the code (e.g., `// increment i by 1` → `i++`)
- [ ] Comments that don't match the code (outdated/incorrect comments)
- [ ] Complex business logic missing an explanation of "why it's done this way"
- [ ] Cryptic code (regex, bitwise operations, etc.) without explanation
- [ ] Temporary workarounds without reason and context explanation
- [ ] TODO/FIXME/HACK comments left without specific context
- [ ] Missing doc comments on public API of functions/classes (for libraries/SDKs)
- [ ] Commented-out code blocks (see dead-code reference)
- [ ] Lint suppression comments (eslint-disable / noqa, etc.) without a specific reason

## Good Examples / Bad Examples

Bad example — comment that repeats the code:
```
// Get the user name
const userName = getUserName();
```

Good example — no comment needed, code is clear enough:
```
const userName = getUserName();
```

Bad example — workaround without context:
```
// Temporary fix
timeout = timeout * 2;
```

Good example — explains why it's needed:
```
// Increase retry margin due to intermittent timeout issues with external API (JIRA-1234)
timeout = timeout * 2;
```

## Anti-patterns

- **Code Translator** — Adding comments to every line. It's just noise for anyone who can read the code.
- **Lying Comments** — Modifying code without updating comments. The most dangerous type of comment.
- **Compensating for Bad Code with Comments** — Instead of adding long comments to complex code, refactoring the code itself is better.
- **Journal Comments** — Leaving change history as comments at the top of a file. `git log` serves this purpose.
- **Closing Brace Comments** — `} // end if`, `} // end for`, etc. This signals the function is too long; consider splitting it.
- **Lint Suppression Without Reason** — Having only `// eslint-disable-next-line @typescript-eslint/no-explicit-any` without explaining why `any` is unavoidable. This causes missed opportunities for type improvement.
