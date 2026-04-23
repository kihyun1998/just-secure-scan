---
name: jss-tauri
description: Scan Tauri frontend↔backend bridge for IPC, allowlist/capability, CSP, and updater issues
argument-hint: <optional: path to scan>
allowed-tools: Read Grep Glob Bash
---

# jss-tauri

Scans Tauri-specific frontend ↔ backend bridge issues. **Scoped strictly to the boundary** — general Rust language issues (`unsafe`, panic, command injection in arbitrary code, TOCTOU, supply chain) are out of scope and covered by `/jss-rust` and `/jss-unsafe`.

If `$ARGUMENTS` is provided, scan only that path. Otherwise scan the entire project.

## Scope Boundary

This skill reports only bridge-layer issues. Split with `/jss-rust`:

| Example | Scanner |
|---|---|
| `#[tauri::command]` receives a JS-supplied path and opens it without traversal check | jss-tauri (boundary) |
| A `#[tauri::command]` internally uses `unsafe` or `unwrap()` | jss-rust (language) |
| `Command::new(user_input)` anywhere | jss-rust (command injection) |
| `tauri.conf.json` has `shell.all: true` | jss-tauri (allowlist) |
| `build.rs` downloads over HTTP | jss-rust (supply chain) |

## Discovery Strategy

Do not read entire files upfront. Identify target files first.
Exclude `target/`, `node_modules/`, `dist/`, `build/`, `.next/` from all Grep searches.

### 0. Project Detection

- Verify `src-tauri/tauri.conf.json` (or `src-tauri/Tauri.toml`) exists. If not, report "Not a Tauri project" and exit.
- Read that config fully — it is the core file for this scan.
- Read `src-tauri/Cargo.toml` — note Tauri major version. v1 and v2 have **different config schemas** (v1 uses `tauri.allowlist.*`, v2 uses `src-tauri/capabilities/*.json` + `tauri.security.*`).

### 1–6. Discovery Targets

1. Read `src-tauri/tauri.conf.json` in full — central config
2. Glob `src-tauri/capabilities/*.json` (v2 only) — capability files
3. Grep for `#\[tauri::command\]` in `src-tauri/src/**/*.rs` → command definitions
4. Grep for `\.invoke_handler\(|tauri::generate_handler!` → command registration site
5. Grep for `invoke\(|@tauri-apps/api` in frontend source (`src/`, `frontend/`, `app/`) → JS call sites
6. Grep for `tauri::WindowEvent|\.listen\(|\.emit\(` → IPC event surface

## Checks

### 1. Allowlist / Capability Configuration (Critical ~ Warning)

**Tauri v1** (`tauri.conf.json` → `tauri.allowlist.*`):
- **Wildcard `all: true`** (Critical): `fs.all`, `shell.all`, `http.all`, `process.all`, `dialog.all`, `globalShortcut.all` — grants every API in the namespace to the webview.
- **Shell without scope** (Critical): `shell.execute: true` or `shell.sidecar: true` without `shell.scope` constraining command names.
- **FS without scope** (Critical): `fs.readFile: true` / `fs.writeFile: true` without `fs.scope` path restrictions. If `fs.scope` is present but entries are `$HOME/**` or `$APPDATA/**` — Warning (too broad).
- **HTTP without scope** (Warning): `http.request: true` without `http.scope` restricting hosts.
- **Path traversal tolerance** (Warning): `fs.scope` entries without `requireLiteralLeadingDot: true` allow `..` traversal through matched paths.

**Tauri v2** (`src-tauri/capabilities/*.json`):
- **Over-broad permissions** (Critical): capability grants combining broad defaults (`fs:default` + `shell:default` + `core:default`) without explicit restrictions.
- **Wildcard windows** (Warning): `"windows": ["*"]` applies the capability to every window, including ones loading remote content.
- **Remote URL + privileged capability** (Critical): capability assigned to a window whose `url` is remote (http/https) rather than `tauri://localhost`.

### 2. CSP (Critical ~ Warning)

- **CSP null or missing** (Critical): `security.csp` is `null`, empty, or absent. Webview has no CSP — XSS is unmitigated.
- **`unsafe-inline` / `unsafe-eval` in script-src** (Critical): either directive in `script-src` defeats CSP protection.
- **`unsafe-inline` in style-src** (Warning): consider hash/nonce.
- **Wildcard source** (Warning): `connect-src *`, `img-src *`, or `default-src *` accepts arbitrary origins.

