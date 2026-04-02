---
description: Rust 패키지(crate) 보안 점검
argument-hint: <선택: 스캔 대상 경로>
---

# secure-scan-rust

Rust 패키지(crate)의 보안 이슈를 점검한다.
`$ARGUMENTS`가 있으면 해당 경로만, 없으면 프로젝트 전체를 점검한다.

## 탐색 전략

전체 파일을 읽지 않는다. 모든 Grep에서 `target/`, `vendor/`를 제외한다.

### 0. 프로젝트 구조 파악 (최우선)

1. `Cargo.toml`을 읽어 `[workspace]` 여부를 판단한다. workspace이면 각 member crate의 `Cargo.toml`, `build.rs`, `src/lib.rs`, `src/main.rs`를 모두 식별한다.
2. 각 crate가 library(`[lib]`)인지 binary(`[[bin]]`)인지 파악한다. 이 정보는 패닉 점검 범위에 영향을 준다.
3. `#![forbid(unsafe_code)]` 선언 여부를 확인한다. 선언되어 있으면 unsafe 관련 점검을 건너뛰고 Info로 "unsafe_code가 금지되어 있음" 보고.

### 1~6. 탐색 대상

1. Grep으로 `unsafe` 검색 (파일명만) → unsafe 블록 위치
2. Grep으로 `Command::new|process::Command` 검색 → 커맨드 인젝션 후보
3. Grep으로 `extern\s*"C"` 검색 → FFI 경계
4. 각 crate의 `build.rs` → supply chain 리스크
5. Grep으로 `unwrap\(\)|\.expect\(` 검색 → 패닉 후보 (library crate만)
6. Grep으로 `Deserialize|from_reader|from_slice|from_str` 검색 → deserialization 후보
7. Grep으로 `Path::new|PathBuf::from` 검색 → path traversal 후보
8. Grep으로 `query\(|execute\(` 검색 → SQL injection 후보

## 점검 항목

### 1. unsafe 블록 (Critical ~ Warning)

- **불필요한 unsafe** (Warning): safe 대안이 있는데 unsafe를 쓴 경우
- **경계 검증 누락** (Critical): unsafe 블록 내에서 포인터 연산, 슬라이스 접근 시 길이/범위 검증 없음
- **transmute 오용** (Critical): `std::mem::transmute`으로 타입 변환 시 크기/정렬 보장 없음
- **unsafe impl** (Warning): `Send`, `Sync` 수동 구현 시 근거 주석 없음
- `// SAFETY:` 주석이 있는 unsafe 블록은 주석 내용의 타당성을 검토한다. 타당하면 보고하지 않고, 불충분하면 Warning.
- `// SAFETY:` 주석이 전혀 없는 unsafe 블록 → Warning
- 불필요하게 `unsafe fn`으로 선언된 함수 (safe fn + 내부 unsafe 블록으로 변경 가능) → Warning

### 2. 패닉 in library (Warning)

library crate(`[lib]`)에서만 점검한다. binary crate(`[[bin]]`)는 이 항목을 건너뛴다.

- `unwrap()`, `expect()`, `panic!()`, `unreachable!()`, `todo!()`, `unimplemented!()`가 라이브러리 모듈에서 사용됨

**예외 (보고하지 않음)**:
- 테스트 코드 (`#[cfg(test)]`, `#[test]`)
- `// INVARIANT:` 또는 불변 조건을 설명하는 주석이 있는 경우
- `Mutex::lock().unwrap()` — Mutex poisoning은 대부분 panic이 올바른 대응
- 상수 입력에 대한 unwrap: `Regex::new("리터럴").unwrap()`, `"123".parse::<i32>().unwrap()` 등
- `once_cell`/`LazyLock`/`lazy_static` 초기화 블록 내의 unwrap
- `expect()`에 의미 있는 메시지가 있고 의도적 사용이 명확한 경우 → Info로 하향

### 3. Command Injection (Critical ~ Info)

- `Command::new(user_input)` — 사용자 입력이 커맨드명에 직접 전달 (Critical)
- `Command::new("sh").arg("-c").arg(format!(...))` — 셸을 통한 커맨드 조합에 사용자 입력 포함 (Critical)
- `.arg(user_input)` — shell expansion은 발생하지 않으나 (`Command`는 `execvp` 직접 호출), 의도하지 않은 플래그가 될 수 있다 (argument injection). (Info)

### 4. Deserialization (Warning ~ Critical)

