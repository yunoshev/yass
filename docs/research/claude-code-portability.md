# yass portability (Linux, Windows) and fewer Claude Code internals: research

Date: 2026-10-06. Research only, nothing in yass was changed.

**Versions.** Claude Code 2.1.290 (build 2026-10-05), checked in three forms: the local macOS binary, the `linux-arm64` and `win32-x64` release binaries (downloaded from the official release channel, SHA-256 checked against the release manifest, never executed on the host; their embedded JS bundle was extracted with `strings` and grepped), and the `linux-arm64` binary actually run inside a throwaway Docker container (Debian bookworm-slim, arm64, glibc). Docs: `code.claude.com/docs/en/*` and the public CHANGELOG, fetched 2026-10-06.

**Evidence tags.** `[doc]` = official docs. `[bundle]` = code read in the compiled JS bundle of 2.1.290 (minified names differ between the macOS, Linux and Windows builds; the logic quoted below was confirmed identical in all three unless said otherwise). `[test]` = ran in the Linux container. `UNVERIFIED` = not observed at runtime; for Windows that means every claim, because there was no Windows machine or VM.

## Summary

1. **Q1, hot reload on Linux: works, and faster than on macOS. Verified.** On Linux the credentials live in a plain file, `~/.claude/.credentials.json`, with no read cache at all. Before every model request Claude Code stats the file and, if its mtime changed, drops its cached login and re-reads. In the container a swap was used by the next request 3 s later, both for an in-place overwrite and for an atomic rename, and a swap back recovered. **Windows (code only, UNVERIFIED):** the Windows build runs the same plaintext-file code and should behave the same way, with one trap: a server-side flag (`tengu_windows_credman`) or `CLAUDE_CODE_FORCE_WINDOWS_CREDMAN=1` moves the login into Windows Credential Manager, and then a file swap does nothing. Confidence: Linux high, Windows medium for the file case, unknown for the Credential Manager case.
2. **Q2, documented usage source: partial.** The statusline `rate_limits` object is documented (0-100 percent plus reset time for the 5h and 7d windows), but it reaches only a user-configured status line in an interactive session and only for the seat that session is using. The documented fields of the SDK `rate_limit_event` carry no usable utilization in the normal "allowed" case; the two-window numbers yass reads (`unifiedWindows`) are undocumented. Verdict: keep the probe, mark it an internal dependency, add the statusline as a free passive reading for the active seat. Confidence: high.
3. **Q3, hooks on Windows: workable only with Git for Windows plus `jq`.** Shell-form hooks run in Git Bash, or in PowerShell when Git Bash is absent, and yass's one-liners are POSIX shell that PowerShell cannot parse. Exec-form hooks (`command` + `args`) are shell-less and portable but cannot hold yass's gating logic. `CLAUDE_CODE_EXECPATH` is undocumented. Confidence: medium (docs and bundle strings; nothing run on Windows).
4. **Q4, identity without internals: partly.** `claude auth status` (documented JSON) gives `email`, `orgId`, `orgName`, `subscriptionType` for login seats, but it reads them from `oauthAccount` in `~/.claude.json`, which a raw credentials swap leaves stale, and it returns nothing for setup-token key seats. Verified only on a key seat; the login-seat fields are from code. Confidence: high for key seats, medium for login seats.
5. **Q5, other documented surfaces: none can hot-swap a running subscription session.** `CLAUDE_CODE_OAUTH_TOKEN` is fixed per process (a 401 even refuses to adopt the stored login), `apiKeyHelper` is the one documented hot-swap mechanism but rejects subscription OAuth tokens (verified), `CLAUDE_CONFIG_DIR` isolates new sessions only. A documented headless import (`CLAUDE_CODE_OAUTH_REFRESH_TOKEN` + `CLAUDE_CODE_OAUTH_SCOPES` + `claude auth login`) might replace yass's raw store write for login seats; untested. Confidence: high on the negatives, low on the import.

Model requests on the key seat: 2 answered haiku requests, 5 messages sent in total (3 in one stream-json session, 2 `apiKeyHelper` runs); every other HTTP attempt was refused with 401 before any inference. Details in "What I ran".

---

## 1. Hot reload of credentials on Linux and Windows (Q1)

### 1.1 Storage backends, per platform

