# Magic Values — Magic Numbers / Hardcoding

## Why It Matters

When `if (status === 3)` appears in code, nobody knows what 3 means. Magic values hide the intent of the code and create the problem of having to find every occurrence when a value needs to change.

## Principles

- Give names to values that carry meaning (constants, enums).
- Self-evident values like 0, 1, -1, empty string are not considered magic values.
- If the same value is used in multiple places, it must be extracted as a constant.
- Configuration values (timeout, retry count, URL, etc.) should be separated into config files or environment variables.

## Checklist

- [ ] Numeric literals with unclear meaning (e.g., `if (retries > 3)`, `setTimeout(fn, 86400000)`)
- [ ] Same string literal repeated in multiple places
- [ ] Hardcoded URLs, file paths, API endpoints
- [ ] Hardcoded configuration values (timeout, page size, max retry count)
- [ ] Bit flags or status codes used as raw numbers
- [ ] DB error codes compared as string literals directly (e.g., PostgreSQL `"23505"`)
- [ ] Time conversion constants like `24 * 60 * 60 * 1000` used inline

## Good Examples / Bad Examples

Bad example:
```
if (user.role === 2) {
  setTimeout(fetchData, 30000);
}
```

Good example:
```
const ROLE_ADMIN = 2;
const FETCH_INTERVAL_MS = 30_000;

if (user.role === ROLE_ADMIN) {
  setTimeout(fetchData, FETCH_INTERVAL_MS);
}
```

Bad example:
```
const response = await fetch("https://api.example.com/v2/users");
```

Good example:
```
const API_BASE_URL = process.env.API_BASE_URL;
const response = await fetch(`${API_BASE_URL}/users`);
```

## Anti-patterns

- **Making Every Number a Constant** — `const ONE = 1` is meaningless. Self-evident values are fine as-is.
- **Value Repeated in Constant Name** — Instead of `const THIRTY = 30`, use `const MAX_RETRIES = 30` to convey meaning.
- **Numeric Constants Instead of Enums** — When there are multiple related constants, group them as an enum or object.
- **Strings Used as Enums** — If string comparisons like `status === "active"` appear in multiple places, define them as constants/enums.
- **DB Error Code Magic Strings** — Directly comparing DB error codes like `code === "23505"` without constants. Define meaningful constants like `const PG_UNIQUE_VIOLATION = "23505"`.
- **Time Conversion Magic Numbers** — Using inline millisecond conversions like `days * 24 * 60 * 60 * 1000`. Use constants like `const MS_PER_DAY = 86_400_000` or utilities.