- `serde_yaml`로 외부 입력 deserialize — YAML bomb 가능 (Warning)
- `bincode`, `postcard` 등 바이너리 포맷에서 length prefix를 신뢰하여 거대한 메모리 할당 유발 가능 (Warning)
- `serde_json::from_str(user_input)`에서 `#[serde(deny_unknown_fields)]` 미사용 (Info)
- 신뢰할 수 없는 소스에서 `Deserialize` 시 크기/깊이 제한 없음 (Warning)

### 5. SQL Injection (Critical)

- `sqlx::query`, `diesel::sql_query`, `rusqlite::execute` 등에서 `format!`으로 사용자 입력을 쿼리에 삽입하는 패턴
- 파라미터 바인딩(`$1`, `?`)을 사용하지 않고 문자열 결합으로 쿼리 조립

### 6. Path Traversal (Warning)

- `Path::new(user_input)` 또는 `PathBuf::from(user_input)`에서 `../` 필터링 없음
- `canonicalize()` 또는 base path 검증 없이 사용자 경로를 파일 I/O에 직접 사용

### 7. Integer Overflow (Warning)

- `as` 캐스팅으로 큰 타입에서 작은 타입으로 변환 (narrowing cast): `u64 as u32`, `i64 as i32`, `usize as u16` 등. Grep 패턴: `as u8|as u16|as u32|as i8|as i16|as i32`
- 외부 입력을 직접 받는 `pub fn` 내의 산술 연산에 `checked_*`, `saturating_*` 미사용

### 8. Supply Chain — build.rs (Warning ~ Critical)

- `build.rs`에서 네트워크 호출 (`reqwest`, `ureq`, `curl`, `TcpStream`) (Critical)
- `build.rs`에서 `$HOME`, 프로젝트 외부 경로에 파일 쓰기 (Critical)
- `build.rs`에서 `Command::new` 실행 — 무엇을 실행하는지 확인 (Warning)
- proc macro crate가 위와 유사한 패턴 포함 (Warning)

### 9. FFI 경계 (Critical ~ Warning)

상세 FFI 점검은 `/secure-scan-ffi`에서 수행. 이 커맨드에서는 핵심만 확인:

- `extern "C"` 함수에서 null 포인터 체크 없이 `*const`/`*mut` 역참조 (Critical)
- C 문자열(`*const c_char`) 수신 시 `CStr::from_ptr` 전에 null 체크 없음 (Critical)
- FFI 반환값의 에러 체크/errno 확인 누락 (Warning)
- `#[no_mangle] pub extern "C" fn`에서 패닉 가능성 — 프로세스 abort (Warning)

### 10. ReDoS (Warning)

- `fancy-regex` 또는 `pcre2` crate에서 사용자 입력으로 정규식 컴파일 (Warning)
- 표준 `regex` crate는 RE2 기반으로 backtracking이 없어 ReDoS 불가. 단, `Regex::new(user_input)`은 컴파일 시간 DoS 가능 — `RegexBuilder`의 `size_limit` 미설정 시 Info.

### 11. TOCTOU (Info)

- `env::temp_dir()`, `/tmp`, `tempdir` 관련 경로에서 `Path::exists()` 후 `File::open()`/`File::create()` 패턴만 보고한다. 일반 경로는 보고하지 않는다.

### 12. Crypto (Warning)

- `md5`, `sha1` crate 사용 (보안 목적이면 Warning, 체크섬용이면 무시)
- 자체 암호화 구현 패턴
- 하드코딩된 키/IV (`let key = b"..."`, `let iv = [0u8; ...]`)

### 13. Async 보안 (Warning ~ Info)

- `tokio::spawn` 내에서 `unwrap()` — 해당 task만 panic하고 에러가 삼켜짐 (Warning)
- 외부 요청에 `timeout` 미설정 — 영원히 blocking되는 DoS 가능 (Info)
- async 코드에서 `std::sync::Mutex` (동기 mutex) 사용 — deadlock 가능 (Warning)

### 14. 의존성 (Info)

- `cargo audit --json 2>/dev/null | head -100` 실행 가능하면 요약 결과 포함. 불가하면 `Cargo.lock` 존재 여부와 주요 의존성 버전 확인. CVE 커트오프 한계 명시.
- `Cargo.toml`에서 불필요하게 넓은 feature 활성화

## 억제

CLAUDE.md의 억제 규칙을 따른다.

## 출력

CLAUDE.md의 공통 출력 포맷을 따른다.
