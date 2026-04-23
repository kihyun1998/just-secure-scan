# Error Handling — Error Handling Patterns

## Why It Matters

Poor error handling makes debugging impossible. Empty catch blocks swallow errors, hiding the root cause of problems, while excessive try-catch obscures normal code flow. Error handling directly determines program stability and maintainability.

## Principles

- Don't swallow errors. If you catch them, log them, propagate them upward, or handle them meaningfully.
- Wrap the narrowest possible scope with try-catch.
- Error messages should include "what failed and why."
- Use exceptions only for exceptional situations. Don't use exceptions for normal flow control.

## Checklist

- [ ] Empty catch blocks (swallowing errors)
- [ ] Catch blocks that ignore the error and only return null/undefined/default values
- [ ] Catching overly broad exceptions (top-level `catch(Exception e)` or `catch(e)`)
- [ ] Missing or unclear error messages (`throw new Error("error")`)
- [ ] Excessive try-catch wrapping the entire function
- [ ] Losing original error information in catch blocks
- [ ] Using exceptions for normal flow control
- [ ] Missing error handling in async/await (unhandled promise rejection)
- [ ] Async IIFE `(async () => { ... })()` used without `.catch()` (unhandled rejection risk)
- [ ] Side effect (logging, notifications, etc.) failure causing the main transaction to also fail

## Good Examples / Bad Examples

Bad example — swallowing errors:
```
try {
  await saveUser(user);
} catch (e) {
  // ignore
}
```

Good example — proper handling:
```
try {
  await saveUser(user);
} catch (error) {
  logger.error("Failed to save user", { userId: user.id, error });
  throw new UserSaveError(`Failed to save user: ${user.id}`, { cause: error });
}
```

Bad example — overly broad try-catch:
```
try {
  const config = readConfig();
  const data = fetchData(config.url);
  const result = processData(data);
  await saveResult(result);
  sendNotification(result);
} catch (e) {
  console.log("something went wrong");
}
```

Good example — wrapping only what's needed:
```
const config = readConfig();

let data;
try {
  data = await fetchData(config.url);
} catch (error) {
  throw new DataFetchError(`Failed to fetch data: ${config.url}`, { cause: error });
}

const result = processData(data);
await saveResult(result);
```

## Anti-patterns

- **Pokemon Exception Handling** — "Gotta catch 'em all." Catching all exceptions with a single top-level catch. Handle appropriately by error type.
- **Swallowing** — Doing nothing in a catch block. If intentionally ignoring, state the reason in a comment (e.g., `catch { /* Use default message on JSON parse failure */ }`). Intentional empty catches with comments are acceptable, but empty catches without comments are treated as errors.
- **Losing Cause on Error Wrapping** — Throwing a new error without including the original error in `cause`.
- **Exceptions for Flow Control** — Patterns like `try { findUser() } catch { createUser() }`. Replace with conditional checks.
- **Catch with Only console.log** — If you only log and don't propagate the error, the caller assumes success.
- **Unprotected Side Effect** — Awaiting side effects (audit logs, notification delivery, etc.) inside the main try-catch means a side effect failure turns the entire request into a 500 error. The business logic already succeeded, but the client perceives failure. Wrap side effects in a separate try-catch or handle as fire-and-forget.