`[doc]` https://code.claude.com/docs/en/authentication#credential-management: macOS Keychain (plaintext fallback in `~/.claude/.credentials.json` when the Keychain rejects the write), Linux `~/.claude/.credentials.json` mode `0600`, Windows `%USERPROFILE%\.claude\.credentials.json` (inherits the profile directory's ACL). `CLAUDE_CONFIG_DIR` moves the file, and keys the macOS Keychain entry to the directory. The docs say the file is managed through `/login` and `/logout`; they do not document hot reload or hand edits.

`[bundle]` the backend is chosen once per process, by platform:

```
Linux:   function Dpr(){return"file"}  ...  function zn(){if(O)return O;return C}      // C = plaintext store
Windows: function zn(){if(z)return z;if(SMn())return K(x,T);return T}                   // T = plaintext, x = "windows-credman"
Windows: function SMn(){if(process.env.CLAUDE_CODE_FORCE_WINDOWS_CREDMAN==="1")return!0;
                        return Q().resolve(ee,Ae)}                                      // Ae reads cachedGrowthBookFeatures.tengu_windows_credman from the global config
macOS:   Keychain store with the plaintext store as fallback
```

So Linux has exactly one backend, the file. Windows has the file by default and Credential Manager (`Bun.secrets`, with the file as fallback) when the env var is `1` or when the server-rolled flag `tengu_windows_credman` is cached as `true` in the global config file.

### 1.2 Read cache and TTL

`[bundle]` Keychain reads are cached for 30 s (`var WAt=30000` and, in the Keychain reader, `if(Date.now()-r.cachedAt<WAt)return r.data`); Credential Manager reads have the same 30 s (`C5t=30000`). The plaintext reader has no cache: `read(e){ ... readFileSync(path,{encoding:"utf8"}) ... }`.

Above the stores sits a memo of the parsed login (`gn()` keeps it in a holder, `$ge.value`), invalidated by the change check that every request runs first:

```
async function fA(e,n){ ...
  let r; try{ r=(await Kf(Gf(Jb(),".credentials.json"))).mtimeMs }catch{ return Uf(e,n) }   // stat the file
  if(r!==e.lastCredentialsMtimeMs){ e.lastCredentialsMtimeMs=r, FR(), Lf(e); return }       // changed: clear caches, notify
  await Sk(e,n) }                                                                           // unchanged: only an "unusable token" recheck, at most every 30 s
```

`Uf` (stat failed, which is the normal case on macOS and on Windows with Credential Manager) clears the memo and re-reads through the store, so on those backends the 30 s store cache is the only delay. The check is called by the check-and-refresh function that the API client factory runs before every model request (`let ke=await bX({credentials:A,storageV5:L}),_e=gn()`), so an idle session notices a swap on its next request, not before.

Consequences, all `[bundle]`:
- Linux and Windows-with-file: the swap is seen on the next request, no TTL. macOS: next request after the 30 s cache expires (the "within ~30 s" yass relies on).
- Detection is `mtimeMs !== lastSeen`, not "newer than". A swap that lands on exactly the same mtime as the previous file is invisible (UNVERIFIED, not tested). This matters for yass: do not preserve mtimes when installing a seat (`cp -p`, `rsync -t`, `touch -r`); write a temp file in the same directory and rename it, so the mtime is fresh.
- A second, newer store layer ("v5", `U()`, `probeCredentials()` with a version number instead of an mtime) exists in the bundle; I could not find what switches it on (no env var or flag by that name). It opens files with `O_NOFOLLOW`, so a symlinked credentials file is refused there. UNVERIFIED at runtime; keep the credentials file a regular file.

### 1.3 What happens on 401 and refresh (can a stale session clobber a swapped file?)

`[bundle]`
- 401 handler: re-read the store; `if(h.accessToken!==e) return ... "tengu_oauth_401_recovered_from_keychain"` (a different token is already stored: adopt it and retry); otherwise force a refresh. A key seat has no refresh token, so the error surfaces (seen in the test, below).
- Refresh is serialized by `<configDir>/.oauth_refresh.lock`, and writes by `<configDir>/.storage-write`. The write-back after a refresh is a compare-and-set:
  `if(!(B.claudeAiOauth!==void 0&&B.claudeAiOauth!==null&&(W===""||W===n)))return w=!0,B` — it stores the refreshed login only if the stored refresh token is still the one that was posted (or empty); otherwise it reports `adopted_sibling` and keeps whatever is stored.