### 3. Dangerous Flags (Critical)

- **`dangerousUseHttpScheme: true`** (Critical): production webview served over http://. MITM trivial.
- **`dangerousRemoteDomainIpcAccess`** (Critical): any entry permits a remote domain to call IPC handlers. `"windows": ["*"]` or permissive `"domain"` regex is effectively RCE.
- **`devPath` in release config** (Critical): `build.devPath` pointing to an external dev server rather than a bundled `dist/` folder in production.
- **`withGlobalTauri: true`** (Warning): exposes `window.__TAURI__` globally — any script in the webview (including third-party) can call IPC.

### 4. `#[tauri::command]` Boundary (Critical ~ Warning)

For every `#[tauri::command]` function, inspect **signature and first-hop usage only** — deeper Rust-internal issues belong to `/jss-rust`.

- **Path arg without traversal check** (Critical): parameter named `path|file|dir|name|target` typed `String` or `PathBuf` reaching `std::fs`, `tokio::fs`, `File::open`, or `read_to_string` without `..` filtering or `canonicalize` + allowlist check.
- **Exec arg without allowlist** (Critical): command string or `Vec<String>` reaching `Command::new` where the binary name is itself the parameter. (Pure `Command::new` misuse unrelated to commands → jss-rust.)
- **Auth-sensitive op without guard** (Warning): command performing privileged action (file write, registry write, process spawn, updater trigger) without an early-return guard. Look for missing `if !is_authorized(...) { return Err(...); }`.
- **Unbounded collections** (Warning): parameter `Vec<T>` or `HashMap<_, _>` accepted without length validation — DoS vector via large payload.
- **Serde `flatten` or `untagged` on command input** (Warning): `#[serde(flatten)]` or `#[serde(untagged)]` on command argument types expands the accepted JSON surface unpredictably.

### 5. IPC Trust Model (Warning)

- **Event listener trusting payload** (Warning): `app.listen_any`, `window.listen`, or equivalent handlers that parse event payloads and act on them (fs/proc/db ops) without source validation. Frontend events can be forged by any code with IPC access.
- **`emit` leaking sensitive data** (Warning): `app.emit` / `window.emit` payloads containing tokens, keys, or other secrets broadcast to all listeners.
- **Frontend-supplied window labels** (Warning): webview-supplied window labels used to route IPC or emit events without validation.

### 6. Updater (Critical ~ Warning)

In `tauri.conf.json` → `updater` (v1) or `plugins.updater` (v2):
- **`active: true` without `pubkey`** (Critical): auto-update enabled with no signature verification. Endpoint compromise → RCE on every client.
- **Non-HTTPS endpoint** (Critical): `endpoints` contains any `http://` URL.
- **`pubkey` placeholder** (Critical): value is `""`, `"TODO"`, `"CHANGEME"`, or a known sample key.
- **Endpoint with unvalidated template vars** (Warning): endpoint template uses variables (`{{arch}}`, `{{current_version}}`) interpolated into host portion.

### 7. Protocol Handlers & Deep Links (Warning ~ Critical)

- **Custom URI scheme without validation** (Warning): `tauri-plugin-deep-link` or OS-level scheme registration where the Rust handler consumes the URL into app state without any structural validation.
- **Deep link param used in `Command::new` / `File::open`** (Critical): parameter extracted from a URL reaches a process or filesystem API. Cross-file inference — mark Confidence accordingly (High if direct, Medium if one hop).

### 8. Window Isolation (Info)

- **Isolation pattern not used** (Info): `build.withGlobalTauri: true` and no separate isolation webview. For apps loading any remote content, the isolation pattern (`tauri.conf.json` → `tauri.pattern`) is recommended.
- **Multiple windows with mixed trust** (Info): multiple windows declared, some loading remote URLs and some loading bundled content, without an isolation intermediary.

## Suppression

Follow suppression rules in CLAUDE.md.

## Output

Follow the common output format in CLAUDE.md, including the inline axis line (Confidence / Blast / Layer) under every finding.
