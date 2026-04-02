---
name: jss-nextjs
description: Comprehensive Next.js + Supabase security scan
argument-hint: <optional: path to scan>
allowed-tools: Read Grep Glob Bash
---

# jss-nextjs

Scans security issues in Next.js + Supabase projects.
If `$ARGUMENTS` is provided, scan only that path. Otherwise scan the entire project.

## Discovery Strategy

Do not read entire files upfront. Identify target files first using the patterns below.
Exclude `node_modules/`, `.next/`, `dist/`, `build/` from all Grep searches.

### 0. Router Detection

Run `ls` to check for `app/`, `src/app/`, `pages/`. Adjust path patterns accordingly:
- App Router: `{app,src/app}/**/route.{ts,js}`, `{app,src/app}/**/action*.{ts,js}`
- Pages Router: `{pages,src/pages}/api/**/*.{ts,js}`, `getServerSideProps` patterns

### 1–7. Discovery Targets

1. Grep for `createClient|createServerClient|createBrowserClient` (file names only) — Supabase client usage
2. Glob for `{app,src/app}/**/route.{ts,js}`, `{pages,src/pages}/api/**/*.{ts,js}` — API Routes
3. Grep for `"use server"` (file names only) — Server Actions
4. Glob for `{,src/}middleware.{ts,js}` — auth route matching
5. `.env*`, `next.config.{js,ts,mjs}` — environment variables, security headers
6. `supabase/migrations/*.sql` — RLS policies
7. Grep for `"use client"` then filter for files that also import `supabase`
8. Glob for `{lib,src/lib,utils,src/utils}/**/supabase*.{ts,js}` — central Supabase client factories

## Checks

### 1. Supabase RLS (Critical ~ Warning)

**Aggregate all migration files** to determine final RLS state per table. Track `DROP TABLE`, `DISABLE ROW LEVEL SECURITY`, `DROP POLICY` to determine final state. Detailed rules match `/jss-rls`:

- RLS not enabled on a table → Critical
- RLS enabled but no policies → Warning
- `FOR ALL` + `USING (true)` → Critical
- `FOR SELECT` + `USING (true)` → Warning (public read may be intentional)
- `SECURITY DEFINER` function without authorization checks → Warning
- If no SQL files exist, ask the user whether to run `supabase db dump --schema public`. Do not execute automatically. If CLI is unavailable, report as "not checked" at the bottom.
- Migration paths: `supabase/migrations/*.sql`, `prisma/migrations/**/*.sql`, `drizzle/**/*.sql`

### 2. Key Exposure (Critical)

- `NEXT_PUBLIC_SUPABASE_SERVICE_ROLE_KEY` or `NEXT_PUBLIC_` prefix containing `service_role`, `secret`, `private`
- Hardcoded Supabase JWTs in source code (`eyJhbGciOi...` pattern)
- `createClient(url, service_role_key)` in locations that may be included in the client bundle (`"use client"` files, `page.tsx`, etc.)
- **Exception**: `NEXT_PUBLIC_SUPABASE_ANON_KEY` and `NEXT_PUBLIC_SUPABASE_URL` are public by design — do not report.

### 3. Authentication/Authorization (Critical ~ Warning)

- **Missing Server Action auth** (Critical): `"use server"` function performs DB writes without an auth call. Auth patterns: `getUser()`, `auth()`, `requireAuth`, `withAuth`, `createServerClient(...).auth.getUser()`. Report as Warning if uncertain.
- **Standalone getSession()** (Warning): `getSession()` does not re-verify JWT on the server — unsuitable for auth. Use `getUser()`.
- **Missing API Route auth** (Critical): POST/PUT/DELETE/PATCH handlers in `route.ts` without auth. Also check `pages/api/` handlers for Pages Router.
- **Middleware bypass** (Warning): `middleware.ts` matcher missing routes that should be protected. Check Parallel Routes and Intercepting Routes (`@modal`, `(.)`, etc.). Downgrade individual Server Action / API Route auth findings to Warning if middleware already covers that route.
- **Supabase admin client misuse** (Critical): `createClient` with `service_role` key reachable from a route without auth verification.

### 4. Server Action Input Validation (Warning)

- Check for validation library usage in `"use server"` functions: `zod`, `valibot`, `yup`, `next-safe-action` (built-in zod schema)
- `FormData.get()` values passed to DB without validation
- Server Actions are public HTTP endpoints — validate accordingly
- TypeScript types are not runtime validation

### 5. XSS (Warning)

- `dangerouslySetInnerHTML` usage without sanitization (DOMPurify, etc.)
- `href={variable}` without `javascript:` protocol filtering. Skip if href is a string literal.
- Markdown rendering libraries without HTML sanitization config

### 6. Security Headers (Info)

- Check `next.config.{js,ts,mjs}` for `headers()` configuration:
  - `Content-Security-Policy`
  - `X-Frame-Options`
  - `X-Content-Type-Options`
  - `Strict-Transport-Security`
  - `Referrer-Policy`
- If none are set — Info. Include note: "These may be configured at the deployment platform level (Vercel, Netlify, etc.)."

### 7. Open Redirect (Warning)

- `redirect(variable)` or `router.push(variable)` patterns. Skip if argument is a string literal (e.g., `redirect('/dashboard')`).
- Redirect URL taken from query parameters without validation.

### 8. SSR/CSR Boundary (Warning)

- Server Component passing entire rows from `.select('*')` to Client Component via props.
- Server Component passing session tokens from `cookies()` to Client Component via props.

### 9. Caching Security (Warning)

- `unstable_cache` or React `cache()` caching per-user data — may leak to other users.
- `revalidatePath` / `revalidateTag` called from unauthenticated Server Actions — DoS vector.

### 10. Supabase Storage (Warning)

- Public bucket configuration (in migration SQL or code)
- File upload without `contentType` or file size validation

### 11. Supabase Cookie Config (Warning)

- `@supabase/ssr`'s `createServerClient` cookie options: verify `httpOnly`, `secure`, `sameSite` settings.

### 12. API Security (Warning ~ Info)

- CORS: `Access-Control-Allow-Origin: *` setting
- Rate limiting: no rate limiting on API Routes (Info)
- `next.config.{js,ts,mjs}` `images.remotePatterns` with `hostname: '**'` wildcard — SSRF vector (Warning)

### 13. Dependencies (Info)

- Run `npm audit --json 2>/dev/null | head -100` if available and include summary. Otherwise check `package-lock.json` for known vulnerable packages. Note CVE cutoff limitation.

## Suppression

Follow suppression rules in CLAUDE.md.

## Output

Follow the common output format in CLAUDE.md.