- A token within 5 minutes of expiry counts as expired and triggers a refresh before the request.

So a session that still holds the old seat cannot overwrite a swapped-in file with its old tokens: its next request notices the new mtime and adopts the new login, and a refresh that straddles the swap fails the compare-and-set. What can still go wrong is on the server side: a refresh that was already posted rotates that seat's refresh token, so a copy of that seat saved earlier by yass is dead. That hazard is identical on macOS, applies to login seats only (key seats never refresh), and could not be exercised here (no login seat allowed). UNVERIFIED.

### 1.4 Test on Linux (container, key seat, real binary)

Setup: one long-lived `claude -p --input-format stream-json --output-format stream-json --model haiku --tools ""` process inside the container; the seat file copied in once with `docker cp` and never printed; a second file with the same JSON shape and a bogus access token ("garbage"). Secret-free transcript (container clock, UTC):

```
21:31:57 start session (one long-lived claude -p, stream-json in/out)
21:32:00 SEND message 1
21:32:02 RESULT 1: "is_error":false "subtype":"success" "total_cost_usd":0.000565 "result":"ok
21:32:02 R1 (valid file) => ok
21:32:02 SWAP valid -> garbage accessToken (in-place cp, same JSON shape)
21:32:05 SEND message 2
21:32:09 RESULT 2: "is_error":true "subtype":"success" "result":"Failed to authenticate. API Error: 401 OAuth access token is invalid.
21:32:09 R2 (garbage file, ~3 s after swap) => err
21:32:09 SWAP back to valid (atomic: cp tmp + mv)
21:32:12 SEND message 3
21:32:14 RESULT 3: "is_error":false "subtype":"success" "total_cost_usd":0.001192 "result":"ok
21:32:14 R3 (valid file restored, ~3 s after swap) => ok
21:32:14 api_retry events: 2        (both error_status 401, "authentication_failed", max_retries 10)
21:32:17 stderr bytes: 0
```

Reading: message 2 was sent 3 s after an in-place overwrite and already carried the new (bogus) token, so there is no TTL and no restart needed. A 30 s cache like macOS's would have answered "ok" with the old token. Message 3, 3 s after an atomic rename-over, used the restored token and was answered, so both swap styles are detected and recovery works. The process was never restarted. Message 2 failed 401 three times (two retries plus the final) and none of those reached the model.

### 1.5 Windows (code and docs only, UNVERIFIED)

- Default backend: the plaintext file at `%USERPROFILE%\.claude\.credentials.json` `[doc]`; same change check, same no-cache read `[bundle, win32-x64]`. Expected behavior equals Linux.
- Credential Manager trap `[bundle]`: if `CLAUDE_CODE_FORCE_WINDOWS_CREDMAN` is `1` or `cachedGrowthBookFeatures.tengu_windows_credman` is `true` in the global config (`~/.claude.json`, or under `CLAUDE_CONFIG_DIR`), reads go through Credential Manager first (30 s cache), the file only as fallback. A swap by file would then be ignored or shadowed. The env var cannot turn the flag off (only `=== "1"` is tested). Whether the flag is on for any given user is a server rollout; a Windows install can be checked by reading that one key.
- Account name for the Credential Manager item is `claude-code-user` (macOS uses `$USER`); the target name follows the Keychain item name, including the config-dir hash suffix `Claude Code-credentials-<first 8 hex of sha256(configDir)>` when `CLAUDE_CONFIG_DIR` is set (undocumented `CLAUDE_SECURESTORAGE_CONFIG_DIR` also exists).
- File replacement semantics (renaming over a file another process just read, antivirus or indexer locks, transient `EPERM`) are not covered by anything I read. A retry loop is advisable. UNVERIFIED.

---

## 2. A documented source for the 5h and 7d numbers (Q2)

What exists, from most to least documented:

