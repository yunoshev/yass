# Plan: a seat-check that survives Claude Code changes

Status: proposal, not started. Date: 2026-10-06. Constraint kept: prompts and hooks only, no scripts. macOS stays the only supported platform; Linux and Windows come later (see the Windows support issue), but nothing in the core may assume macOS.

## Problem

`agents/seat-check.md` is ~145 lines and mixes three things: the contract with the hooks (file formats), judgment (when to switch), and exact macOS commands that lean on Claude Code internals (the Keychain item, `rate_limit_event` in `-p` output, `~/.claude.json` `oauthAccount`, transcript layout). A Claude Code update breaks commands in the middle of the prompt, and every new platform would grow it further.

## Principle

The core says **what must be true and how to check it**, not how to get there. Exact commands live in short per-platform recipes, each with a postcondition and the Claude Code version it was last verified on. A failed postcondition pauses switching and tells the operator; the background agent never improvises with secrets.

## Layout

| File | Holds | Changes when |
|---|---|---|
| `agents/seat-check.md` (core, target ~60–70 lines) | role, task messages, ground rules, file contracts, steps as goals + postconditions, "decide by `policy.md`" | rarely |
| `platform/macos.md` (target ~40 lines) | recipes R1–R6 below | a Claude Code update, a macOS quirk |
| `platform/linux.md`, `platform/windows.md` | the same recipes for those systems | later |
| `skills/seats/default-policy.md` | also the decision thresholds now hard-coded in the core | the owner says so |

The agent reads the core plus only its own platform file.

## The key simplification: the store is touched in exactly two places

Everything platform-specific about the login reduces to two recipes:
- **R1 read:** copy the login Claude Code uses into `~/.yass/run/current.json` (mode 0600).
- **R2 write:** put a seat's `credentials.json` into the store, secret never in a process argument.

