# just-agents Roadmap 2

Next-phase work beyond the current `roadmap.md`. Three tracks:

1. **Evaluation model** — move from single-axis severity to multi-axis tagging + synthesis to avoid "wall of warnings" fatigue.
2. **Desktop security skills** — `jss-tauri`, `jss-flutter-desktop`.
3. **State-management review references** — `state-riverpod.md`, `state-zustand.md` (and optionally `state-bloc.md`, `state-redux-toolkit.md`).

Execution order at the bottom.

---

## Track 1 — Evaluation Model

### Motivation

Current output is one-dimensional: Critical / Warning / Info. Problems:

- Same "Warning" label covers "single record leak" and "unauthenticated admin API" — reviewers can't triage.
- No confidence axis — a low-confidence Critical should not block a high-confidence one.
- No architectural view — 5 point-wise warnings in the auth layer never promote to "auth layer is systemically thin".

### A. Per-finding multi-axis tagging

Keep Severity as the top-level label. Add three axes to every finding.

| Axis | Values | Purpose |
|---|---|---|
| **Confidence** | High / Medium / Low | Separates confirmed from probable. Low-conf Critical ≠ High-conf Critical. |
| **Blast radius** | Record / Table / Database / Infrastructure | "Single row leak" vs "full DB dump" distinction. **Optional** — omit when the finding doesn't map to data exposure (e.g. missing CSRF token, missing security header). |
| **Defense layer** | Auth / Authz / Input / Output / Transport / Config / Supply chain | Groups findings architecturally; drives the synthesis pass. |

Output format addition (per finding) — axes inline on one line, separated by `·`. Omit axes that don't apply:

```
## Critical — Immediate action required
- `src/app/api/admin/route.ts:14` — unauthenticated admin endpoint
  Confidence: High · Blast: Database · Layer: Auth
  Impact: attacker reads all user records without credentials
  Fix: add session check via middleware matcher

- `next.config.js:12` — CSP header missing
  Confidence: High · Layer: Config       ← Blast omitted (no data-exposure mapping)
  Impact: XSS payloads execute with no CSP fallback
  Fix: set security.headers.contentSecurityPolicy
```

### B. Report-level synthesis pass

Produced by `security-scanner` agent after all per-skill scans complete. Two new sections:

**Attacker scenarios** — top 3 findings stitched into end-to-end attack paths from an unauthenticated outsider. Example:

```
### Scenario 1: full DB read as unauthenticated outsider
1. Hit /api/admin/users (unauthenticated — Finding #3)
2. Response leaks user emails + internal IDs (Finding #7 — over-exposed columns)
3. Same endpoint accepts ?limit=99999 without cap (Finding #11 — missing pagination limit)
Outcome: attacker dumps full users table in one request.
```

**Systemic patterns** — group findings by Defense layer. If 3+ findings share a layer, promote the layer itself to a top-level concern.

```
### Systemic: Auth layer (4 findings)
Middleware matcher excludes /api/admin/*, 2 Server Actions skip session check,
1 API route trusts client-provided user_id. Recommend: audit all auth checkpoints,
not just these 4 lines.
```

### Spec location

- Axis values + output format: `CLAUDE.md` under Security Scan section.
- Synthesis templates: `agents/security-scanner.md`.
- Every `jss-*` skill updated to emit the three axes per finding.

### What this does NOT add

- **Numeric scoring** (CVSS-style). Rejected — too much ceremony; categorical axes already give enough signal.
- **Red/Blue/Architect persona pass** (option C from discussion). Deferred to Phase 4 as an optional `--deep` mode.

---

## Track 2 — Desktop Security Skills

### `jss-tauri` (new)

Highest-value new skill. **Scoped strictly to the frontend ↔ backend boundary** — not Rust language issues. Tauri's security posture is largely controlled by `tauri.conf.json` and `#[tauri::command]` signatures, making config + boundary static analysis high-leverage.

**Detection**: `src-tauri/tauri.conf.json` exists.

**Checks (bridge-focused only)**