| Source | Documented? | Gives | Limits |
|---|---|---|---|
| Statusline `rate_limits.{five_hour,seven_day}.{used_percentage,resets_at}` | Yes, https://code.claude.com/docs/en/statusline (since v2.1.80, CHANGELOG) | Both windows, percent 0-100, reset epoch seconds | Only in a user-configured `statusLine` command, interactive sessions; "only for claude.ai Pro and Max subscribers" and "only after the first API response"; plugins cannot set `statusLine` (`plugins-reference`: only `agent` and `subagentStatusLine` apply). Reflects the seat of the session that made the last request. |
| SDK `rate_limit_event` fields `status`, `resetsAt`, `utilization`, `rateLimitType` | Yes, `agent-sdk/typescript` (`SDKRateLimitEvent`) | One window | `[test]` the event on a normal "allowed" response had `status`, `resetsAt`, `rateLimitType` and no top-level `utilization`; documented fields do not give both windows. |
| `rate_limit_info.unifiedWindows.{five_hour,seven_day}.{utilization,resetsAt}` | No | Both windows; utilization is a fraction 0-1 (yass multiplies by 100) | Undocumented. `[test]` present on a key seat in stream-json output. |
| Response headers `anthropic-ratelimit-unified-*` (the origin of the above `[bundle]`) | No | Same | Would need a local pass-through proxy via `ANTHROPIC_BASE_URL` (documented) to read; not worth it. |
| `GET /api/oauth/usage` and the `/usage` command | No (endpoint); `/usage` is a documented command but its report format is not | All windows (`five_hour`, `seven_day`, `seven_day_opus`, `seven_day_sonnet`, extra usage, ...) with no model request | `[bundle]` needs the `user:profile` OAuth scope, which login seats have and setup-token key seats do not. `[test]` `claude -p "/usage"` on a key seat returned only a local cost summary (zero cost, zero tokens), no plan limits. Not tested on a login seat (not allowed): UNVERIFIED. |
| OpenTelemetry metrics (`claude_code.cost.usage`, `claude_code.token.usage`) | Yes, `monitoring-usage` | Spend and tokens | No plan percentages, no window resets. Not a substitute. |

`[test]` one captured event, key seat (values omitted): `{"type":"rate_limit_event","rate_limit_info":{"status":"allowed","resetsAt":...,"rateLimitType":"five_hour","overageStatus":...,"isUsingOverage":false,"unifiedWindows":{"five_hour":{"utilization":...,"resetsAt":...},"seven_day":{"utilization":...,"resetsAt":...}}}}`. It appeared on the first successful response of the session and not on the second (n=2, reason unknown), which is harmless for yass because each probe is a fresh process.

Conclusion: no documented, stable replacement exists for reading both windows of a seat that is not the active one, because the numbers only arrive with a response made on that seat's token (or through the undocumented usage endpoint for login seats). Practical moves: keep the one-request probe and treat `unifiedWindows` as the internal dependency to pin to a Claude Code version range with a clear "reading unavailable" fallback; use the documented statusline fields for the active seat when the owner has (or accepts) a status line that tees the JSON to `~/.yass/`. UNVERIFIED: statusline availability for Team/Enterprise seats and key seats (the docs name Pro and Max only).

---

## 3. Hooks on Windows (Q3)

`[doc]` https://code.claude.com/docs/en/hooks (common fields, exec form) and https://code.claude.com/docs/en/setup (Windows):
- Shell-form `command` runs in `sh -c` on macOS and Linux, in Git Bash on Windows, or in PowerShell when Git Bash is not installed. The per-hook `shell` field accepts `"bash"` or `"powershell"` (default `bash`, or `powershell` on Windows without Git Bash). PowerShell 7 (`pwsh.exe`) is preferred, Windows PowerShell 5.1 otherwise.
- Exec form (`command` plus `args`) spawns the program directly, no shell, cross-platform, with `${CLAUDE_PLUGIN_ROOT}`-style placeholders substituted. On Windows `command` must resolve to a real executable (`.exe`); `.cmd` and `.bat` shims need shell form.
- `CLAUDE_PROJECT_DIR`, `CLAUDE_PLUGIN_ROOT`, `CLAUDE_PLUGIN_DATA` are exported to both forms.
- On Windows, paths inside the hook's stdin JSON arrive with backslashes (`C:\project\...`) even under Git Bash, where `$PWD` looks like `/c/project`; comparisons written with forward slashes silently never match.
- Git for Windows is "recommended" but optional for Claude Code itself (without it the shell tool is PowerShell). WSL counts as Linux.
- `[bundle]` string when no shell is found: "Claude Code on Windows requires a shell tool. Git Bash was not found and the PowerShell tool is disabled (CLAUDE_CODE_USE_POWERSHELL_TOOL=0)."

