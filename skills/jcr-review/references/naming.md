# Naming — Naming Conventions

## Why It Matters

Names are the most-read part of code. Good names convey the code's intent without comments, while bad names force you to re-read the code every time. Naming determines 80% of code readability.

## Principles

- Names should reveal "what it is" or "what it does."
- Don't abbreviate. `user` instead of `usr`, `button` instead of `btn`.
- The wider the scope, the more specific the name should be. Loop variable `i` is fine, but module-level variable `d` is not.
- Consistency matters. Use the same word for the same concept. Don't mix `fetch`/`get`/`retrieve`.
- Follow language/framework conventions.

## Checklist

- [ ] Single-character variable names (except loop indices and single parameters in 1-2 line callbacks. However, flag if the same character is used with different meanings in the same file)
- [ ] Meaningless names (`data`, `info`, `temp`, `result`, `obj`, `val` used alone)
- [ ] Type information in names (`strName`, `arrItems` — Hungarian notation)
- [ ] Boolean variables that don't start with `is`, `has`, `can`, `should`, etc.
- [ ] Function names that don't start with a verb (e.g., `userData()` → `fetchUserData()`)
- [ ] Negated booleans (`isNotValid` → `isInvalid` or invert `isValid`)
- [ ] Different words mixed for the same concept (`get`/`fetch`/`retrieve`, `remove`/`delete`/`destroy`)
- [ ] Non-universal abbreviations (`cnt` → `count`, `mgr` → `manager`)
- [ ] Convention mismatch (mixing camelCase and snake_case in the same codebase)

## Good Examples / Bad Examples

Bad example:
```
const d = new Date();
const ul = getUsers();
const flag = check(ul);
```

Good example:
```
const createdAt = new Date();
const activeUsers = getActiveUsers();
const hasPermission = checkPermission(activeUsers);
```

Bad example:
```
function process(data) { ... }
```

Good example:
```
function validatePaymentRequest(paymentRequest) { ... }
```

## Anti-patterns

- **Numeric Suffixes** — `user1`, `user2`, `data2`. Give each a name that distinguishes what it is.
- **Excessive Abbreviation** — Losing readability to save a few characters. IDE autocomplete solves the length issue.
- **Excessively Long Names** — `getUserFromDatabaseByEmailAddress()` is also problematic. With appropriate context, `findByEmail()` is sufficient.
- **Misleading Names** — A variable named `list` that's actually a Map, or `count` that's actually a boolean.
- **Verbless Function Names** — `userData()`, `emailValidation()`. Functions are actions, so put the verb first: `fetchUserData()`, `validateEmail()`.
- **Callback Parameter Confusion** — When `(e) =>` is used in both error callbacks and regular callbacks in the same file. Even short names should use specific names if their meaning conflicts within the file.