| Category | Items |
|---|---|
| **allowlist** | `fs.all`, `shell.all`, `http.all`, `process.all` wildcards. Missing `fs.scope` / `shell.scope` restrictions. |
| **CSP** | `security.csp` is `null`, or contains `unsafe-inline` / `unsafe-eval`. |
| **Dangerous flags** | `dangerousUseHttpScheme`, `dangerousRemoteDomainIpcAccess`, `devPath` present in production config. |
| **`#[tauri::command]` boundary** | Command function receives webview input that reaches `File`/path APIs without `..` filtering. Command exposed without auth gate when the op is sensitive. Command signature accepts unbounded collections. |
| **IPC trust model** | `invoke()` handlers skip input validation; implicit trust between webview and Rust. Event payloads from webview assumed safe. |
| **Updater** | `updater.active: true` without `pubkey`, non-HTTPS endpoint, missing signature verification. |
| **Protocol handlers** | Custom URI scheme registered without validating arguments before passing to Rust. |
| **Window isolation** | Isolation pattern not applied; direct IPC between untrusted webview and privileged Rust. |

**Scope boundary with `jss-rust`** (different lenses, no overlap by design):

| Example | Scanner |
|---|---|
| `#[tauri::command]` receives a path from JS and opens it without traversal check | `jss-tauri` (boundary) |
| A `#[tauri::command]` internally uses `unsafe` or `unwrap()` | `jss-rust` (language) |
| `Command::new(user_input)` anywhere (Tauri or not) | `jss-rust` (command injection) |
| `tauri.conf.json` has `shell.all: true` | `jss-tauri` (allowlist) |

`file:line`-based dedup stays as-is — in practice the two skills inspect different lines because they look at different things.

### `jss-flutter-desktop` (new)

Flutter on Windows/macOS/Linux has no OS sandbox — filesystem, processes, registry become attack surface.

**Detection**: `pubspec.yaml` with `flutter` SDK **and** at least one of `windows/`, `macos/`, `linux/` directories present.

**Checks**

| Category | Items |
|---|---|
| **File access** | `File()` / `Directory()` built from user input without path normalization (traversal). |
| **Process** | `Process.run` / `Process.start` with `runInShell: true` or string concatenation. PATH-based executable resolution. |
| **FFI** | `DynamicLibrary.open` with hardcoded names relative to cwd (DLL hijacking on Windows). |
| **Platform channel** | `MethodChannel` arguments reaching native code without validation on the Kotlin/Swift/C++ side. |
| **Auto-updater** | `auto_updater` / `flutter_autoupdate` without signature verification or HTTPS. |
| **Local storage** | Sensitive data written via `path_provider.getTemporaryDirectory()` or Documents dir unencrypted. |
| **Windows registry** | `win32_registry` writes to HKLM (elevation abuse). |
| **Shell integration** | Protocol handler / file association registration without argument validation. |

### Split vs extend `jss-flutter`?

Keep existing `jss-flutter` as mobile-oriented (permissions, deep links, cert pinning). `jss-flutter-desktop` runs alongside it for desktop-capable projects. Multi-platform apps run both.

### Agent update

`security-scanner.md` stack-detection table additions:

| Signal | Stack |
|---|---|
| `src-tauri/tauri.conf.json` | Tauri |
| `pubspec.yaml` + (`windows/` or `macos/` or `linux/`) | Flutter desktop (run both `jss-flutter` and `jss-flutter-desktop`) |

Scan execution order for Tauri:
1. `/jss-secrets`
2. `/jss-tauri` — bridge/config only
3. `/jss-rust` (scoped to `src-tauri/`) — Rust language issues

---

## Track 3 — State-Management Review References

`flutter-dart.md` set the precedent that framework-specific references are acceptable inside `jcr-review`. Extend with state management, which is the most common source of quality issues in real codebases.

### `state-riverpod.md` (Flutter)

| Perspective | Check |
|---|---|
| **ref usage** | `ref.read` called inside `build` method (should be `ref.watch`). |
| **autoDispose** | One-shot providers missing `.autoDispose` — memory leak. |
| **family abuse** | `.family(arg)` called with unbounded arg values — cache explosion. |
| **modifier chain** | `.autoDispose.family.future` or longer — reconsider design. |
| **AsyncValue handling** | `.when` missing a branch, or `.value!` force-unwrap. |
| **StateNotifier → Notifier** | New code using legacy `StateNotifier` API. |
| **Circular dependency** | Provider A watches B, B watches A. |
| **Over-globalization** | Widget-local state promoted to global provider without cause. |
| **Test overrides** | `ProviderScope(overrides:)` missing → hits real API in tests. |
| **Ref storage** | `ref` stored as instance field — becomes invalid after dispose. |

