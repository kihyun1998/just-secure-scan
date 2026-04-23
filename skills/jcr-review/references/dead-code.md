# Dead Code — Unnecessary Code Detection

## Why It Matters

Dead code confuses readers. It raises unnecessary questions like "why is this here? Was it left for future use?" and can accidentally introduce bugs when touched during refactoring. It lowers the signal-to-noise ratio of the codebase.

## Principles

- Delete unused code. It can be recovered from git if needed.
- "We might use it later" is not a reason to keep it.
- Commented-out code is dead code.

## Checklist

- [ ] Variables declared but never used
- [ ] Functions/methods that are never called
- [ ] Imported but unused modules
- [ ] Unreachable code (code after return/throw)
- [ ] Commented-out code blocks
- [ ] Unused function parameters
- [ ] Empty files or empty classes/modules
- [ ] Constants/enum values that are no longer referenced
- [ ] Code in the opposite branch of a condition that is always true/false
- [ ] Defined utility functions not called anywhere in the project (common functions created but logic duplicated inline)

## Good Examples / Bad Examples

Bad example:
```
const temp = calculateValue();
const result = otherFunction();
return result;
// temp is never used
```

Good example:
```
const result = otherFunction();
return result;
```

Bad example:
```
// function oldHandler(req, res) {
//   res.send("old");
// }

function newHandler(req, res) {
  res.send("new");
}
```

Good example:
```
function newHandler(req, res) {
  res.send("new");
}
```

## Anti-patterns

- **"Just in Case" Keeping** — Git history exists, so delete boldly.
- **Dead Code Left with TODO Comments** — Keep the TODO, but delete the dead code. A text description of what to implement in the TODO is sufficient.
- **Debugging console.log/print** — Always remove after debugging is complete.
- **Keeping Unused Parameters with _ Prefix** — While allowed by some language conventions, if the parameter itself is unnecessary, it should be removed from the signature.
- **Utility Bypass** — Creating common utility functions but never using them, instead repeating the same logic inline at each call site. This causes the utility to become dead code, or the inline versions to diverge in behavior.