Everything else — which seat is current, expiry, saving a rotated login back, verifying a switch — becomes file-to-file `jq`/hash work on `run/current.json`, identical on every platform. On Linux and Windows R1/R2 are a plain file copy of `~/.claude/.credentials.json` (documented in Claude Code's Authentication docs, "Credential management").

## Recipes (per platform)

| Recipe | Does | Postcondition | Internal dependency? |
|---|---|---|---|
| R1 read store | store → `run/current.json` | valid JSON, `claudeAiOauth.accessToken` non-empty | location documented; format not |
| R2 write store | seat file → store | R1 again; access-token hash equals the seat's | same |
| R3 probe current | one tiny request on the store's login → normalized reading | reading has `h5`, `d7` and both resets | yes: `rate_limit_event` in `-p` output (look for a documented source, see research) |
| R4 probe another seat | same with `CLAUDE_CODE_OAUTH_TOKEN` (documented) from the seat file | same | same |
| R5 show account | after a switch to a login, `/status` shows its account (`~/.claude.json` `oauthAccount`) | field equals the seat's `account.json` | yes; display only, see research |
| R6 live activity | main sessions active in 5 min; live subagent context | numbers, or "unknown" | yes: transcript layout under `~/.claude/projects` |
| R7 clipboard (skill only) | read a pasted token to a file, clear the clipboard | file matches the token pattern | no |

Each recipe states "verified on Claude Code X.Y.Z".

## Inventory of today's `seat-check.md`

| Section (lines) | Goes to | Note |
|---|---|---|
| Intro (8) | core | neutral wording: "the login Claude Code keeps for every session" |
| Task messages (10–17) | core | unchanged |
| Ground rules (21–25) | core | rule 2 becomes "use your platform's recipes; if one fails its postcondition, don't improvise: pause" |
| Files (27–38) | core | it's the contract with the hooks; trim the prose |
| 1. Identify (40–56) | core goal + R1 | the matching rules (exact beats rotated, account + org pair, unknown) stay in the core as words; the comparison runs on files |
| 2. Measure (58–86) | core goal + R3/R4/R6 | probe-age and "within 15 points" thresholds move to default-policy |
| 3. Decide (88–103) | core, shrunk | thresholds (3 points urgent, 150k live context, 20 min deadline) move to default-policy; outcome names and the `pending.json`/`notice.json` formats stay (hook contract) |
| 4a save back | core | becomes "copy `run/current.json` to `<from>` if it holds a refresh token"; the 15-minute expiry guard stays (it's about token rotation, not the platform) |
| 4b write | R2 + core check | |
| 4c account display | R5 | |
| 4d notify | core | `switched.json` stays (cross-platform `systemMessage`); `osascript` becomes an optional macOS extra |
| 4e resume note | core | unchanged |
| 5. Record | core | unchanged |

## Self-test and fail-safe

- `config.json` gets `verified_cc_version`. Each run compares it with `claude --version`.
- On a change, before any switch: R1 (read and parse), R3 (probe the current seat); no write. Pass → store the new version. Fail → pause.
- Any recipe failing its postcondition at run time → pause too.
- Pause = `config.json` `paused: "<reason>"`, no switching, and one `systemMessage` per session (same delivery as `switched.json`): `yass paused: Claude Code 2.x.y changed <what>; run /yass:seats repair`.
- `/yass:seats repair` (new skill section): interactive, with the owner. The model reads the official docs (Authentication, CLI reference, hooks) and `claude --help`, proposes a fixed recipe, tests it read-only, the owner approves, the recipe and its version stamp are updated in the repo and pushed.

## Hooks: as dumb as possible

- Target: a hook only (a) decides whether it's time and starts the agent, (b) prints a prepared message once per session.
- `SubagentStop` today sums live subagent context with `find`/`jq`/`awk` itself. Options: keep it (fast, but logic in a hook) or let the agent do it on each throttled call (portable, costs a model run per call while a plan waits). Recommendation: keep it for macOS now; revisit with the Windows issue.
- `jq` in hooks is a portability question (Git Bash on Windows has none): see research.

## Checks at the seams (invariant probes)

1. R1: JSON shape, non-empty access token.
2. R2: read back and compare hashes; mismatch → put `<from>` back, pause.
3. Save back: a login file must carry a refresh token, else stop.
4. Probe: both windows and resets present, else "probe failed", seat not counted.
5. Hook contracts: the sandbox test (fake HOME, `true` instead of `claude`) for every hook change, as done for the status line.
6. After a switch: the next probe of the new current seat returns a reading.

## Steps

1. Answer the open questions (research below).
2. Move thresholds from the core into `default-policy.md` (and the owner's `policy.md`), wording unchanged. Check: a manual check decides the same as before on today's numbers.
3. Write `platform/macos.md` with R1–R7 from today's commands. Check: each recipe's postcondition on this Mac.
4. Rewrite the core around goals, postconditions and R1/R2. Check: `Measure all.`, then a manual `Switch to <seat>` and back, on keys only (no login rotation risk), journal and status line correct.
5. Add the version stamp, self-test and pause. Check: fake a version change; fake a failing R1 (sandbox HOME) → pause message, no switch.
6. Add `/yass:seats repair` to the skill.
7. README: platform section, "verified on".

## Answers from research (2026-10-06, Claude Code 2.1.290)

Details and evidence: `docs/research/claude-code-portability.md`.

1. **Linux hot swap works, verified in a container:** a running session stats `~/.claude/.credentials.json` before every request and re-reads it when the mtime changes; a swap took effect on the next request (~3 s). A stale session can't overwrite a swapped-in file (refresh write-back checks the stored refresh token). So R1/R2 on Linux are a plain file copy. **Windows (code only, unverified):** same file logic, except when Windows Credential Manager is on (`CLAUDE_CODE_FORCE_WINDOWS_CREDMAN=1` or a server flag): then a file swap does nothing. The Windows recipe must detect that and pause.
2. **Usage numbers:** no documented source for all seats. The status line's `rate_limits` is documented but only for the active seat of an interactive session; the probe's `unifiedWindows` stays an internal dependency (R3/R4), guarded by the self-test.
3. **Hooks on Windows** run in Git Bash (PowerShell without it); yass's one-liners need Git for Windows plus `jq`. Keep that as a stated requirement for now, or make hooks dumber (see "Hooks").
4. **Identity:** `claude auth status` reads `oauthAccount` from `~/.claude.json` and shows nothing for keys, so identity stays token-hash based (R1 + files); R5 stays as display only.
5. **Nothing documented hot-swaps a running session** (`CLAUDE_CODE_OAUTH_TOKEN` is fixed per process, `apiKeyHelper` rejects subscription tokens). A documented headless import (`CLAUDE_CODE_OAUTH_REFRESH_TOKEN` + `claude auth login`) might replace the raw store write for login seats; untested.
