---
description: Next.js + Supabase 보안 점검
argument-hint: <선택: 스캔 대상 경로>
---

# secure-scan-nextjs

Next.js + Supabase 프로젝트의 보안 이슈를 점검한다.
`$ARGUMENTS`가 있으면 해당 경로만, 없으면 프로젝트 전체를 점검한다.

## 탐색 전략

전체 파일을 읽지 않는다. 아래 순서로 대상 파일을 먼저 식별한다.
모든 Grep에서 `node_modules/`, `.next/`, `dist/`, `build/`를 제외한다.

### 0. 라우터 감지

`ls`로 `app/`, `src/app/`, `pages/` 존재 여부를 확인. 이하 경로 패턴을 그에 맞게 조정한다.
- App Router: `{app,src/app}/**/route.{ts,js}`, `{app,src/app}/**/action*.{ts,js}`
- Pages Router: `{pages,src/pages}/api/**/*.{ts,js}`, `getServerSideProps` 패턴

### 1~7. 탐색 대상

1. Grep으로 `createClient|createServerClient|createBrowserClient` 검색 (파일명만, `--files-with-matches`) → Supabase 클라이언트 사용처
2. Glob으로 `{app,src/app}/**/route.{ts,js}`, `{pages,src/pages}/api/**/*.{ts,js}` → API Route
3. Grep으로 `"use server"` 검색 (파일명만) → Server Action
4. Glob으로 `{,src/}middleware.{ts,js}` → 인증 경로 매칭
5. `.env*`, `next.config.{js,ts,mjs}` → 환경변수, 보안 헤더
6. `supabase/migrations/*.sql` → RLS 정책
7. Grep으로 `"use client"` 검색 후 그 중 `supabase` import가 있는 파일
8. Glob으로 `{lib,src/lib,utils,src/utils}/**/supabase*.{ts,js}` → 중앙 Supabase 클라이언트 팩토리

## 점검 항목

### 1. Supabase RLS (Critical ~ Warning)

**전체 마이그레이션 파일을 합산**하여 테이블별 최종 RLS 상태를 판단한다. `DROP TABLE`, `DISABLE ROW LEVEL SECURITY`, `DROP POLICY`도 추적하여 최종 상태 기반으로 판단. 상세 규칙은 `/secure-scan-rls`와 동일:

- RLS 미활성화 테이블 → Critical
- RLS 활성화 + 정책 없음 → Warning
- `FOR ALL` + `USING (true)` → Critical
- `FOR SELECT` + `USING (true)` → Warning (공개 읽기가 의도적일 수 있음)
- `SECURITY DEFINER` 함수에서 인가 검증 없음 → Warning
- SQL 파일이 없으면 사용자에게 `supabase db dump --schema public` 실행 여부를 확인한 후 진행. 자동 실행하지 않는다. CLI 미설치 시 "점검 불가" 항목으로 리포트 하단에 표기.
- 마이그레이션 경로: `supabase/migrations/*.sql`, `prisma/migrations/**/*.sql`, `drizzle/**/*.sql`

### 2. 키 노출 (Critical)

- `NEXT_PUBLIC_SUPABASE_SERVICE_ROLE_KEY` 또는 `NEXT_PUBLIC_` 접두사에 `service_role`, `secret`, `private` 포함
- 소스 코드에 Supabase JWT가 하드코딩 (`eyJhbGciOi...` 패턴)
- `createClient(url, service_role_key)`가 클라이언트 번들에 포함될 수 있는 위치에 있는지 확인 (`"use client"` 파일, `page.tsx` 등)
- **예외**: `NEXT_PUBLIC_SUPABASE_ANON_KEY`와 `NEXT_PUBLIC_SUPABASE_URL`은 설계상 공개 키이므로 보고하지 않는다.

### 3. 인증/인가 (Critical ~ Warning)

