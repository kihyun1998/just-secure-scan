---
name: jss-auth
description: Scan Next.js + Supabase authentication and authorization security issues
argument-hint: <optional: path to scan>
allowed-tools: Read Grep Glob Bash
---

# jss-auth

Scans authentication and authorization security issues in Next.js + Supabase projects.

## Scan Scope

If `$ARGUMENTS` is provided, scan only that path. Otherwise scan the entire project.
Always exclude build artifacts: `node_modules/`, `.next/`, `dist/`, `build/`.

## Discovery Order

1. Glob for `{,src/}middleware.{ts,js}` — locate middleware file
2. `ls` to check for `app/`, `src/app/`, `pages/` (detect router type)
3. Glob for API Routes: `{app,src/app}/**/route.{ts,js}`, `{pages,src/pages}/api/**/*.{ts,js}`
4. Grep for `"use server"` (file names only) — Server Action files
5. Grep for `createClient|createServerClient` — Supabase client usage
6. Glob for `{lib,src/lib,utils,src/utils}/**/supabase*.{ts,js}` — central client factories

## Checks

### 1. Middleware Route Coverage (Warning)

- Read `middleware.ts` matcher patterns and identify protected routes.
- Report API Routes and Server Actions whose paths are not covered by the matcher.
- Also check Parallel Routes (`@modal/`) and Intercepting Routes (`(.)`, `(..)`).
- If no middleware exists at all — Warning: "No authentication middleware found".
- Skip routes that are intentionally public (webhooks, health checks, etc.) where public intent is clear from filename or comments.

### 2. Missing Auth in Server Actions (Critical)

- Grep for `"use server"` to identify Server Actions.
- Check whether an auth call exists before any DB write within the function.
- Auth function patterns: `getUser()`, `auth()`, `requireAuth`, `withAuth`, `createServerClient(...).auth.getUser()`. Custom wrappers are traced up to 1 level deep; beyond that, report as "manual verification required".
- Downgrade to Warning if middleware already protects that route.
- If uncertain, report as Warning.

### 3. Missing Auth in API Routes (Critical)

- POST/PUT/DELETE/PATCH handlers in `route.ts` / `pages/api/*.ts` without auth verification — Critical.
- GET handlers — Warning only if they access sensitive data.

### 4. Standalone getSession() Usage (Warning)

- `getSession()` does not re-verify the JWT on the server, making it unsuitable for authentication decisions. Use `getUser()` instead.
- Downgrade to Info if `getSession()` is used solely for UI display (e.g., showing user name) and not for auth decisions.

### 5. Supabase Admin Client Misuse (Critical)

- Check if a Supabase client created with a `service_role` key is reachable from a route without auth verification.
- Trace central client factories that use `service_role` and check auth state of importing files.

### 6. Server Action Input Validation (Warning)

- Check whether `"use server"` functions use validation libraries: `zod`, `valibot`, `yup`, `next-safe-action`.
- Flag patterns where `FormData.get()` values are passed directly to the DB without validation.
- TypeScript types are not runtime validation and do not count.
- Downgrade to Info for simple boolean toggles or single ID parameters.

### 7. SSR/CSR Boundary Auth Data Leakage (Warning)

- Server Component passing session tokens from `cookies()` to Client Component props — Warning.
- Server Component passing entire user rows from `.select('*')` to Client Component — Warning.

### 8. Caching and Auth (Warning)

- `unstable_cache` or React `cache()` caching per-user data — may leak to other users.
- `revalidatePath` / `revalidateTag` called from unauthenticated Server Actions — DoS vector.

### 9. Supabase Cookie/Session Config (Warning ~ Info)

- Verify correct cookie renewal middleware pattern for `@supabase/ssr`'s `createServerClient`.
- `@supabase/ssr` v0.5+ handles cookie settings internally. Skip manual cookie checks for that version. For earlier versions, verify `httpOnly`, `secure`, `sameSite` settings.
- Version check: read `@supabase/ssr` version from `package.json`.

### 10. Open Redirect (Warning)

- `redirect(variable)` or `router.push(variable)` patterns after login/logout. Skip if the argument is a string literal.
- Query parameters (`?next=`, `?redirect=`, etc.) used for redirection without validation.

## Suppression

Follow suppression rules in CLAUDE.md.

## Output

Follow the common output format in CLAUDE.md.