### `state-zustand.md` (React)

| Perspective | Check |
|---|---|
| **Missing selector** | `const state = useStore()` subscribing to whole store — over-renders. |
| **Shallow comparison** | Object/array selectors without `shallow` — new reference every render. |
| **Store monolith** | One store mixing domains — slices pattern recommended. |
| **Persist misuse** | `persist` middleware serializing non-serializable values (Function, Promise, Date). |
| **Async actions** | Store-internal API calls without loading/error state. |
| **Derived state in store** | Values that should be computed via selector stored as state. |
| **Deep nested updates** | `set({a: {...state.a, b: val}})` more than 2 levels without `immer`. |
| **Test isolation** | No store reset between tests. |
| **Subscribe leak** | `subscribe()` return value ignored. |
| **SSR hydration** | Next.js + `persist` hydration mismatch. |

### Agent update — stack-driven selection

`code-reviewer.md` picks the reference by **detected stack**, not by imported library. This lets the reviewer suggest "consider adopting Riverpod" even when the project hasn't introduced it yet.

| Detected stack | Applied reference |
|---|---|
| Flutter (`pubspec.yaml` + `flutter` SDK) | `state-riverpod.md` |
| React / Next.js (`package.json` with `react` or `next`) | `state-zustand.md` |
| Tauri with web frontend (`src-tauri/tauri.conf.json` + React in `package.json`) | `state-zustand.md` |

Reference files are treated as **a lens, not a mechanical checklist** — if the project uses Redux or Bloc, the reviewer still applies state-management principles but flags library-specific calls as "out of scope, manual review".

`jcr-refactor/SKILL.md` adds links to the two new references.

### Future references (out of scope for Phase 3)

- `state-bloc.md` (Flutter)
- `state-redux-toolkit.md` (React)
- `state-jotai.md` (React)

Add only when explicit demand appears.

---

## Execution Order

### Phase 1 — Evaluation model (foundation)

Must happen first. New skills built before this would need reformatting afterward.

1. Draft axis spec in `CLAUDE.md` (Confidence / Blast / Defense layer values + output format).
2. Update the 9 existing `jss-*` skills to emit the three axes per finding.
3. Add Attacker scenarios + Systemic patterns sections to `agents/security-scanner.md`.
4. Verify against a known project end-to-end.

### Phase 2 — Desktop skills

5. `jss-tauri` SKILL.md (new-format from day one).
6. `jss-flutter-desktop` SKILL.md.
7. Extend stack detection in `security-scanner.md` for Tauri and Flutter desktop.
8. Update README contents table.

### Phase 3 — State-management references

Parallel-friendly with Phase 2; independent of the security evaluation model.

9. `skills/jcr-review/references/state-riverpod.md`.
10. `skills/jcr-review/references/state-zustand.md`.
11. Update `code-reviewer.md` with stack-based reference selection (table above).
12. Update `jcr-refactor/SKILL.md` references list.

### Phase 4 — Optional

13. Additional state-mgmt references (`state-bloc.md`, `state-redux-toolkit.md`, `state-jotai.md`) — on demand only.
14. ✅ Persona pass (`--deep` flag on `security-scanner`): Red team / Blue team / Architect perspectives, diffed against each other.
15. ~~Fixture projects~~ — dropped. Without an automated verification pipeline the yml quickly drifts from reality; reconsider only if we build `tools/verify.py` (or equivalent) first.

---

## Decisions recorded

| Topic | Decision |
|---|---|
| Axis output format | Inline single-line: `Confidence: X · Blast: Y · Layer: Z` |
| Blast radius for non-data findings | Axis optional — omit when not applicable (no new values added) |
| Tauri ↔ Rust overlap | Scope split by concern (bridge vs language), no category-aware dedup needed |
| Reference file size | No hard cap; judge by content |
| State-mgmt coverage | Riverpod + Zustand only; stack-driven selection (not dependency-driven) |