- **Server Action 인증 누락** (Critical): `"use server"` 함수 내에서 인증 호출 없이 DB 쓰기 수행. 인증 함수 패턴: `getUser()`, `auth()`, `requireAuth`, `withAuth`, `createServerClient(...).auth.getUser()`. 불확실한 경우 Warning으로 보고.
- **`getSession()` 단독 사용** (Warning): `getSession()`은 JWT를 서버에서 재검증하지 않으므로 인증용으로 부적합. `getUser()`를 사용해야 한다.
- **API Route 인증 누락** (Critical): `route.ts`의 POST/PUT/DELETE/PATCH 핸들러에서 인증 검증 없음. Pages Router에서는 `pages/api/` 핸들러도 동일하게 점검.
- **Middleware 우회** (Warning): `middleware.ts`의 `matcher` 패턴에 보호해야 할 경로가 빠져 있는지. Parallel Routes, Intercepting Routes(`@modal`, `(.)` 등)도 확인. 단, middleware에서 해당 경로가 이미 보호되고 있다면 개별 Server Action/API Route의 인증 누락은 Warning으로 하향.
- **Supabase admin client 오용** (Critical): `createClient`에 `service_role` key를 쓰는 코드가 인증 검증 없이 호출 가능한 경로에 있는지

### 4. Server Action 입력 검증 (Warning)

- `"use server"` 함수의 매개변수에 검증 라이브러리 사용 여부: `zod`, `valibot`, `yup`, `next-safe-action` (내장 zod 스키마)
- `FormData`를 직접 `.get()`으로 꺼내서 검증 없이 DB에 전달하는 패턴
- Server Action은 공개 HTTP endpoint임을 감안하여 점검
- TypeScript 타입은 런타임 검증이 아니므로 검증으로 인정하지 않는다

### 5. XSS (Warning)

- `dangerouslySetInnerHTML` 사용처에서 sanitize 여부 (DOMPurify 등)
- `href={변수}` 패턴에서 `javascript:` 프로토콜 필터링 여부. `href` 값이 문자열 리터럴이면 건너뛴다.
- 마크다운 렌더링 라이브러리 사용 시 HTML sanitize 설정

### 6. 보안 헤더 (Info)

- `next.config.{js,ts,mjs}`에서 `headers()` 설정 확인:
  - `Content-Security-Policy`
  - `X-Frame-Options`
  - `X-Content-Type-Options`
  - `Strict-Transport-Security`
  - `Referrer-Policy`
- 하나도 없으면 Info로 보고. 배포 플랫폼(Vercel, Netlify 등)에서 설정했을 수 있으므로 "플랫폼 설정 확인 권장" 문구 포함.

### 7. Open Redirect (Warning)

- `redirect(변수)` 또는 `router.push(변수)` 패턴. 인자가 문자열 리터럴(`redirect('/dashboard')`)이면 건너뛴다.
- 쿼리 파라미터에서 redirect URL을 가져와 검증 없이 사용

### 8. SSR/CSR 경계 (Warning)

- Server Component에서 Supabase `.select('*')`로 가져온 전체 row를 Client Component에 props로 전달하는 경우
- Server Component에서 `cookies()`로 꺼낸 세션 토큰을 Client Component props로 전달하는 경우

### 9. 캐싱 보안 (Warning)

- `unstable_cache` 또는 React `cache()` 내에서 사용자별 데이터를 캐싱하는 패턴 — 다른 사용자에게 노출 가능
- 인증 없는 Server Action에서 `revalidatePath`/`revalidateTag` 호출 — DoS 벡터

### 10. Supabase Storage (Warning)

- 공개(public) 버킷 설정 확인 (마이그레이션 SQL 또는 코드에서)
- 파일 업로드 시 `contentType`, 파일 크기 검증 여부

### 11. Supabase 쿠키 설정 (Warning)

- `@supabase/ssr`의 `createServerClient`에서 `cookies()` 옵션 검증: `httpOnly`, `secure`, `sameSite` 설정

### 12. API 보안 (Warning ~ Info)

- CORS: `Access-Control-Allow-Origin: *` 설정
- Rate limit: API Route에 rate limiting 미적용 (Info)
- `next.config.{js,ts,mjs}`의 `images.remotePatterns`에 `hostname: '**'` 와일드카드 — SSRF 벡터 (Warning)

### 13. 의존성 (Info)

- `npm audit --json 2>/dev/null | head -100` 실행 가능하면 요약 결과 포함. 불가하면 `package-lock.json`에서 알려진 취약 패키지 확인. CVE 커트오프 한계 명시.

## 억제

CLAUDE.md의 억제 규칙을 따른다.

## 출력

CLAUDE.md의 공통 출력 포맷을 따른다.
