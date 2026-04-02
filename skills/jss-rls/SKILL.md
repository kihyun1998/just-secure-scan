---
name: jss-rls
description: Scan Supabase Row Level Security (RLS) policies
argument-hint: <optional: migration path>
allowed-tools: Read Grep Glob Bash
---

# jss-rls

Scans Supabase Row Level Security policies. Focuses on SQL analysis — source code path analysis belongs to `/jss-auth` and `/jss-nextjs`.

## Scan Scope

If `$ARGUMENTS` is provided, scan only SQL files at that path. Otherwise search these paths in order:
1. `supabase/migrations/*.sql`
2. `prisma/migrations/**/*.sql`
3. `drizzle/**/*.sql`, `migrations/**/*.sql`

## Discovery Order

1. Glob for SQL files at the paths above.
2. If no SQL files found, ask the user whether to run `supabase db dump --schema public`. Do not execute automatically.
3. If no SQL files exist and CLI is unavailable, report as "not checked".
4. If Supabase type files (`types/supabase.ts`, `database.types.ts`, etc.) exist, use them to identify table names.

## Checks

### Final-State Analysis

**Aggregate all migration files** to determine the final state per table. Do not judge based on individual files.

DDL statements to track:
- `CREATE TABLE` / `CREATE TABLE IF NOT EXISTS` — table creation
- `DROP TABLE` — table deletion (remove from check targets)
- `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` — RLS enabled
- `ALTER TABLE ... DISABLE ROW LEVEL SECURITY` — RLS disabled
- `CREATE POLICY ... ON table` — policy added
- `DROP POLICY ... ON table` — policy removed
- `ALTER TABLE ... RENAME TO` — track table renames

Excluded tables (auto-skip): `_prisma_migrations`, `schema_migrations`, `__drizzle_migrations`

### 1. RLS Not Enabled (Critical)

- Tables where `ENABLE ROW LEVEL SECURITY` is missing or was followed by `DISABLE` in the final state.

### 2. No Policies (Warning)

- Tables with RLS enabled but no `CREATE POLICY` (or all policies dropped). All access is blocked — may be intentional, so Warning.

### 3. Overly Permissive Policies

- `FOR ALL` + `USING (true)` → **Critical** (effectively no RLS)
- `FOR SELECT` + `USING (true)` → **Warning** (public read may be intentional, e.g., `categories`, `public_posts`)
- `FOR INSERT/UPDATE/DELETE` + `WITH CHECK (true)` → **Warning** (may be intentional for anonymous form submissions)
- `FOR ALL` + `WITH CHECK (true)` → **Critical**

### 4. SECURITY DEFINER Functions (Warning)

- Functions created with `SECURITY DEFINER` bypass the caller's RLS.
- Check for authorization checks inside the function (`auth.uid()`, `auth.role()`, etc.).
- Data modification without authorization checks — Warning.

### 5. GRANT Permissions (Warning)

- `GRANT ALL ON table TO anon` — full permissions to anonymous users
- `GRANT INSERT/UPDATE/DELETE ON table TO anon` — write access to anonymous users
- `GRANT ... TO authenticated` is normal — Info.

### 6. Direct auth.users Access (Warning)

- SQL directly selecting from or joining with `auth.users`. This is a Supabase internal table and direct access is not recommended.

## Output

Follow the common output format in CLAUDE.md.
Additionally output an RLS status summary table:

```
| Table  | RLS | Policies | Status                        |
|--------|-----|----------|-------------------------------|
| users  | ON  | 3        | OK                            |
| posts  | ON  | 0        | Warning: no policies          |
| orders | OFF | -        | Critical: RLS not enabled     |
```

## Suppression

Follow suppression rules in CLAUDE.md.
