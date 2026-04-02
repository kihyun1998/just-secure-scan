---
description: Next.js 인증/인가 점검
argument-hint: <선택: 스캔 대상 경로>
---

# secure-scan-auth

Next.js + Supabase 프로젝트의 인증/인가 관련 보안 이슈를 점검한다.

## 스캔 범위

`$ARGUMENTS`가 있으면 해당 경로만, 없으면 프로젝트 전체를 점검한다.
빌드 아티팩트(`node_modules/`, `.next/`, `dist/`, `build/`)는 항상 제외한다.

## 탐색 순서

1. Glob으로 `{,src/}middleware.{ts,js}` → middleware 파일 확인
2. `ls`로 `app/`, `src/app/`, `pages/` 존재 여부 확인 (라우터 감지)
3. Glob으로 API Route: `{app,src/app}/**/route.{ts,js}`, `{pages,src/pages}/api/**/*.{ts,js}`
4. Grep으로 `"use server"` 검색 (파일명만) → Server Action 파일
5. Grep으로 `createClient|createServerClient` 검색 → Supabase 클라이언트 사용처
6. Glob으로 `{lib,src/lib,utils,src/utils}/**/supabase*.{ts,js}` → 중앙 클라이언트 팩토리

## 점검 항목

### 1. Middleware 경로 커버리지 (Warning)

- `middleware.ts`의 `matcher` 패턴을 읽어 보호 대상 경로 목록을 파악한다.
- API Route, Server Action이 있는 경로 중 matcher에 포함되지 않은 것을 보고한다.
- Parallel Routes(`@modal/`), Intercepting Routes(`(.)`, `(..)`)도 확인한다.
- middleware가 아예 없으면 Warning: "인증 미들웨어 없음"
- 의도적으로 공개인 경로(webhook, health check 등)는 파일명이나 주석에서 공개 의도가 명확하면 건너뛴다.

### 2. Server Action 인증 누락 (Critical)

- Grep으로 `"use server"` 검색하여 Server Action을 식별한다.
- 함수 내에서 DB 쓰기 수행 전 인증 호출이 있는지 확인.
- 인증 함수 패턴: `getUser()`, `auth()`, `requireAuth`, `withAuth`, `createServerClient(...).auth.getUser()`. 커스텀 래퍼는 1단계까지 추적. 그 이상은 "수동 확인 필요"로 보고.
- middleware에서 해당 경로가 이미 보호되고 있다면 Warning으로 하향.
- 불확실한 경우 Warning으로 보고.

### 3. API Route 인증 누락 (Critical)

- `route.ts`/`pages/api/*.ts`의 POST/PUT/DELETE/PATCH 핸들러에서 인증 검증 없음 → Critical
- GET 핸들러는 민감 데이터 접근이 있는 경우만 Warning

### 4. getSession() 단독 사용 (Warning)

- `getSession()`은 JWT를 서버에서 재검증하지 않으므로 인증 판단에 부적합. `getUser()`를 사용해야 한다.
- `getSession()`이 UI 표시 목적(예: 사용자 이름)으로만 사용되고 인증 판단에 사용되지 않으면 Info로 하향.

### 5. Supabase admin client 오용 (Critical)

- `service_role` key로 생성된 Supabase 클라이언트가 인증 검증 없이 호출 가능한 경로에 있는지 확인.
- 중앙 클라이언트 팩토리에서 `service_role`을 사용하는 클라이언트를 추적하고, import하는 파일의 인증 상태 확인.

### 6. Server Action 입력 검증 (Warning)

- `"use server"` 함수에서 `zod`, `valibot`, `yup`, `next-safe-action` 등 검증 라이브러리 사용 여부.
- `FormData.get()`으로 꺼내서 검증 없이 DB에 전달하는 패턴.
- TypeScript 타입은 런타임 검증이 아니므로 검증으로 인정하지 않는다.
- 단순 boolean toggle이나 ID 하나만 받는 경우는 Info로 하향.

### 7. SSR/CSR 경계의 인증 데이터 노출 (Warning)

- Server Component에서 `cookies()`로 꺼낸 세션 토큰을 Client Component props로 전달하는 경우 → Warning
- Server Component에서 Supabase `.select('*')`로 가져온 전체 사용자 row를 Client Component에 전달하는 경우 → Warning

### 8. 캐싱과 인증 (Warning)

- `unstable_cache` 또는 React `cache()` 내에서 사용자별 데이터를 캐싱 → 다른 사용자에게 노출 가능
- 인증 없는 Server Action에서 `revalidatePath`/`revalidateTag` 호출 → DoS 벡터

### 9. Supabase 쿠키/세션 설정 (Warning ~ Info)

- `@supabase/ssr`의 `createServerClient`에서 쿠키 갱신 미들웨어 패턴이 올바른지 확인.
- `@supabase/ssr` v0.5+ 에서는 쿠키 설정이 라이브러리 내부에서 처리된다. 해당 버전이면 수동 쿠키 설정 점검을 건너뛴다. 이전 버전이면 `httpOnly`, `secure`, `sameSite` 설정을 검증.
- 버전 확인: `package.json`에서 `@supabase/ssr` 버전을 읽는다.

### 10. Open Redirect (Warning)

- 로그인/로그아웃 후 `redirect(변수)` 또는 `router.push(변수)` 패턴. 인자가 문자열 리터럴이면 건너뛴다.
- 쿼리 파라미터(`?next=`, `?redirect=` 등)에서 URL을 가져와 검증 없이 리다이렉트.

## 억제

CLAUDE.md의 억제 규칙을 따른다.

## 출력

CLAUDE.md의 공통 출력 포맷을 따른다.
