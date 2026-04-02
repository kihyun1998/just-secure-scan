---
description: Supabase RLS(Row Level Security) 정책 점검
argument-hint: <선택: 마이그레이션 경로>
---

# secure-scan-rls

Supabase의 Row Level Security 정책을 점검한다. SQL 분석에 집중하며, 소스 코드 경로 분석은 `/secure-scan-auth`나 `/secure-scan-nextjs`의 영역이다.

## 스캔 범위

`$ARGUMENTS`가 있으면 해당 경로의 SQL 파일만, 없으면 아래 경로를 순차 탐색한다:
1. `supabase/migrations/*.sql`
2. `prisma/migrations/**/*.sql`
3. `drizzle/**/*.sql`, `migrations/**/*.sql`

## 탐색 순서

1. 위 경로에서 SQL 파일 목록을 Glob으로 가져온다.
2. SQL 파일이 없으면 사용자에게 `supabase db dump --schema public` 실행 여부를 확인한다. 자동 실행하지 않는다.
3. SQL 파일이 전혀 없고 CLI도 없으면 "점검 불가"로 리포트한다.
4. Supabase 타입 파일(`types/supabase.ts`, `database.types.ts` 등)이 있으면 테이블 목록 파악에 활용한다.

## 점검 항목

### 최종 상태 기반 분석

**전체 마이그레이션 파일을 합산**하여 테이블별 최종 상태를 판단한다. 개별 파일 단위로 판단하지 않는다.

추적해야 하는 DDL:
- `CREATE TABLE` / `CREATE TABLE IF NOT EXISTS` → 테이블 생성
- `DROP TABLE` → 테이블 삭제 (삭제된 테이블은 점검 대상에서 제거)
- `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` → RLS 활성화
- `ALTER TABLE ... DISABLE ROW LEVEL SECURITY` → RLS 비활성화
- `CREATE POLICY ... ON 테이블` → 정책 추가
- `DROP POLICY ... ON 테이블` → 정책 삭제
- `ALTER TABLE ... RENAME TO` → 테이블명 변경 추적

예외 테이블 (자동 건너뜀): `_prisma_migrations`, `schema_migrations`, `__drizzle_migrations`

### 1. RLS 미활성화 (Critical)

- 최종 상태에서 `ENABLE ROW LEVEL SECURITY`가 없거나 `DISABLE`된 테이블

### 2. 정책 없음 (Warning)

- RLS 활성화 후 `CREATE POLICY`가 하나도 없는 (또는 전부 `DROP POLICY`된) 테이블 — 모든 접근 차단됨. 의도적일 수 있으므로 Warning.

### 3. 과도한 허용 정책

- `FOR ALL` + `USING (true)` → **Critical** (사실상 RLS 없는 것과 동일)
- `FOR SELECT` + `USING (true)` → **Warning** (공개 읽기가 의도적일 수 있음, 예: `categories`, `public_posts`)
- `FOR INSERT/UPDATE/DELETE` + `WITH CHECK (true)` → **Warning** (익명 폼 제출 등 의도적일 수 있음)
- `FOR ALL` + `WITH CHECK (true)` → **Critical**

### 4. SECURITY DEFINER 함수 (Warning)

- `CREATE FUNCTION ... SECURITY DEFINER`로 정의된 함수는 호출자의 RLS를 우회한다.
- 함수 내부에서 `auth.uid()`, `auth.role()` 등 인가 검증이 있는지 확인.
- 인가 검증 없이 데이터 수정을 수행하면 Warning.

### 5. GRANT 권한 (Warning)

- `GRANT ALL ON 테이블 TO anon` — 익명 사용자에게 전체 권한 부여
- `GRANT INSERT/UPDATE/DELETE ON 테이블 TO anon` — 익명 사용자에게 쓰기 권한
- `GRANT ... TO authenticated`는 일반적이므로 Info.

### 6. auth.users 직접 접근 (Warning)

- SQL에서 `auth.users` 테이블을 직접 SELECT/JOIN하는 패턴. `auth.users`는 Supabase 내부 테이블이며 직접 접근은 권장되지 않음.

## 출력

CLAUDE.md의 공통 출력 포맷을 따른다.
테이블별 RLS 상태 요약표를 추가로 출력한다:

```
| 테이블 | RLS | 정책 수 | 상태 |
|--------|-----|---------|------|
| users  | ON  | 3       | OK   |
| posts  | ON  | 0       | Warning: 정책 없음 |
| orders | OFF | -       | Critical: RLS 미활성화 |
```

## 억제

CLAUDE.md의 억제 규칙을 따른다.