`CLAUDE_CODE_EXECPATH`: undocumented (zero hits in the docs). `[bundle]` it is set to `process.execPath` in the environment overrides of the Bash tool's shell (`he[I4e]=process.execPath`, `I4e="CLAUDE_CODE_EXECPATH"`). Whether hook processes get it was not established on any OS; yass's hooks already have the fallback `${CLAUDE_CODE_EXECPATH:-$HOME/.local/bin/claude}`. The Windows fallback path would be `%USERPROFILE%\.local\bin\claude.exe` `[doc: setup uninstall steps]`. UNVERIFIED how `$CLAUDE_CODE_EXECPATH` (a backslash path) behaves when expanded in Git Bash.

What the current hooks need, and Windows status (the Git for Windows tool inventory is from general knowledge of that distribution, not checked here, UNVERIFIED):

| Used by yass hooks | Linux | Git Bash on Windows |
|---|---|---|
| `jq` | usually installed, else package | not bundled; must be installed separately |
| `find -mmin`, `touch`, `date +%s`, `tail`, `cat`, `printf`, `nohup`, `env -u` | GNU, present | MSYS2 coreutils/findutils, believed present |
| `shasum` (seat identity) | `sha256sum` always, `shasum` often | `sha256sum` present, `shasum` believed absent |
| `xxd` (Keychain hex payload) | package `xxd`/`vim-common` | believed present (ships with Git's vim); moot once the Keychain path is gone |
| `osascript` | absent | absent |
| `$HOME` | `~` | maps to the user profile directory, so `$HOME/.claude` is the same directory as `%USERPROFILE%\.claude` (UNVERIFIED) |

Without Git for Windows the shell-form hooks fall back to PowerShell and fail to parse. There is no OS condition field in the hook schema that I found, so one `hooks.json` cannot declare "bash form on Unix, PowerShell form on Windows" (UNVERIFIED that none exists). Exec form is portable but runs unconditionally, and yass's gating (policy file present, throttle by mtime, `YASS_CHILD`, env-token sessions) has to run before anything expensive is started; on every `PostToolUse` that gate must be cheap. Hence: Windows support realistically means "Git for Windows and `jq` required", as Claude Code itself recommends Git for Windows on native Windows.

---

## 4. Identity without internals (Q4)

`[doc]` https://code.claude.com/docs/en/cli-reference: `claude auth status` prints JSON (`--text` for human output), exit code 0 if logged in and 1 if not, with `configDirectory` (v2.1.268+, CHANGELOG). Also `claude auth login` (`--email`, `--sso`, `--console`), `claude auth logout`, `claude setup-token`.

`[test]` on a setup-token key seat, valid file (values null where Claude Code has no data):

```
{"loggedIn":true,"authMethod":"claude.ai","apiProvider":"firstParty","analyticsDisabled":false,
 "projectsDirectory":"/root/.claude/projects","configDirectory":"/root/.claude",
 "email":null,"orgId":null,"orgName":null,"subscriptionType":null}
```

- With the same file replaced by one holding a bogus token: byte-identical output. `auth status` reports that a credential exists, it does not validate it. Only a model request (or the usage endpoint) proves a seat works.
- With `apiKeyHelper` configured in settings: `authMethod: "api_key_helper"`, `apiKeySource: "apiKeyHelper"`, no email.
- A key seat has no identity at all in Claude Code's own output. Its only identity is the token (yass hashes it), which is the same on every platform.

`[bundle]` for login seats the status builder fills `email`, `orgId`, `orgName` from `oauthAccount` in the global config file (`~/.claude.json`, or `$CLAUDE_CONFIG_DIR/.claude.json`) and `subscriptionType` from the stored credential. The account-profile refresh at startup merges the server's profile only if it is for the same account: `if(n.account_uuid!=null&&n.account_uuid!==e.accountUuid)return e` — so after a raw credentials swap to another account `oauthAccount` stays at the previous account until something rewrites it. This is why yass rewrites `oauthAccount` today, and it means `claude auth status` right after a swap reports the old account for login seats (UNVERIFIED at runtime: no login seat allowed). The same function exists in the Linux and Windows builds.

What this allows:
- A documented identity key for login seats: `(email, orgId)` from `claude auth status`, no parsing of `~/.claude.json`, provided the seat's `oauthAccount` has been set right (which only an internal write or a real `claude auth login` does).
- No documented way to read `accountUuid`/`organizationUuid`; yass uses those today from `~/.claude.json`.
- Import: the only documented non-interactive login is `CLAUDE_CODE_OAUTH_REFRESH_TOKEN` + `CLAUDE_CODE_OAUTH_SCOPES` with `claude auth login` `[doc: env-vars]` ("exchanges this token directly instead of opening a browser"). Not tested; it would consume (rotate) the refresh token, so yass would have to re-capture the stored login afterwards. If it also populates `oauthAccount` through Claude Code's own code, it could replace both the raw store write and the `oauthAccount` rewrite for login seats on all three platforms. UNVERIFIED, and the biggest open item.
- Do not use `claude auth logout` to "switch": `[bundle]` it revokes the token server-side ("OAuth token revoke failed (status=...); continuing with local logout").

---

## 5. Other documented surfaces (Q5)

`[doc]` https://code.claude.com/docs/en/authentication#authentication-precedence, in order: cloud provider (`CLAUDE_CODE_USE_BEDROCK`/`VERTEX`/`FOUNDRY`), `ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_API_KEY`, `apiKeyHelper`, `CLAUDE_CODE_OAUTH_TOKEN`, Anthropic profile or federation credentials, subscription login from `/login`.

- **`CLAUDE_CODE_OAUTH_TOKEN`** (from `claude setup-token`, one-year token, inference only, no Remote Control and no claude.ai connectors). Identical on every OS, perfect for new processes (CI, scripts, a fixed seat per terminal). Fixed for the life of the process: `[bundle]` on a 401 the session prints "OAuth 401: keeping the user-supplied CLAUDE_CODE_OAUTH_TOKEN instead of adopting the stored credential. Mint a fresh token with `claude setup-token` and restart with it, or unset the variable and run /login." So such sessions cannot be moved by yass, which is why the hooks skip them. `--bare` mode ignores the variable.
- **`apiKeyHelper`** (`[doc]` settings-reference): a command whose output becomes the credential; rerun after 5 minutes (`CLAUDE_CODE_API_KEY_HELPER_TTL_MS`) and on any 401/403. This is the one documented hot-swap path. `[test]` with the key seat's OAuth token as helper output: `auth status` accepted it as `api_key_helper`, but every request got 401 "API key is invalid" (the output is sent as `X-Api-Key` and `Authorization: Bearer`; subscription OAuth tokens are not accepted as an API key). Dead end for subscription seats; fine for Console API keys.
- **`CLAUDE_CONFIG_DIR`**: documented for "multiple accounts side by side"; the credentials file (or the Keychain entry on macOS) is keyed to the directory. Gives per-terminal seats for new sessions on all platforms; does nothing for running ones. A session started with a different config directory also does not see yass's swaps in the default one.
- **Profiles and federation** (`ANTHROPIC_PROFILE`, `ant` CLI, workload identity): Console or federation credentials, not claude.ai subscription seats; features that need the claude.ai login (connectors, `/schedule`) are off while selected.
- **Managed settings** `forceLoginMethod`, `forceLoginOrgUUID`: can make Claude Code refuse a seat of another organization or method. yass cannot detect this except by a failed probe.
- **Plaintext file location and mode** (`~/.claude/.credentials.json`, `0600`) is documented per OS. That makes "write the file" a documented location with undocumented reload semantics (section 1), and the file's JSON layout (`claudeAiOauth` object, `accessToken`, `refreshToken`, `expiresAt`, `scopes`, `subscriptionType`) undocumented; yass only needs to treat it as a blob and hash two fields.
- **Write guards**: `[bundle]` Claude Code's file tools refuse the Anthropic profile store and a "host credentials file" (the path in `CLAUDE_CODE_HOST_CREDS_FILE`, set only when a host process manages the login); 2.1.287 made allow rules and hooks prompt for shell writes to those. I found no guard keyed to `~/.claude/.credentials.json` itself, but did not test an allowed-Bash agent writing it (UNVERIFIED; under `-p` a prompt would be a refusal). The sandbox default deny list does include `Read(~/.claude/.credentials.json)`, relevant only if the seat-check agent is ever sandboxed.

---

## Implications for yass

**What can move to documented surfaces**
- Where the login lives: documented per OS, so the store adapter (Keychain on macOS, file on Linux and Windows) rests on documented locations. Only the hot-reload behavior is undocumented, and it is cheap to self-test (section 1.4 is a 15-line recipe that costs two tiny requests).
- Identity for login seats: `claude auth status` fields, after the `oauthAccount` is right.
- Usage for the active seat: statusline `rate_limits`, if the owner accepts a one-line status-line tee. Optional, free.
- New-session seats (terminals, CI): `CLAUDE_CODE_OAUTH_TOKEN` or `CLAUDE_CONFIG_DIR`, identical everywhere.
- Possibly the login import itself (`claude auth login` with a refresh token), if a later test shows it writes the store and `oauthAccount` correctly and survives refresh-token rotation.

**What stays an internal dependency (pin, wrap in one adapter, self-test)**
- The hot-reload contract: a store change is picked up on the next request (30 s TTL on macOS, none on Linux and Windows-file). Self-test after a switch: probe once and compare identity.
- `rate_limit_event.rate_limit_info.unifiedWindows` as the only way to read both windows of a non-active key seat; `/api/oauth/usage` stays optional for login seats.
- `oauthAccount` in `~/.claude.json` (identity staleness after a raw swap) and the `accountUuid`/`organizationUuid` fields.
- `CLAUDE_CODE_EXECPATH` (with a PATH fallback).
- Credentials JSON layout; Keychain item naming with the config-dir hash.

**What blocks Linux and Windows**
- Linux: nothing found in Claude Code. Blockers are inside yass: the Keychain write (`security -i` hex payload), `xxd`, `shasum`, `osascript`. Replace with: build the seat file in the same directory, `chmod 600`, `mv` over `~/.claude/.credentials.json` (fresh mtime, no `-p`), regular file only. WSL is Linux with its own `~/.claude`, separate from the Windows one.
- Windows: (1) the Credential Manager switch (env var or server flag) would silently disable file swaps and cannot be overridden from outside; detect it (`cachedGrowthBookFeatures.tengu_windows_credman` in the global config, or the env var) and refuse or warn, and writing a Credential Manager item from a shell is a separate unverified problem; (2) hooks need Git for Windows and `jq`, with no per-OS hook variant available; (3) backslash paths in hook input; (4) file-replace retry on lock errors; (5) none of it has run on a Windows machine, so a short Windows smoke test (the section 1.4 recipe, Git Bash, a key seat) is required before claiming support.
- Both: the seat-check agent's probe and the statusline option behave the same on all three platforms; the one request per probe stays.

**Open items, in value order**: (a) a Windows smoke test; (b) the `claude auth login` refresh-token import on a login seat; (c) `/usage` and `/api/oauth/usage` on a login seat; (d) a same-mtime swap (expected miss); (e) whether hooks see `CLAUDE_CODE_EXECPATH`; (f) the interactive statusline for key and Team seats.

---

## What I ran

- Read yass's `README.md`, `agents/seat-check.md`, `hooks/hooks.json`, `skills/seats/SKILL.md` (read-only). Nothing in `~/.yass`, the Keychain or `~/.claude.json` was touched; no login, no logout.
- Downloaded the 2.1.290 release manifest, the `linux-arm64` and `win32-x64` binaries (SHA-256 equal to the manifest) and the install script into a scratch directory; extracted printable strings and grepped the macOS, Linux and Windows bundles. Fetched the docs pages named above plus the CHANGELOG.
- Docker: Docker Desktop was not running, so I started it myself. One image pulled (`debian:bookworm-slim`), one container, one setup-token key seat (no refresh token) copied in with `docker cp` file to file; its contents were never printed. The container ran: `claude auth status` on a valid file and on a bogus-token file; the 3-message hot-reload session of section 1.4; `claude -p /usage`; two `apiKeyHelper` runs.
- Model requests on the key seat: 2 answered haiku requests (messages 1 and 3 of the session); 5 messages sent in all (3 in the session, 2 helper runs); the remaining attempts, roughly a dozen retries, were rejected with 401 before inference (message 2 and both helper runs). The first helper run hit the default 10 retries and the 90 s timeout; the second used `CLAUDE_CODE_MAX_RETRIES=1`. The first answered request may have opened that seat's 5-hour window.
- Cleanup: container removed, the pulled image removed (image and container lists identical to the lists taken before), Docker quit (daemon down, no Docker.app processes left), no copy of the key seat's credentials anywhere in the scratch directory, the secret-free transcript checked for token patterns.
